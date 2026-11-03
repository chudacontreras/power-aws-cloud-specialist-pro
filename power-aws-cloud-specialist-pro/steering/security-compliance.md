# Security & Compliance — Steering Guide

## When to load this file
Load when user mentions: iam, policy, role, permission, kms, encryption, secrets manager,
guardduty, security hub, waf, shield, cognito, scp, permission boundary, irsa,
compliance, audit, cloudtrail, access key, credential, vulnerability, cve.

---

## IAM Best Practices

### Least Privilege Policy Template
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadSpecificBucket",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

> ❌ Never use `"Action": "*"` or `"Resource": "*"` in production policies.

### Permission Boundary Template
```hcl
resource "aws_iam_policy" "developer_boundary" {
  name = "developer-permission-boundary"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:*", "lambda:*", "dynamodb:*", "logs:*"]
        Resource = "*"
      },
      {
        Effect   = "Deny"
        Action   = ["iam:CreateUser", "iam:DeleteUser", "organizations:*"]
        Resource = "*"
      }
    ]
  })
}
```

### Service Control Policy (SCP) — Prevent Region Usage
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllOutsideAllowedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "sts:*", "route53:*",
        "cloudfront:*", "waf:*", "support:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["us-east-1", "eu-west-1"]
        }
      }
    }
  ]
}
```

---

## KMS & Encryption

### Customer Managed Key with Key Policy
```hcl
resource "aws_kms_key" "app" {
  description             = "CMK for ${var.app_name} application data"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  multi_region            = false

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "Enable IAM User Permissions"
        Effect    = "Allow"
        Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
        Action    = "kms:*"
        Resource  = "*"
      },
      {
        Sid    = "Allow App Role Usage"
        Effect = "Allow"
        Principal = { AWS = aws_iam_role.app.arn }
        Action = ["kms:Decrypt", "kms:GenerateDataKey"]
        Resource = "*"
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_kms_alias" "app" {
  name          = "alias/${var.app_name}-key"
  target_key_id = aws_kms_key.app.key_id
}
```

---

## Secrets Manager

### Store and Rotate a Secret
```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "${var.app_name}/${var.environment}/db-password"
  description             = "RDS master password"
  recovery_window_in_days = 30
  kms_key_id              = aws_kms_key.app.arn

  tags = local.common_tags
}

resource "aws_secretsmanager_secret_rotation" "db_password" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = aws_lambda_function.rotation.arn

  rotation_rules {
    automatically_after_days = 30
  }
}
```

### Retrieve Secret in Python
```python
import boto3
import json
from botocore.exceptions import ClientError

def get_secret(secret_name: str, region: str = "us-east-1") -> dict:
    client = boto3.client("secretsmanager", region_name=region)
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response["SecretString"])
    except ClientError as e:
        error_code = e.response["Error"]["Code"]
        if error_code == "ResourceNotFoundException":
            raise ValueError(f"Secret {secret_name} not found")
        raise
```

---

## GuardDuty & Security Hub

### Enable GuardDuty (Terraform)
```hcl
resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs { enable = true }
    kubernetes {
      audit_logs { enable = true }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes { enable = true }
      }
    }
  }
}
```

### EventBridge Rule for GuardDuty High Severity
```hcl
resource "aws_cloudwatch_event_rule" "guardduty_high" {
  name        = "guardduty-high-severity"
  description = "Capture GuardDuty HIGH severity findings"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [{ numeric = [">=", 7] }]
    }
  })
}
```

---

## Security Checklist (Pre-Production)

### IAM
- [ ] No root account API keys exist
- [ ] MFA enabled on root and all admin users
- [ ] Access Analyzer enabled — no public S3 or IAM roles
- [ ] All humans use roles via SSO, not long-lived access keys
- [ ] Service accounts use IAM Roles (EC2 instance profile, IRSA, ECS task role)

### Network
- [ ] No security group with `0.0.0.0/0` for SSH (port 22) or RDP (port 3389)
- [ ] VPC Flow Logs enabled
- [ ] S3 Block Public Access enabled at account level
- [ ] All endpoints use HTTPS — ACM certificates with auto-renewal

### Data
- [ ] Encryption at rest enabled for all data stores (RDS, S3, DynamoDB, EBS)
- [ ] KMS CMK used for sensitive workloads (not aws/*)
- [ ] S3 state bucket: versioning ON, MFA delete ON
- [ ] Secrets in Secrets Manager — not in environment variables plaintext

### Monitoring
- [ ] CloudTrail enabled in all regions with log file integrity validation
- [ ] GuardDuty enabled
- [ ] Security Hub enabled with CIS AWS Foundations Benchmark
- [ ] High-severity findings alert to PagerDuty/SNS
