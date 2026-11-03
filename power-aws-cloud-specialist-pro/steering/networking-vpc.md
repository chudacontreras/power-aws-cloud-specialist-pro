# Networking & VPC — Steering Guide

## When to load this file
Load when user mentions: vpc, subnet, cidr, security group, nacl, nat gateway,
igw, internet gateway, peering, transit gateway, direct connect, vpn, route53,
cloudfront, alb, nlb, api gateway, dns, endpoint, flow logs, egress.

---

## VPC Design Principles

### CIDR Planning Template

| Environment | VPC CIDR | Public Subnets | Private App Subnets | Private DB Subnets |
|-------------|----------|----------------|--------------------|--------------------|
| Production | 10.0.0.0/16 | /24 × 3 AZs | /22 × 3 AZs | /24 × 3 AZs |
| Staging | 10.1.0.0/16 | /24 × 2 AZs | /22 × 2 AZs | /24 × 2 AZs |
| Development | 10.2.0.0/16 | /24 × 2 AZs | /22 × 2 AZs | /24 × 2 AZs |

> Reserve CIDR ranges that don't overlap with on-premises or future peered VPCs.

### 3-Tier Architecture (Terraform)
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "prod-vpc"
  cidr = "10.0.0.0/16"

  azs              = ["us-east-1a", "us-east-1b", "us-east-1c"]
  public_subnets   = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets  = ["10.0.10.0/22", "10.0.14.0/22", "10.0.18.0/22"]
  database_subnets = ["10.0.100.0/24", "10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway     = true
  single_nat_gateway     = false  # One per AZ for HA in prod
  enable_dns_hostnames   = true
  enable_dns_support     = true

  # VPC Flow Logs
  enable_flow_log                      = true
  create_flow_log_cloudwatch_log_group = true
  create_flow_log_cloudwatch_iam_role  = true
  flow_log_max_aggregation_interval    = 60

  # EKS tags for subnet discovery
  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }

  tags = local.common_tags
}
```

---

## VPC Endpoints (Reduce NAT Gateway Costs)

```hcl
# Gateway Endpoints (free)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids
}

resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.${var.region}.dynamodb"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = module.vpc.private_route_table_ids
}

# Interface Endpoints (hourly cost, but < NAT for heavy traffic)
resource "aws_vpc_endpoint" "ecr_dkr" {
  vpc_id              = module.vpc.vpc_id
  service_name        = "com.amazonaws.${var.region}.ecr.dkr"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = module.vpc.private_subnets
  security_group_ids  = [aws_security_group.vpc_endpoints.id]
  private_dns_enabled = true
}
```

---

## Security Groups (Least Privilege)

```hcl
# ALB Security Group
resource "aws_security_group" "alb" {
  name        = "${var.app_name}-alb-sg"
  description = "ALB ingress from internet"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description = "HTTPS from internet"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description     = "To app containers"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }
}

# App Security Group — only accepts from ALB
resource "aws_security_group" "app" {
  name        = "${var.app_name}-app-sg"
  description = "App layer - accepts only from ALB"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "From ALB only"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Outbound internet via NAT"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

> ❌ Never open `0.0.0.0/0` for SSH or RDP. Use AWS Systems Manager Session Manager instead.

---

## Application Load Balancer

```hcl
resource "aws_lb" "main" {
  name               = "${var.app_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnets

  enable_deletion_protection = var.environment == "prod"
  drop_invalid_header_fields = true

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.bucket
    prefix  = "${var.app_name}-alb"
    enabled = true
  }

  tags = local.common_tags
}

# HTTPS Listener with redirect from HTTP
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

---

## Transit Gateway (Multi-VPC Connectivity)

```hcl
resource "aws_ec2_transit_gateway" "main" {
  description                     = "Main Transit Gateway"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  dns_support                     = "enable"
  vpn_ecmp_support                = "enable"

  tags = merge(local.common_tags, { Name = "main-tgw" })
}

resource "aws_ec2_transit_gateway_vpc_attachment" "prod" {
  subnet_ids         = module.prod_vpc.private_subnets
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = module.prod_vpc.vpc_id

  tags = { Name = "prod-vpc-attachment" }
}
```

---

## Troubleshooting Connectivity

```bash
# VPC Reachability Analyzer
aws ec2-instance-connect create-network-insights-analysis \
  --network-insights-path-id pip-xxxxxxxxxx

# Test connectivity between two resources
aws ec2 create-network-insights-path \
  --source <source-id> \
  --destination <destination-id> \
  --protocol tcp \
  --destination-port 443

# Check VPC Flow Logs for rejected traffic
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs \
  --filter-pattern "[version, account, eni, source, destination, srcport, destport="443", protocol, packets, bytes, start, end, action="REJECT", log]"
```

### DNS Resolution Issues in VPC
```bash
# Test DNS resolution from within the VPC (via SSM)
aws ssm start-session --target <instance-id>

# Inside the session:
nslookup my-rds.cluster.us-east-1.rds.amazonaws.com
dig +short my-service.internal.example.com

# Check Route 53 Resolver rules
aws route53resolver list-resolver-rules
```
