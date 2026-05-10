# CI/CD Setup Guide

Quick start guide for setting up the complete CI/CD pipeline.

## Quick Setup Checklist

### 1. Prerequisites

- [ ] GitHub repository created
- [ ] Kubernetes cluster(s) set up (dev, staging, production)
- [ ] Container registry configured (GitHub Container Registry recommended)
- [ ] kubectl installed and configured
- [ ] Helm 3 installed
- [ ] AWS EKS access configured (if using AWS)

### 2. Repository Setup

#### Copy CI/CD Files
```bash
# Copy the entire devops/ci-cd directory to your repository root
cp -r devops/ci-cd/.github .
mkdir -p scripts
cp devops/ci-cd/scripts/* scripts/
```

#### Directory Structure
```
your-repo/
├── .github/
│   └── workflows/
│       ├── build-and-test.yml
│       ├── deploy-dev.yml
│       ├── deploy-staging.yml
│       ├── deploy-prod.yml
│       ├── security-scan.yml
│       └── dependency-update.yml
├── scripts/
│   ├── build.sh
│   ├── test.sh
│   ├── deploy.sh
│   ├── rollback.sh
│   └── health-check.sh
├── docker/
│   ├── Dockerfile.api
│   ├── Dockerfile.web
│   └── Dockerfile.worker
├── helm/
│   ├── api/
│   ├── web/
│   └── worker/
└── k8s/
    ├── dev/
    ├── staging/
    └── production/
```

### 3. Configure GitHub Secrets

Go to: Repository Settings → Secrets and variables → Actions → New repository secret

#### Required Secrets
```bash
# AWS Configuration
AWS_ROLE_ARN=arn:aws:iam::ACCOUNT:role/GithubActionsRole
AWS_PROD_ROLE_ARN=arn:aws:iam::ACCOUNT:role/GithubActionsProdRole
AWS_REGION=us-east-1

# Database & Services (Staging)
STAGING_DATABASE_URL=postgresql://user:pass@host:5432/db
STAGING_REDIS_URL=redis://host:6379
STAGING_API_KEY=your-staging-api-key

# Database & Services (Production)
PROD_DATABASE_URL=postgresql://user:pass@host:5432/db
PROD_REDIS_URL=redis://host:6379
PROD_API_KEY=your-production-api-key
PROD_JWT_SECRET=your-jwt-secret

# Security Tools
SNYK_TOKEN=your-snyk-token
CODECOV_TOKEN=your-codecov-token
FOSSA_API_KEY=your-fossa-key

# Notifications
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

### 4. Configure GitHub Environments

#### Development Environment
1. Go to: Settings → Environments → New environment
2. Name: `development`
3. Add environment URL: `https://dev.example.com`
4. No protection rules needed

#### Staging Environment
1. Name: `staging`
2. Add environment URL: `https://staging.example.com`
3. Protection rules (optional):
   - Required reviewers: 1
   - Wait timer: 0 minutes

#### Production Environment
1. Name: `production`
2. Add environment URL: `https://example.com`
3. Protection rules:
   - Required reviewers: 2 (DevOps team)
   - Deployment branches: Only protected branches

#### Production Approval Environment
1. Name: `production-approval`
2. Protection rules:
   - Required reviewers: 2+ (Senior engineers, DevOps)
   - Required approval from code owners

### 5. Create Dockerfiles

Create these files in the `docker/` directory:

**docker/Dockerfile.api**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

ARG BUILD_DATE
ARG VCS_REF
ARG VERSION
LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.version=$VERSION

EXPOSE 3000
CMD ["node", "dist/api/main.js"]
```

**docker/Dockerfile.web**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build:web

FROM nginx:alpine
COPY --from=builder /app/dist/web /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

ARG BUILD_DATE
ARG VCS_REF
ARG VERSION
LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.version=$VERSION

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**docker/Dockerfile.worker**
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package*.json ./

ARG BUILD_DATE
ARG VCS_REF
ARG VERSION
LABEL org.opencontainers.image.created=$BUILD_DATE \
      org.opencontainers.image.revision=$VCS_REF \
      org.opencontainers.image.version=$VERSION

CMD ["node", "dist/worker/main.js"]
```

### 6. Create Helm Charts

Create basic Helm charts for each service:

**helm/api/Chart.yaml**
```yaml
apiVersion: v2
name: api
description: API service Helm chart
type: application
version: 1.0.0
appVersion: "1.0.0"
```

**helm/api/values.yaml**
```yaml
replicaCount: 2

image:
  repository: ghcr.io/yourorg/yourapp-api
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80

env:
  - name: NODE_ENV
    value: production
  - name: PORT
    value: "3000"
```

**helm/api/templates/deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "api.fullname" . }}
  labels:
    {{- include "api.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "api.selectorLabels" . | nindent 8 }}
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: {{ .Values.service.targetPort }}
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
        env:
          {{- toYaml .Values.env | nindent 12 }}
```

Repeat similar structure for web and worker services.

### 7. Update Workflow Configuration

Edit workflow files to match your setup:

#### Update Registry and Image Names
```yaml
# In all workflow files, update:
env:
  REGISTRY: ghcr.io  # or your registry
  IMAGE_NAME: yourorg/yourapp  # your image name
```

#### Update Cluster Names
```yaml
# In deploy-*.yml files, update:
env:
  CLUSTER_NAME: your-cluster-name
  NAMESPACE: your-namespace
```

### 8. Configure Package.json Scripts

Add these scripts to your `package.json`:

```json
{
  "scripts": {
    "build": "tsc",
    "test": "jest",
    "test:unit": "jest --testPathPattern=unit",
    "test:integration": "jest --testPathPattern=integration",
    "test:e2e": "jest --testPathPattern=e2e",
    "test:smoke": "jest --testPathPattern=smoke",
    "test:coverage": "jest --coverage",
    "lint": "eslint . --ext .ts,.tsx,.js,.jsx",
    "format:check": "prettier --check .",
    "format": "prettier --write ."
  }
}
```

### 9. Test the Pipeline

#### Test Build Locally
```bash
# Test Docker build
./scripts/build.sh --service api --version test --dry-run

# Test all builds
./scripts/build.sh --service all --version test
```

#### Test Scripts
```bash
# Test script execution
./scripts/test.sh --type unit
./scripts/health-check.sh dev api --quick
```

#### Test Deployment (Dry Run)
```bash
# Test deployment
./scripts/deploy.sh api dev v1.0.0 --dry-run
```

### 10. Deploy to Development

1. **Push to main branch**
   ```bash
   git add .
   git commit -m "Add CI/CD pipeline"
   git push origin main
   ```

2. **Monitor workflow**
   - Go to Actions tab in GitHub
   - Watch build-and-test workflow
   - Watch deploy-dev workflow

3. **Verify deployment**
   ```bash
   kubectl get pods -n development
   kubectl get services -n development
   ```

### 11. Deploy to Staging

1. **Create staging tag**
   ```bash
   git tag v1.0.0-staging
   git push origin v1.0.0-staging
   ```

2. **Monitor deployment**
   - Check Actions tab
   - Verify staging environment

### 12. Deploy to Production

1. **Create production tag**
   ```bash
   git tag v1.0.0
   git push origin v1.0.0
   ```

2. **Approve deployment**
   - Go to Actions tab
   - Find the deployment workflow
   - Review and approve

3. **Monitor production**
   - Watch deployment progress
   - Check health checks
   - Monitor application logs

## Common Customizations

### Add New Service

1. Create Dockerfile: `docker/Dockerfile.newservice`
2. Create Helm chart: `helm/newservice/`
3. Update workflows to include new service in matrix
4. Update deploy.sh to include new service

### Change Deployment Strategy

Edit `deploy-prod.yml`:
```yaml
# For blue-green deployment
strategy:
  type: BlueGreen

# For canary deployment
strategy:
  canary:
    steps:
      - setWeight: 20
      - pause: {duration: 5m}
      - setWeight: 50
      - pause: {duration: 5m}
      - setWeight: 100
```

### Add Custom Health Check

Edit `health-check.sh` to add custom checks:
```bash
check_custom_metric() {
    local service=$1
    # Add your custom health check logic
}
```

### Customize Notifications

Edit workflow files to add more notification channels:
```yaml
- name: Send notification
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: Custom notification message
```

## Troubleshooting

### Workflows Not Triggering
- Check branch protection rules
- Verify workflow syntax with: `actionlint`
- Check repository permissions

### Docker Build Fails
- Verify Dockerfile paths
- Check Docker daemon is running
- Review build logs

### Deployment Fails
- Check cluster connectivity
- Verify namespace exists
- Check RBAC permissions
- Review pod logs

### Health Checks Fail
- Verify health endpoints exist
- Check service configuration
- Review application logs

## Next Steps

1. Set up monitoring (Prometheus, Grafana)
2. Configure alerting (PagerDuty, OpsGenie)
3. Add performance testing
4. Set up log aggregation (ELK, Splunk)
5. Configure backup/disaster recovery
6. Document runbooks

## Support

For issues or questions:
1. Check the main README.md
2. Review workflow logs
3. Check Kubernetes events
4. Contact DevOps team

## Maintenance

Regular tasks:
- [ ] Review and update dependencies weekly
- [ ] Review security scan results daily
- [ ] Test rollback procedures monthly
- [ ] Update documentation as needed
- [ ] Review and optimize resource allocations quarterly
