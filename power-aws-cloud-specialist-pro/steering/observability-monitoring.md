# Observability & Monitoring — Steering Guide

## When to load this file
Load when user mentions: cloudwatch, alarm, metric, logs, log group, log insights,
x-ray, tracing, dashboard, prometheus, grafana, container insights, sns, alert,
incident, pagerduty, datadog, anomaly detection, synthetic monitoring.

---

## CloudWatch Best Practices

### Structured Logging with AWS Lambda Powertools (Python)
```python
from aws_lambda_powertools import Logger, Tracer, Metrics
from aws_lambda_powertools.metrics import MetricUnit
from aws_lambda_powertools.utilities.typing import LambdaContext

logger = Logger(service="order-service")
tracer = Tracer(service="order-service")
metrics = Metrics(namespace="MyApp", service="order-service")

@logger.inject_lambda_context(log_event=True)
@tracer.capture_lambda_handler
@metrics.log_metrics(capture_cold_start_metric=True)
def handler(event: dict, context: LambdaContext) -> dict:
    order_id = event.get("order_id")
    
    logger.info("Processing order", extra={"order_id": order_id})
    
    with tracer.capture_method():
        result = process_order(order_id)
    
    metrics.add_metric(name="OrdersProcessed", unit=MetricUnit.Count, value=1)
    
    return {"statusCode": 200, "body": result}
```

### CloudWatch Metric Alarm (Terraform)
```hcl
resource "aws_cloudwatch_metric_alarm" "high_error_rate" {
  alarm_name          = "${var.service_name}-high-error-rate"
  alarm_description   = "Error rate exceeded 5% for 5 minutes"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "5XXError"
  namespace           = "AWS/ApplicationELB"
  period              = 300
  statistic           = "Average"
  threshold           = 5
  treat_missing_data  = "notBreaching"

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]

  tags = local.common_tags
}
```

### Anomaly Detection Alarm
```hcl
resource "aws_cloudwatch_metric_alarm" "latency_anomaly" {
  alarm_name          = "${var.service_name}-latency-anomaly"
  comparison_operator = "GreaterThanUpperThreshold"
  evaluation_periods  = 3
  threshold_metric_id = "e1"
  treat_missing_data  = "notBreaching"

  metric_query {
    id          = "e1"
    expression  = "ANOMALY_DETECTION_BAND(m1, 2)"
    label       = "Latency (Expected)"
    return_data = true
  }

  metric_query {
    id          = "m1"
    return_data = false
    metric {
      metric_name = "TargetResponseTime"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "p99"
      dimensions = {
        LoadBalancer = aws_lb.main.arn_suffix
      }
    }
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

## CloudWatch Log Insights Queries

### Lambda Error Rate
```
fields @timestamp, @message, @requestId
| filter @message like /ERROR/
| stats count(*) as errorCount by bin(5m)
| sort @timestamp desc
```

### ECS / Container Error Analysis
```
fields @timestamp, level, message, container_name
| filter level = "ERROR"
| stats count() as errors by container_name
| sort errors desc
| limit 20
```

### API Gateway 5XX by Endpoint
```
fields @timestamp, httpMethod, resourcePath, status
| filter status >= 500
| stats count() as error_count by httpMethod, resourcePath
| sort error_count desc
| limit 20
```

### VPC Flow Logs — Rejected Traffic
```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| stats count() as rejections by srcAddr, dstAddr, dstPort
| sort rejections desc
| limit 25
```

---

## Container Insights & EKS Observability

### Enable Container Insights for EKS (Terraform)
```hcl
resource "aws_cloudwatch_log_group" "eks_cluster" {
  name              = "/aws/eks/${var.cluster_name}/cluster"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.logs.arn
}

# CloudWatch Agent DaemonSet via Helm
resource "helm_release" "cloudwatch_agent" {
  name       = "aws-cloudwatch-metrics"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-cloudwatch-metrics"
  namespace  = "amazon-cloudwatch"
  create_namespace = true

  set {
    name  = "clusterName"
    value = var.cluster_name
  }
}
```

### Prometheus + Grafana on EKS (kube-prometheus-stack)
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword="${GRAFANA_ADMIN_PASSWORD}" \
  --set grafana.ingress.enabled=true
```

---

## SLI / SLO Definitions

### Key Metrics by Service Type

| Service Type | SLI | Recommended SLO |
|-------------|-----|----------------|
| API / REST | p99 latency | < 500ms |
| API / REST | Error rate | < 0.1% |
| Data pipeline | Processing lag | < 5 minutes |
| Batch job | Success rate | > 99.5% |
| Database | Connection pool usage | < 80% |
| ECS/EKS | Pod availability | > 99.9% |

### Composite Alarm (Multi-condition alerting)
```hcl
resource "aws_cloudwatch_composite_alarm" "service_degraded" {
  alarm_name = "${var.service_name}-service-degraded"
  alarm_description = "Service is degraded: high errors AND high latency"

  alarm_rule = "ALARM(${aws_cloudwatch_metric_alarm.high_error_rate.alarm_name}) AND ALARM(${aws_cloudwatch_metric_alarm.latency_anomaly.alarm_name})"

  alarm_actions = [aws_sns_topic.critical_alerts.arn]
}
```

---

## Incident Response Runbook Template

When an alarm fires, follow this structure:
1. **Acknowledge** — Confirm who owns the incident
2. **Assess** — Check CloudWatch dashboards and recent deployments
3. **Contain** — Scale up / roll back / enable circuit breaker
4. **Diagnose** — CloudWatch Log Insights, X-Ray traces, describe tasks/pods
5. **Resolve** — Apply fix, verify metrics return to baseline
6. **Post-mortem** — Document timeline, root cause, and action items within 48h
