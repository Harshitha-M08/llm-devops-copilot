# Quick Reference Card

Essential commands and workflows for the CI/CD pipeline.

## Common Workflows

### Development Deployment
```bash
# Push to main branch triggers automatic deployment
git add .
git commit -m "feat: add new feature"
git push origin main

# Monitor deployment
# Go to: GitHub → Actions → deploy-dev workflow
```

### Staging Deployment
```bash
# Create staging tag
git tag v1.0.0-staging -m "Release candidate 1.0.0"
git push origin v1.0.0-staging

# Monitor deployment
# Go to: GitHub → Actions → deploy-staging workflow
```

### Production Deployment
```bash
# Create production tag
git tag v1.0.0 -m "Production release 1.0.0"
git push origin v1.0.0

# Approve deployment
# Go to: GitHub → Actions → deploy-prod workflow → Review deployments
```

## Script Commands

### Build Images
```bash
# Build single service
./scripts/build.sh -s api -v v1.0.0

# Build all services
./scripts/build.sh -s all -e prod -v v1.0.0 -p

# Build with cache
./scripts/build.sh -s web -v v1.0.0 -c
```

### Run Tests
```bash
# All tests with coverage
./scripts/test.sh -t all -c

# Unit tests only
./scripts/test.sh -t unit

# Integration tests
./scripts/test.sh -t integration -e staging

# Watch mode
./scripts/test.sh -t unit -w
```

### Deploy Services
```bash
# Deploy to development
./scripts/deploy.sh api dev v1.0.0

# Deploy all to staging
./scripts/deploy.sh all staging v1.0.0

# Dry run
./scripts/deploy.sh web prod v1.0.0 --dry-run

# Deploy with auto-rollback
./scripts/deploy.sh api prod v1.0.0 --rollback
```

### Rollback Deployments
```bash
# Rollback production API
./scripts/rollback.sh production api

# Rollback all staging services
./scripts/rollback.sh staging all

# Rollback to specific revision
./scripts/rollback.sh production web --revision 3

# Dry run rollback
./scripts/rollback.sh staging api --dry-run
```

### Health Checks
```bash
# Check production API
./scripts/health-check.sh production api

# Check all services in staging
./scripts/health-check.sh staging all

# Quick check
./scripts/health-check.sh dev web --quick

# Strict check
./scripts/health-check.sh production all --strict
```

## kubectl Commands

### View Deployments
```bash
# List all deployments
kubectl get deployments -n production

# Get deployment details
kubectl describe deployment api -n production

# View deployment history
kubectl rollout history deployment/api -n production
```

### View Pods
```bash
# List pods
kubectl get pods -n production

# Get pod details
kubectl describe pod <pod-name> -n production

# View pod logs
kubectl logs -f <pod-name> -n production

# View logs for all pods of a service
kubectl logs -l app=api -n production --tail=100
```

### View Services
```bash
# List services
kubectl get services -n production

# Get service details
kubectl describe service api -n production

# Check endpoints
kubectl get endpoints api -n production
```

### Debug Commands
```bash
# Get events
kubectl get events -n production --sort-by='.lastTimestamp'

# Get resource usage
kubectl top pods -n production

# Check pod status
kubectl get pods -n production -o wide

# Exec into pod
kubectl exec -it <pod-name> -n production -- /bin/sh
```

## Helm Commands

### List Releases
```bash
# List all releases
helm list -n production

# Get release history
helm history api -n production

# Get release status
helm status api -n production
```

### Manage Releases
```bash
# Rollback release
helm rollback api -n production

# Rollback to specific revision
helm rollback api 3 -n production

# Uninstall release
helm uninstall api -n production

# Upgrade release
helm upgrade api ./helm/api -n production --set image.tag=v1.0.0
```

## Git Commands

### Tagging
```bash
# Create annotated tag
git tag -a v1.0.0 -m "Release 1.0.0"

# Create staging tag
git tag v1.0.0-staging

# Push single tag
git push origin v1.0.0

# Push all tags
git push origin --tags

# Delete local tag
git tag -d v1.0.0

# Delete remote tag
git push origin :refs/tags/v1.0.0
```

### Branch Management
```bash
# Create feature branch
git checkout -b feature/new-feature

# Push feature branch
git push origin feature/new-feature

# Merge to main
git checkout main
git merge feature/new-feature
git push origin main
```

## Monitoring Commands

### Check Workflow Status
```bash
# Using GitHub CLI
gh run list --limit 10

# Watch specific workflow
gh run watch <run-id>

# View workflow logs
gh run view <run-id> --log
```

### Check Deployment Status
```bash
# Check rollout status
kubectl rollout status deployment/api -n production

# Check pod readiness
kubectl get pods -n production -l app=api -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}'

# Check service endpoints
kubectl get endpoints api -n production -o json | jq '.subsets[].addresses | length'
```

## Emergency Commands

### Quick Rollback
```bash
# Production emergency rollback
./scripts/rollback.sh production all

# Or using kubectl
kubectl rollout undo deployment/api -n production
```

### Scale Down/Up
```bash
# Scale down
kubectl scale deployment api --replicas=0 -n production

# Scale up
kubectl scale deployment api --replicas=5 -n production

# Auto-scale
kubectl autoscale deployment api --min=5 --max=20 --cpu-percent=70 -n production
```

### Emergency Maintenance
```bash
# Enable maintenance mode (if configured)
kubectl apply -f k8s/maintenance-mode.yaml -n production

# Disable maintenance mode
kubectl delete -f k8s/maintenance-mode.yaml -n production
```

## Troubleshooting

### Build Issues
```bash
# Check Docker
docker info
docker ps

# Clean Docker
docker system prune -a

# Rebuild without cache
./scripts/build.sh -s api -v test --no-cache
```

### Deployment Issues
```bash
# Check pod logs
kubectl logs <pod-name> -n production --previous

# Check events
kubectl get events -n production | grep Error

# Describe pod
kubectl describe pod <pod-name> -n production

# Check secrets
kubectl get secrets -n production
```

### Network Issues
```bash
# Test service connectivity
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -O- http://api:80/health

# Check DNS
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup api

# Port forward for testing
kubectl port-forward svc/api 8080:80 -n production
```

## Useful Aliases

Add to ~/.bashrc or ~/.zshrc:

```bash
# kubectl aliases
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kex='kubectl exec -it'

# Environment-specific
alias kdev='kubectl -n development'
alias kstg='kubectl -n staging'
alias kprod='kubectl -n production'

# Helm aliases
alias h='helm'
alias hls='helm list'
alias hst='helm status'
alias hhist='helm history'

# Script aliases
alias build='./scripts/build.sh'
alias test='./scripts/test.sh'
alias deploy='./scripts/deploy.sh'
alias rollback='./scripts/rollback.sh'
alias health='./scripts/health-check.sh'
```

## Environment Variables

### Common Variables
```bash
# Set environment
export ENVIRONMENT=production
export NAMESPACE=production

# Set registry
export REGISTRY=ghcr.io
export IMAGE_NAME=yourorg/yourapp

# Set cluster
export CLUSTER_NAME=prod-cluster
export AWS_REGION=us-east-1

# Set timeouts
export KUBECTL_TIMEOUT=5m
export HELM_TIMEOUT=10m
```

## Quick Checks

### Pre-Deployment Checklist
- [ ] All tests passing
- [ ] Security scans clean
- [ ] Staging deployment successful
- [ ] Database migrations ready
- [ ] Secrets configured
- [ ] Monitoring alerts set
- [ ] Rollback plan ready
- [ ] Team notified

### Post-Deployment Checklist
- [ ] Health checks passing
- [ ] All pods running
- [ ] Services responding
- [ ] No error spikes
- [ ] Metrics normal
- [ ] Logs clean
- [ ] Users notified
- [ ] Documentation updated

## Support Contacts

### Emergency
- On-call: Check PagerDuty
- DevOps: Slack #devops-alerts
- Security: Slack #security

### Non-Emergency
- Issues: GitHub Issues
- Questions: Slack #engineering
- Docs: Internal Wiki

---

**Tip:** Bookmark this page for quick reference during deployments!
