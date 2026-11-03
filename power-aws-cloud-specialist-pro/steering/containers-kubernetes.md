# Containers, ECS, EKS & Kubernetes — Steering Guide

## When to load this file
Load when user mentions: docker, dockerfile, container, ecs, eks, kubernetes, k8s,
kubectl, helm, pod, deployment, service, ingress, fargate, ecr, karpenter,
cluster autoscaler, hpa, vpa, daemonset, statefulset, configmap, secret.

---

## Docker Best Practices

### Optimized Multi-Stage Dockerfile
```dockerfile
# ---- Build Stage ----
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ---- Runtime Stage ----
FROM python:3.12-slim AS runtime

# Security: non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Copy only what's needed
COPY --from=builder /install /usr/local
COPY --chown=appuser:appuser src/ .

USER appuser

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["python", "main.py"]
```

> Always pin base image versions. Use `docker scout` or `trivy` for vulnerability scanning.

---

## ECS Best Practices

### Task Definition Template (Fargate)
```json
{
  "family": "my-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecs-execution-role",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/ecs-task-role",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-service:latest",
      "essential": true,
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-service",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3
      },
      "readonlyRootFilesystem": true,
      "user": "1000"
    }
  ]
}
```

### ECS Auto Scaling with Target Tracking
```hcl
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = 20
  min_capacity       = 2
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu_tracking" {
  name               = "cpu-target-tracking"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

---

## Kubernetes Best Practices

### Production Deployment Template
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  namespace: production
  labels:
    app: my-service
    version: "1.0.0"
    team: platform
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-service
    spec:
      serviceAccountName: my-service-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["my-service"]
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-service:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
```

### IRSA (IAM Roles for Service Accounts) — EKS
```hcl
# Terraform: create the IRSA role
module "irsa" {
  source = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name = "my-service-irsa"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["production:my-service-sa"]
    }
  }

  role_policy_arns = {
    s3 = aws_iam_policy.my_service_s3.arn
  }
}
```

```yaml
# Kubernetes: annotate the ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-sa
  namespace: production
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/my-service-irsa
```

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

---

## ECS vs EKS Decision Guide

| Criterion | Choose ECS | Choose EKS |
|-----------|-----------|-----------|
| Team K8s expertise | Low | High |
| Complexity tolerance | Low/Medium | Medium/High |
| AWS-native integration | Strong preference | Flexible |
| Multi-cloud portability | Not needed | Required |
| Custom scheduling needs | Basic | Advanced |
| Ecosystem (Helm, ArgoCD) | Not required | Required |
| Operational overhead | Minimal | Moderate |

---

## Troubleshooting Containers

### ECS Task fails to start
```bash
# Get stopped task details
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks <task-arn> \
  --query 'tasks[0].{status:lastStatus,reason:stoppedReason,containers:containers[*].{name:name,reason:reason,exitCode:exitCode}}'

# View CloudWatch logs
aws logs get-log-events \
  --log-group-name /ecs/my-service \
  --log-stream-name ecs/app/<task-id>
```

### Pod CrashLoopBackOff
```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
```

### OOMKilled — Memory Issues
```bash
# Check resource usage
kubectl top pods -n <namespace>
kubectl top nodes

# Check VPA recommendations
kubectl describe vpa <vpa-name> -n <namespace>
```
