# Architecture & Well-Architected — Steering Guide

## When to load this file
Load when user mentions: architecture, well-architected, design, multi-region, 
high availability, disaster recovery, rto, rpo, resilience, fault tolerance,
scalability, migration, lift and shift, refactoring, microservices, event-driven.

---

## AWS Well-Architected Framework — Quick Reference

### Six Pillars Summary

| Pillar | Core Question | Key AWS Services |
|--------|--------------|-----------------|
| **Operational Excellence** | Can you run and monitor systems to deliver value and continually improve? | CloudWatch, Systems Manager, CodePipeline |
| **Security** | How do you protect information, systems, and assets? | IAM, KMS, GuardDuty, Security Hub, WAF |
| **Reliability** | How do you recover from failures and scale to meet demand? | Multi-AZ, Auto Scaling, Route 53, Backup |
| **Performance Efficiency** | How do you use resources efficiently? | Auto Scaling, CloudFront, ElastiCache, Graviton |
| **Cost Optimization** | How do you avoid unnecessary costs? | Cost Explorer, Savings Plans, Spot, S3 Intelligent-Tiering |
| **Sustainability** | How do you minimize environmental impact? | Graviton, Spot, efficient storage tiering |

---

## Multi-Region Architecture Pattern

### Active-Passive (Pilot Light)
```
Region A (Primary)                Region B (DR)
├── VPC                           ├── VPC (mirrored)
├── EKS/ECS (active)              ├── EKS/ECS (scaled to 0 or minimal)
├── RDS Aurora (writer)  ─────────├── Aurora (read replica → failover)
├── S3 (source) ──────────────────├── S3 (CRR replicated)
├── Route 53 (primary)            └── Route 53 (failover record)
└── CloudFront (global)
```

**RTO**: 15–60 minutes | **RPO**: < 1 minute (Aurora Global DB)

### Active-Active (Multi-Region)
```
CloudFront (Global Edge)
    │
    ├── Route 53 Latency Routing
    │       ├── us-east-1 ALB → EKS → Aurora (writer)
    │       └── eu-west-1 ALB → EKS → Aurora Global (writer)
    │
    └── DynamoDB Global Tables (multi-master)
        S3 CRR (bidirectional)
```

**RTO**: < 1 minute | **RPO**: Near-zero

---

## High Availability Checklist

### Compute
- [ ] Deploy across minimum 3 AZs in production
- [ ] Auto Scaling Group with health checks replacing unhealthy instances
- [ ] ECS/EKS with PodAntiAffinity or ECS spread placement across AZs
- [ ] ALB with sticky sessions only when stateless architecture is not possible

### Database
- [ ] RDS Multi-AZ enabled (synchronous replication)
- [ ] Aurora Global Database for cross-region HA
- [ ] ElastiCache: Redis cluster mode with replicas across AZs
- [ ] DynamoDB: Automatically multi-AZ; use Global Tables for multi-region

### Networking
- [ ] NAT Gateway: one per AZ (not single NAT)
- [ ] VPC endpoints for S3, DynamoDB (no single-point internet dependency)
- [ ] Route 53 health checks with failover routing

---

## RTO / RPO Target Guide

| Tier | Business Impact | RTO Target | RPO Target | Strategy |
|------|----------------|------------|------------|----------|
| Tier 0 | Revenue-critical | < 1 min | Near-zero | Active-Active multi-region |
| Tier 1 | Business-critical | < 15 min | < 1 min | Warm Standby / Aurora Global |
| Tier 2 | Important | < 1 hour | < 15 min | Pilot Light |
| Tier 3 | Supporting | < 4 hours | < 1 hour | Backup & Restore |

---

## Event-Driven Architecture Pattern

```
API Gateway → Lambda → EventBridge → Multiple Consumers
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
               Lambda      SQS →    Kinesis →
            (notification) Lambda  (analytics)
                           (orders)
```

### EventBridge Rule (Terraform)
```hcl
resource "aws_cloudwatch_event_rule" "order_created" {
  name        = "order-created"
  description = "Route order.created events"

  event_pattern = jsonencode({
    source      = ["com.myapp.orders"]
    detail-type = ["order.created"]
    detail = {
      status = ["PENDING"]
    }
  })
}

resource "aws_cloudwatch_event_target" "process_order" {
  rule      = aws_cloudwatch_event_rule.order_created.name
  target_id = "ProcessOrder"
  arn       = aws_sqs_queue.order_processing.arn

  dead_letter_config {
    arn = aws_sqs_queue.order_dlq.arn
  }

  retry_policy {
    maximum_event_age_in_seconds = 3600
    maximum_retry_attempts       = 3
  }
}
```

---

## Migration Framework (AWS 7Rs)

| Strategy | Description | Effort | Risk | Best For |
|----------|-------------|--------|------|----------|
| **Retire** | Decommission unused apps | Low | Low | Shadow IT, duplicates |
| **Retain** | Keep on-premises for now | None | None | Compliance, not ready |
| **Rehost** (Lift & Shift) | Move as-is to EC2 | Low | Low | Speed, large volumes |
| **Relocate** | Move VMs to VMware Cloud on AWS | Low | Low | VMware environments |
| **Repurchase** | Move to SaaS (e.g., Salesforce) | Medium | Medium | Legacy CRM/ERP |
| **Replatform** | Migrate to managed service (RDS, ECS) | Medium | Medium | DB migrations, containers |
| **Refactor / Re-architect** | Redesign as cloud-native | High | High | Business value, agility |

---

## Architecture Review Template

When reviewing an architecture, always assess:

1. **Single Points of Failure** — Any component that, if it fails, brings down the system
2. **Blast Radius** — What's the impact of each component failing
3. **Scalability Bottlenecks** — Where does the system break at 10x load
4. **Data Flow & Latency** — Are there unnecessary hops, synchronous calls that could be async
5. **Security Surface** — What's exposed to the internet, what can reach what
6. **Operational Complexity** — Can a team of N maintain and troubleshoot this at 2 AM
7. **Cost Efficiency** — Is the architecture over-provisioned for its actual load
8. **Observability** — Can you detect, diagnose, and resolve incidents from dashboards alone
