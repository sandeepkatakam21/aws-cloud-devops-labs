# Blue/Green Deployment Pipeline for EKS

**Version: v1.0.0 - September 2025**

## Overview

This project implements a Blue/Green deployment strategy for Amazon Elastic Kubernetes Service (EKS) using AWS CodePipeline, CodeBuild, and Helm. The pipeline enables zero-downtime deployments by maintaining two identical production environments (Blue and Green) and switching traffic between them seamlessly.

## Architecture

### System Architecture Diagram

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Source Code   │───▶│   CodePipeline  │───▶│   CodeBuild     │
│   (GitHub)      │    │                 │    │   (Build & Test)│
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                ▼                        │
                       ┌─────────────────┐               │
                       │   Deploy Stage  │◀──────────────┘
                       └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │   EKS Cluster   │
                       │                 │
                       │  ┌───────────┐  │    ┌─────────────────┐
                       │  │   Blue    │  │    │  Application    │
                       │  │   Env     │◀─┼────│  Load Balancer  │
                       │  └───────────┘  │    │   (ALB/NLB)     │
                       │                 │    └─────────────────┘
                       │  ┌───────────┐  │
                       │  │   Green   │  │
                       │  │   Env     │  │
                       │  └───────────┘  │
                       └─────────────────┘
```

### Deployment Flow

1. **Code Commit** - Developer pushes code to GitHub repository
2. **Pipeline Trigger** - CodePipeline starts automatically on code changes
3. **Build Phase** - CodeBuild compiles, tests, and builds Docker image
4. **Deploy Phase** - Helm deploys to inactive environment (Blue or Green)
5. **Validation** - Health checks validate the new deployment
6. **Traffic Switch** - Load balancer routes traffic to newly deployed environment
7. **Cleanup** - Previous environment remains as standby for quick rollback

## Features

### Zero Downtime Deployments
- **Seamless Transitions**: Traffic switches instantly between environments
- **No Service Interruption**: Users experience zero downtime during deployments
- **Load Balancer Integration**: ALB/NLB handles traffic routing transparently

### Enhanced Safety
- **Pre-Production Validation**: New version runs in production-like environment before traffic switch
- **Isolated Testing**: Comprehensive health checks on Green environment before activation
- **Gradual Rollout**: Optional canary deployments before full traffic switch

### Quick Rollback Capability
- **Instant Recovery**: Switch back to previous version within seconds
- **Minimal Risk**: Previous version remains warm and ready
- **Automated Rollback**: Configurable health check failures trigger automatic rollback

### Full Automation
- **CI/CD Integration**: End-to-end automation from code commit to production
- **Infrastructure as Code**: Helm charts manage Kubernetes resources
- **Monitoring Integration**: CloudWatch and Prometheus metrics for deployment validation

## Setup

### Prerequisites

Before setting up this pipeline, ensure you have:

#### AWS Resources
- AWS Account with appropriate permissions
- Amazon EKS cluster (v1.24+) with worker nodes
- Application Load Balancer (ALB) or Network Load Balancer (NLB)
- IAM roles for CodePipeline and CodeBuild
- ECR repository for Docker images

#### Tools and Access
- AWS CLI configured with appropriate credentials
- kubectl configured for your EKS cluster
- Helm 3.x installed
- Docker for local testing
- GitHub repository with webhook permissions

#### Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "eks:ListClusters",
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    }
  ]
}
```

### Installation Steps

#### 1. Repository Integration

**Clone and Configure Repository:**
```bash
git clone <your-repository-url>
cd blue-green-eks
```

**Set up GitHub Webhook:**
- Navigate to your GitHub repository settings
- Add webhook URL pointing to CodePipeline
- Configure for push events on main branch

#### 2. CodePipeline Configuration

**Create Pipeline:**
```bash
aws codepipeline create-pipeline --cli-input-json file://pipeline-config.json
```

**Key Pipeline Stages:**
- **Source**: GitHub integration with webhook trigger
- **Build**: CodeBuild project for Docker image creation
- **Deploy**: Helm deployment to inactive environment
- **Test**: Health checks and validation
- **Switch**: Traffic routing to new environment

#### 3. Helm Chart Setup

**Install Helm Chart:**
```bash
# Add repository
helm repo add blue-green ./helm-chart

# Install to Blue environment
helm install myapp-blue blue-green/blue-green-app \
  --set environment=blue \
  --set image.tag=v1.0.0 \
  --namespace production

# Install to Green environment
helm install myapp-green blue-green/blue-green-app \
  --set environment=green \
  --set image.tag=v1.0.1 \
  --namespace production
```

**Helm Chart Structure:**
```
helm-chart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── configmap.yaml
└── README.md
```

#### 4. Load Balancer Configuration

**ALB Configuration Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blue-green-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-active
            port:
              number: 80
```

#### 5. Monitoring and Health Checks

**Health Check Endpoint:**
```bash
# Configure liveness and readiness probes
kubectl patch deployment myapp-green -p '{
  "spec": {
    "template": {
      "spec": {
        "containers": [{
          "name": "myapp",
          "livenessProbe": {
            "httpGet": {
              "path": "/health",
              "port": 8080
            },
            "initialDelaySeconds": 30
          }
        }]
      }
    }
  }
}'
```

### Deployment Process

#### Automated Deployment Flow

1. **Code Commit**: Developer pushes to main branch
2. **Pipeline Trigger**: CodePipeline starts automatically
3. **Build Phase**: CodeBuild creates and pushes Docker image
4. **Deploy Phase**: Helm deploys to inactive environment
5. **Validation**: Health checks confirm deployment success
6. **Traffic Switch**: Load balancer routes to new environment
7. **Cleanup**: Previous environment becomes standby

#### Manual Deployment Commands

```bash
# Deploy to Green environment
./deploy.sh green v2.0.0

# Validate deployment
./health-check.sh green

# Switch traffic to Green
./switch-traffic.sh green

# Rollback if needed
./rollback.sh blue
```

## Troubleshooting

### Best Practices

- **Environment Parity**: Ensure Blue and Green environments are identical
- **Health Check Coverage**: Implement comprehensive health checks for all critical components
- **Monitoring**: Set up proper monitoring and alerting for both environments
- **Database Considerations**: Plan for database schema changes and migrations
- **Resource Management**: Monitor resource usage to prevent environment drift
- **Security**: Apply security patches and updates to both environments consistently

### Common Issues and Solutions

#### Deployment Failures

**Symptoms:**
- Pods failing to start
- Image pull errors
- Configuration issues

**Troubleshooting Steps:**
```bash
# Check pod status
kubectl get pods -l environment=green

# View deployment logs
kubectl logs -l app=myapp,environment=green

# Describe failed pods
kubectl describe pod <pod-name>
```

#### Traffic Switching Issues

**Symptoms:**
- Traffic not routing to new environment
- Load balancer configuration errors
- Service discovery problems

**Troubleshooting Steps:**
```bash
# Verify service endpoints
kubectl get endpoints myapp-active

# Check ingress configuration
kubectl describe ingress blue-green-ingress
```

#### Health Check Failures

**Common Solutions:**
- Verify health check endpoints are responding correctly
- Check application startup time vs. health check timing
- Review resource limits and requests
- Validate network policies and security groups

#### Rollback Scenarios

**When to Rollback:**
- Health checks consistently failing
- Performance degradation detected
- Critical bugs discovered in production
- Database migration issues

**Rollback Process:**
```bash
# Immediate rollback
./rollback.sh blue

# Verify rollback success
kubectl get services -l app=myapp
```

## Changelog

### v1.0.0 - September 2025

#### Added
- Initial implementation of Blue/Green deployment pipeline
- AWS CodePipeline integration with GitHub
- Helm chart templates for Kubernetes deployments
- Application Load Balancer configuration
- Health check and monitoring setup
- Automated traffic switching mechanism
- Rollback capabilities
- Comprehensive documentation

#### Features
- Zero-downtime deployment strategy
- Automated CI/CD pipeline
- Infrastructure as Code with Helm
- Multi-environment support (Blue/Green)
- CloudWatch integration for monitoring
- GitHub webhook integration
- Docker image management with ECR

#### Technical Details
- EKS cluster compatibility (v1.24+)
- Helm 3.x support
- ALB/NLB load balancer integration
- IAM role-based permissions
- Kubernetes namespace isolation

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

### References and Further Learning

#### AWS Documentation
- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [AWS CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/latest/userguide/)
- [AWS CodeBuild User Guide](https://docs.aws.amazon.com/codebuild/latest/userguide/)
- [Application Load Balancer Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)

#### Kubernetes and Helm
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Kubernetes Blue-Green Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#blue-green-deployments)

#### Best Practices Resources
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)
- [GitOps with AWS](https://aws.amazon.com/blogs/containers/gitops-with-aws/)

#### Tutorials and Examples
- [EKS Workshop](https://www.eksworkshop.com/)
- [AWS CodeSuite Workshop](https://catalog.workshops.aws/codecommit/)
- [Helm Chart Development Guide](https://helm.sh/docs/chart_best_practices/)

#### Monitoring and Observability
- [Container Insights for EKS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/ContainerInsights.html)
- [Prometheus on EKS](https://docs.aws.amazon.com/eks/latest/userguide/prometheus.html)
- [AWS X-Ray for Kubernetes](https://docs.aws.amazon.com/xray/latest/devguide/xray-kubernetes.html)

### Contributing

Contributions are welcome! Please read our contributing guidelines and submit pull requests for any improvements. When contributing, please:

- Follow the existing code style and formatting
- Add tests for new features
- Update documentation as needed
- Ensure all tests pass before submitting
- Provide clear commit messages and pull request descriptions
