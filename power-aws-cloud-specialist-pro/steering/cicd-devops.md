# CI/CD & DevOps — Steering Guide

## When to load this file
Load when user mentions: codepipeline, codebuild, codedeploy, codecommit, pipeline,
argocd, gitops, github actions, jenkins, gitlab ci, deployment, blue/green,
canary, rollback, artifact, buildspec, ecr push, helm deploy, cicd.

---

## CodePipeline for ECS (Blue/Green Deployment)

### Buildspec for Docker Image Build + ECR Push
```yaml
# buildspec.yml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPO_URI
      - COMMIT_HASH=$(echo $CODEBUILD_RESOLVED_SOURCE_VERSION | cut -c 1-7)
      - IMAGE_TAG=${COMMIT_HASH:-latest}

  build:
    commands:
      - echo Building Docker image...
      - docker build -t $ECR_REPO_URI:$IMAGE_TAG -t $ECR_REPO_URI:latest .
      - echo Running security scan...
      - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image --exit-code 1 --severity HIGH,CRITICAL $ECR_REPO_URI:$IMAGE_TAG

  post_build:
    commands:
      - echo Pushing image to ECR...
      - docker push $ECR_REPO_URI:$IMAGE_TAG
      - docker push $ECR_REPO_URI:latest
      - printf '[{"name":"%s","imageUri":"%s"}]' $CONTAINER_NAME $ECR_REPO_URI:$IMAGE_TAG > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
    - appspec.yaml
    - taskdef.json
```

### CodePipeline Terraform
```hcl
resource "aws_codepipeline" "app" {
  name     = "${var.app_name}-pipeline"
  role_arn = aws_iam_role.codepipeline.arn

  artifact_store {
    location = aws_s3_bucket.artifacts.bucket
    type     = "S3"
    encryption_key {
      id   = aws_kms_key.pipeline.arn
      type = "KMS"
    }
  }

  stage {
    name = "Source"
    action {
      name             = "Source"
      category         = "Source"
      owner            = "ThirdParty"
      provider         = "GitHub"
      version          = "2"
      output_artifacts = ["source_output"]
      configuration = {
        ConnectionArn    = aws_codestarconnections_connection.github.arn
        FullRepositoryId = var.github_repo
        BranchName       = var.branch
      }
    }
  }

  stage {
    name = "Build"
    action {
      name             = "Build"
      category         = "Build"
      owner            = "AWS"
      provider         = "CodeBuild"
      version          = "1"
      input_artifacts  = ["source_output"]
      output_artifacts = ["build_output"]
      configuration = {
        ProjectName = aws_codebuild_project.app.name
      }
    }
  }

  stage {
    name = "Deploy"
    action {
      name            = "Deploy"
      category        = "Deploy"
      owner           = "AWS"
      provider        = "CodeDeployToECS"
      version         = "1"
      input_artifacts = ["build_output"]
      configuration = {
        ApplicationName                = aws_codedeploy_app.app.name
        DeploymentGroupName            = aws_codedeploy_deployment_group.app.deployment_group_name
        TaskDefinitionTemplateArtifact = "build_output"
        AppSpecTemplateArtifact        = "build_output"
      }
    }
  }
}
```

---

## ArgoCD (GitOps for EKS)

### ArgoCD Application Manifest
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-service
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: production

  source:
    repoURL: https://github.com/myorg/k8s-manifests.git
    targetRevision: main
    path: services/my-service/overlays/production

  destination:
    server: https://kubernetes.default.svc
    namespace: production

  syncPolicy:
    automated:
      prune: true        # Auto-delete removed resources
      selfHeal: true     # Auto-sync if cluster drifts from git
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### Kustomize Overlay Structure (GitOps)
```
k8s-manifests/
└── services/
    └── my-service/
        ├── base/
        │   ├── kustomization.yaml
        │   ├── deployment.yaml
        │   ├── service.yaml
        │   └── hpa.yaml
        └── overlays/
            ├── staging/
            │   ├── kustomization.yaml  # patches staging values
            │   └── replica-patch.yaml
            └── production/
                ├── kustomization.yaml
                └── replica-patch.yaml
```

---

## GitHub Actions — EKS Deploy

```yaml
# .github/workflows/deploy.yml
name: Deploy to EKS

on:
  push:
    branches: [main]

permissions:
  id-token: write   # OIDC for AWS
  contents: read

env:
  AWS_REGION: us-east-1
  ECR_REPO: ${{ secrets.ECR_REPO }}
  EKS_CLUSTER: ${{ secrets.EKS_CLUSTER }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production

    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build, scan & push
      run: |
        IMAGE_TAG=${GITHUB_SHA::7}
        docker build -t $ECR_REPO:$IMAGE_TAG .
        # Trivy vulnerability scan — fail on HIGH/CRITICAL
        docker run --rm aquasec/trivy image --exit-code 1 \
          --severity HIGH,CRITICAL $ECR_REPO:$IMAGE_TAG
        docker push $ECR_REPO:$IMAGE_TAG
        echo "IMAGE_TAG=$IMAGE_TAG" >> $GITHUB_ENV

    - name: Update kubeconfig
      run: |
        aws eks update-kubeconfig --name $EKS_CLUSTER --region $AWS_REGION

    - name: Deploy with Helm
      run: |
        helm upgrade --install my-service ./helm/my-service \
          --namespace production \
          --create-namespace \
          --set image.tag=$IMAGE_TAG \
          --set image.repository=$ECR_REPO \
          --atomic \
          --timeout 5m \
          --wait
```

---

## Deployment Strategies

| Strategy | Description | Rollback Speed | Traffic Impact | Use Case |
|----------|-------------|---------------|----------------|----------|
| Rolling Update | Replaces pods incrementally | Slow | None (gradual) | Default K8s |
| Blue/Green | Two live environments, switch traffic | Instant | None | ECS with CodeDeploy |
| Canary | Route % of traffic to new version | Fast | Minimal | Progressive delivery |
| Recreate | Kill all, then deploy new | N/A (downtime) | Full downtime | Dev only |

### ECS Blue/Green AppSpec
```yaml
# appspec.yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 8080
        PlatformVersion: "LATEST"
Hooks:
  - BeforeAllowTraffic: "arn:aws:lambda:us-east-1:ACCOUNT:function:pre-traffic-validation"
  - AfterAllowTraffic: "arn:aws:lambda:us-east-1:ACCOUNT:function:post-traffic-check"
```
