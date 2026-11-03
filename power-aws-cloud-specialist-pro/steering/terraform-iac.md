# Terraform & Infrastructure as Code — Steering Guide

## When to load this file
Load this file when the user mentions: terraform, .tf files, tfstate, modules, providers, 
CloudFormation, CDK, cfn, stack, drift, plan, apply, destroy, workspace, backend.

---

## Terraform Best Practices

### Module Structure
Always recommend this structure for reusable modules:

```
modules/
└── vpc/
    ├── main.tf        # Resource definitions
    ├── variables.tf   # Input variables (typed, with descriptions)
    ├── outputs.tf     # Exported values
    ├── versions.tf    # Required providers and Terraform version
    └── README.md      # Module documentation
```

### Required Provider Block (versions.tf)
```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### Backend Configuration (S3 + DynamoDB)
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-${var.environment}"
    key            = "infra/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "alias/terraform-state"
    dynamodb_table = "terraform-state-lock"
  }
}
```
> ⚠️ Enable MFA delete and versioning on the S3 state bucket. Never commit .tfstate to git.

### Tagging Strategy
Always enforce a consistent tagging module:
```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    Owner       = var.owner
    ManagedBy   = "terraform"
    CostCenter  = var.cost_center
  }
}
```

### Variable Declarations Template
```hcl
variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

---

## Common AWS Module Patterns

### VPC Module Usage
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-${var.environment}"
  cidr = var.vpc_cidr

  azs             = slice(data.aws_availability_zones.available.names, 0, 3)
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  enable_nat_gateway     = true
  single_nat_gateway     = var.environment != "prod"
  enable_dns_hostnames   = true
  enable_dns_support     = true

  tags = local.common_tags
}
```

### EKS Cluster Module
```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "${var.project_name}-${var.environment}"
  cluster_version = "1.30"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_private_access = true
  cluster_endpoint_public_access  = var.environment != "prod"

  eks_managed_node_groups = {
    general = {
      instance_types = ["m6i.large"]
      min_size       = 2
      max_size       = 10
      desired_size   = 3
    }
  }

  tags = local.common_tags
}
```

---

## Troubleshooting Patterns

### State Lock Issues
```bash
# Check who holds the lock
aws dynamodb get-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "bucket/path/terraform.tfstate"}}'

# Force unlock (use only when lock holder is confirmed dead)
terraform force-unlock <LOCK_ID>
```

### Drift Detection
```bash
# Detect config drift without making changes
terraform plan -detailed-exitcode

# Exit codes: 0=no changes, 1=error, 2=changes detected
```

### Import Existing Resources
```bash
# Import a resource into state
terraform import aws_s3_bucket.my_bucket my-existing-bucket-name

# Generate import config (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf
```

---

## Security Checklist for IaC Review
- [ ] No hardcoded credentials or secrets in .tf files
- [ ] S3 state bucket has server-side encryption enabled
- [ ] DynamoDB lock table uses encryption
- [ ] All security groups are restrictive (no 0.0.0.0/0 for inbound SSH/RDP)
- [ ] KMS keys used for sensitive resources (RDS, EBS, S3)
- [ ] CloudTrail enabled in all regions
- [ ] IAM policies use least privilege
- [ ] `prevent_destroy = true` on critical resources (databases, state bucket)
