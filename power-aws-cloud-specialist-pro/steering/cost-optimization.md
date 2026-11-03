# Cost Optimization — Steering Guide

## When to load this file
Load when user mentions: cost, billing, savings plan, reserved instance, spot,
rightsizing, cost explorer, budget, waste, unused resources, cost allocation,
finops, optimization, over-provisioned, pricing.

---

## Cost Optimization Principles (Well-Architected)

1. **Implement Cloud Financial Management** — Ownership and accountability per team
2. **Adopt a consumption model** — Pay for what you use, not what you provision
3. **Measure overall efficiency** — Track cost per business metric
4. **Stop spending on undifferentiated heavy lifting** — Use managed services
5. **Analyze and attribute expenditure** — Use tagging + Cost Allocation Tags

---

## Quick Wins Checklist

### Compute
- [ ] Identify EC2 instances with < 10% CPU utilization over 14 days → rightsize or stop
- [ ] Convert On-Demand EC2 to Savings Plans (1-year) for stable workloads → ~30% savings
- [ ] Use Spot Instances for batch/dev/CI workloads → up to 90% savings
- [ ] Move dev/test EC2 to auto-stop schedules (nights/weekends) → ~65% savings
- [ ] Review Fargate tasks — consider EC2 launch type for high-throughput workloads

### Storage
- [ ] Enable S3 Intelligent-Tiering for buckets with unknown access patterns
- [ ] Set S3 Lifecycle Policies: Standard → Standard-IA (30d) → Glacier (90d) → Delete (365d)
- [ ] Delete unattached EBS volumes → use AWS Config rule `ec2-volume-inuse-check`
- [ ] Delete unused EBS snapshots older than 90 days
- [ ] Review EFS — if mostly cold data, enable EFS Intelligent-Tiering

### Databases
- [ ] Use Aurora Serverless v2 for variable workloads (dev, test, reporting)
- [ ] Enable Aurora auto-pause for non-production clusters (idle > 5 min)
- [ ] Review RDS Multi-AZ — disable in dev/test environments
- [ ] Use DynamoDB on-demand for unpredictable traffic; provisioned + auto-scaling for predictable
- [ ] Delete unused RDS snapshots (keep 7 days for dev, 30 days for prod)

### Networking
- [ ] Delete unused Elastic IPs (billed when not attached)
- [ ] Review NAT Gateway data transfer — consider VPC endpoints for S3, DynamoDB
- [ ] Analyze CloudFront cache hit ratio — low ratio = more origin fetches = higher cost
- [ ] Review unused/idle load balancers

---

## Savings Plans vs Reserved Instances

| | Compute Savings Plan | EC2 Instance Savings Plan | Reserved Instance |
|-|---------------------|--------------------------|-----------------|
| Flexibility | EC2, Lambda, Fargate | Specific EC2 family | Specific EC2 type+region |
| Discount | Up to 66% | Up to 72% | Up to 75% |
| Best for | Mixed compute | EC2-heavy, flexible AZ | Stable, known instance type |
| Commitment | 1 or 3 years | 1 or 3 years | 1 or 3 years |

> **Recommendation**: Start with Compute Savings Plans for flexibility. Layer EC2 Instance Savings Plans for your most stable workloads.

---

## Spot Instance Strategy for EKS (Karpenter)

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-workers
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      - key: node.kubernetes.io/instance-type
        operator: In
        values:
          - m5.xlarge
          - m5a.xlarge
          - m6i.xlarge
          - m6a.xlarge
          - m7i.xlarge
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 30s
```

---

## AWS Cost Anomaly Detection (Terraform)

```hcl
resource "aws_ce_anomaly_monitor" "service_monitor" {
  name         = "service-cost-monitor"
  monitor_type = "DIMENSIONAL"

  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "alerts" {
  name      = "cost-anomaly-alerts"
  frequency = "DAILY"

  monitor_arn_list = [aws_ce_anomaly_monitor.service_monitor.arn]

  subscriber {
    type    = "SNS"
    address = aws_sns_topic.cost_alerts.arn
  }

  threshold_expression {
    dimension {
      key           = "ANOMALY_TOTAL_IMPACT_PERCENTAGE"
      values        = ["20"]
      match_options = ["GREATER_THAN_OR_EQUAL"]
    }
  }
}
```

---

## Cost Allocation Tagging Strategy

Required tags for all resources:

| Tag Key | Example Values | Purpose |
|---------|---------------|---------|
| `Environment` | dev, staging, prod | Env-level cost split |
| `Project` | checkout-service | Product cost tracking |
| `Team` | platform, backend | Team accountability |
| `CostCenter` | CC-1234 | Finance allocation |
| `ManagedBy` | terraform, manual | Governance |

### Enforce Tags with SCP
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": ["ec2:RunInstances", "rds:CreateDBInstance"],
    "Resource": "*",
    "Condition": {
      "Null": {
        "aws:RequestTag/Environment": "true",
        "aws:RequestTag/Team": "true"
      }
    }
  }]
}
```

---

## Monthly Cost Review Queries (Cost Explorer CLI)

```bash
# Top 10 services this month
aws ce get-cost-and-usage \
  --time-period Start=$(date -d "first day of this month" +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups | sort_by(@, &Metrics.UnblendedCost.Amount) | reverse(@) | [:10]' \
  --output table

# EC2 Rightsizing recommendations
aws ce get-rightsizing-recommendation \
  --service EC2 \
  --configuration RecommendationTarget=CROSS_INSTANCE_FAMILY \
  --query 'RightsizingRecommendations[:5].{Resource:CurrentInstance.ResourceId,Action:RightsizingType,MonthlySavings:ModifyRecommendationDetail.TargetInstances[0].EstimatedMonthlySavings.Value}'
```
