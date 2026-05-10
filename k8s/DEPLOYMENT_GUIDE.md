# Quick Start Deployment Guide

This guide will help you quickly deploy the AI system to your Kubernetes cluster.

## Pre-flight Checklist

- [ ] Kubernetes cluster is running (v1.24+)
- [ ] kubectl is configured and can access your cluster
- [ ] NGINX Ingress Controller is installed
- [ ] cert-manager is installed (for TLS)
- [ ] Storage classes are available

## Step-by-Step Deployment

### 1. Install Prerequisites

#### Install NGINX Ingress Controller
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.2/deploy/static/provider/cloud/deploy.yaml
```

#### Install cert-manager
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

### 2. Update Configuration

#### Update Image Registry
Edit `base/kustomization.yaml` and update the image registry:
```yaml
images:
  - name: llm-service
    newName: your-registry.azurecr.io/llm-service  # <-- Update this
    newTag: latest
```

#### Update Secrets
Edit `base/secrets.yaml` and replace all `changeme-*` values with secure passwords:
```yaml
stringData:
  POSTGRES_PASSWORD: your-secure-password       # <-- Change
  REDIS_PASSWORD: your-redis-password           # <-- Change
  OPENAI_API_KEY: sk-your-actual-api-key        # <-- Change
```

#### Update Production Domains (for prod only)
Edit `overlays/prod/ingress-patch.yaml`:
```yaml
hosts:
  - ai-system.yourdomain.com        # <-- Your domain
  - api.ai-system.yourdomain.com    # <-- Your API domain
```

### 3. Deploy to Development

```bash
# Navigate to k8s directory
cd devops/k8s

# Deploy to dev
kubectl apply -k overlays/dev/

# Wait for pods to be ready
kubectl wait --for=condition=ready pod --all -n ai-system --timeout=300s

# Check status
kubectl get all -n ai-system
```

### 4. Verify Development Deployment

```bash
# Check pods are running
kubectl get pods -n ai-system

# Check services
kubectl get svc -n ai-system

# Check logs
kubectl logs -l app=llm-service -n ai-system --tail=50

# Port forward to access frontend (optional)
kubectl port-forward svc/approval-frontend 8080:80 -n ai-system
# Access at http://localhost:8080
```

### 5. Deploy to Staging

```bash
# Deploy to staging
kubectl apply -k overlays/staging/

# Check status
kubectl get all -n ai-system
```

### 6. Deploy to Production

```bash
# Review production settings
cat overlays/prod/kustomization.yaml
cat overlays/prod/replica-patch.yaml

# Deploy to production
kubectl apply -k overlays/prod/

# Monitor rollout
kubectl rollout status deployment/llm-service -n ai-system
kubectl rollout status deployment/worker-service -n ai-system
kubectl rollout status deployment/approval-backend -n ai-system

# Check all components
kubectl get all -n ai-system
kubectl get ingress -n ai-system
kubectl get hpa -n ai-system
```

### 7. Configure DNS

Point your domain to the Ingress Controller's external IP:

```bash
# Get the external IP
kubectl get svc -n ingress-nginx ingress-nginx-controller

# Create DNS A records:
# ai-system.yourdomain.com -> <EXTERNAL-IP>
# api.ai-system.yourdomain.com -> <EXTERNAL-IP>
```

### 8. Verify Production Deployment

```bash
# Check TLS certificate
kubectl get certificate -n ai-system

# Check ingress
kubectl describe ingress ai-system-ingress -n ai-system

# Test endpoints
curl https://ai-system.yourdomain.com/health
curl https://api.ai-system.yourdomain.com/api/health
```

## Common Issues and Solutions

### Issue: Pods in ImagePullBackOff
**Solution**: Check image names and ensure your registry credentials are configured
```bash
kubectl describe pod <pod-name> -n ai-system
```

### Issue: Pods in CrashLoopBackOff
**Solution**: Check logs and environment variables
```bash
kubectl logs <pod-name> -n ai-system
kubectl describe pod <pod-name> -n ai-system
```

### Issue: Services not accessible
**Solution**: Check service endpoints and network policies
```bash
kubectl get endpoints -n ai-system
kubectl describe networkpolicy -n ai-system
```

### Issue: Database connection failures
**Solution**: Check PostgreSQL is running and credentials are correct
```bash
kubectl logs postgres-0 -n ai-system
kubectl exec -it postgres-0 -n ai-system -- psql -U ai_user -d ai_system -c "SELECT 1"
```

### Issue: RabbitMQ cluster not forming
**Solution**: Check RabbitMQ logs and peer discovery
```bash
kubectl logs rabbitmq-0 -n ai-system
kubectl exec -it rabbitmq-0 -n ai-system -- rabbitmqctl cluster_status
```

## Rollback

If something goes wrong:

```bash
# Rollback specific deployment
kubectl rollout undo deployment/llm-service -n ai-system

# Or delete and redeploy previous version
kubectl delete -k overlays/prod/
kubectl apply -k overlays/prod/  # Make sure to revert changes first
```

## Clean Up

### Delete specific environment
```bash
kubectl delete -k overlays/dev/
kubectl delete -k overlays/staging/
kubectl delete -k overlays/prod/
```

### Complete removal
```bash
kubectl delete namespace ai-system
```

## Useful Commands

```bash
# Watch all resources
kubectl get all -n ai-system -w

# Follow logs
kubectl logs -f deployment/llm-service -n ai-system

# Execute command in pod
kubectl exec -it <pod-name> -n ai-system -- /bin/sh

# Get events
kubectl get events -n ai-system --sort-by='.lastTimestamp'

# Check resource usage
kubectl top pods -n ai-system
kubectl top nodes

# Scale manually
kubectl scale deployment llm-service --replicas=5 -n ai-system

# Update image
kubectl set image deployment/llm-service llm-service=new-image:tag -n ai-system
```

## Production Checklist

Before going to production:

- [ ] All secrets are properly secured (not using default values)
- [ ] Image tags are pinned to specific versions (not using `latest`)
- [ ] Resource limits are appropriate for workload
- [ ] Monitoring and alerting are configured
- [ ] Backup strategy is in place
- [ ] DNS records are configured
- [ ] TLS certificates are valid
- [ ] Network policies are tested
- [ ] Disaster recovery plan is documented
- [ ] Team is trained on troubleshooting procedures

## Next Steps

1. Set up monitoring with Prometheus/Grafana
2. Configure log aggregation with ELK/Loki
3. Set up CI/CD pipelines
4. Configure automated backups
5. Implement GitOps with ArgoCD/Flux
6. Set up alerting rules
7. Document runbooks for common issues
8. Conduct load testing
9. Review and tune resource limits
10. Plan for scaling and capacity

## Support

For help:
- Review logs: `kubectl logs <pod-name> -n ai-system`
- Check events: `kubectl get events -n ai-system`
- Contact: devops@example.com
