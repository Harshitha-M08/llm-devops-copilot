# Deployment Guide - AI-Driven Hybrid Kubernetes System

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Infrastructure Setup](#infrastructure-setup)
3. [Secrets Management](#secrets-management)
4. [Database Initialization](#database-initialization)
5. [Service Deployment](#service-deployment)
6. [Environment-Specific Deployment](#environment-specific-deployment)
7. [Post-Deployment Verification](#post-deployment-verification)
8. [Rollback Procedures](#rollback-procedures)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

### Required Tools

#### 1. Kubernetes Cluster
- Kubernetes version 1.24 or higher
- kubectl CLI tool installed and configured
- Cluster with at least:
  - **Dev**: 3 nodes (2 vCPU, 4GB RAM each)
  - **Staging**: 5 nodes (4 vCPU, 8GB RAM each)
  - **Production**: 10+ nodes (8 vCPU, 16GB RAM each)

#### 2. Helm
```bash
# Install Helm 3.x
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify installation
helm version
```

#### 3. kubectl
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

#### 4. Container Registry Access
- Docker Hub account OR
- AWS ECR credentials OR
- Google GCR credentials OR
- Private registry access

#### 5. Additional Tools
```bash
# Install kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash

# Install kubectx and kubens (optional but recommended)
sudo apt install kubectx

# Install k9s for cluster management (optional)
curl -sS https://webinstall.dev/k9s | bash
```

### Required Credentials

1. **OpenAI API Key** - Get from https://platform.openai.com/api-keys
2. **Anthropic API Key** - Get from https://console.anthropic.com/
3. **Qdrant Cloud Account** - Sign up at https://qdrant.io/ (free tier available)
4. **SMTP Credentials** - For email notifications (Gmail, SendGrid, or AWS SES)
5. **Slack Webhook** (optional) - For Slack notifications
6. **Domain Name** - For production deployment with TLS

### Verify Prerequisites

```bash
# Check Kubernetes cluster access
kubectl cluster-info
kubectl get nodes

# Check if you have cluster-admin permissions
kubectl auth can-i create deployments --all-namespaces

# Verify Helm installation
helm list --all-namespaces

# Check available storage classes
kubectl get storageclass
```

---

## Infrastructure Setup

### 1. Cloud Provider Setup

#### AWS EKS

```bash
# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Create EKS cluster
eksctl create cluster \
  --name ai-system-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.xlarge \
  --nodes 3 \
  --nodes-min 3 \
  --nodes-max 10 \
  --managed

# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name ai-system-cluster

# Verify cluster
kubectl get nodes
```

#### GCP GKE

```bash
# Install gcloud SDK
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# Initialize gcloud
gcloud init

# Create GKE cluster
gcloud container clusters create ai-system-cluster \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type n1-standard-4 \
  --enable-autoscaling \
  --min-nodes 3 \
  --max-nodes 10 \
  --enable-autorepair \
  --enable-autoupgrade

# Get credentials
gcloud container clusters get-credentials ai-system-cluster --zone us-central1-a

# Verify cluster
kubectl get nodes
```

#### On-Premises / Self-Managed

For on-premises deployment, ensure you have:
- Kubernetes cluster provisioned (using kubeadm, Rancher, or similar)
- Persistent storage configured (NFS, Ceph, or local storage)
- Load balancer solution (MetalLB for bare metal)
- Ingress controller installed

### 2. Install Required Kubernetes Components

#### Install Ingress NGINX

```bash
# Add Helm repository
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Install ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer \
  --set controller.metrics.enabled=true \
  --set controller.podAnnotations."prometheus\.io/scrape"=true \
  --set controller.podAnnotations."prometheus\.io/port"=10254

# Verify installation
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

#### Install cert-manager (for TLS certificates)

```bash
# Add Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

# Verify installation
kubectl get pods -n cert-manager
```

#### Install Metrics Server (for HPA)

```bash
# Install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify installation
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

#### Install Prometheus & Grafana (Monitoring)

```bash
# Add Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin123

# Verify installation
kubectl get pods -n monitoring
```

### 3. Network Configuration

#### Configure DNS

```bash
# Get LoadBalancer IP
kubectl get svc ingress-nginx-controller -n ingress-nginx

# Create DNS A records:
# ai-system.example.com -> LoadBalancer IP
# api.ai-system.example.com -> LoadBalancer IP
```

#### Configure Firewall Rules

For AWS:
```bash
# Allow HTTP/HTTPS traffic
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0
```

For GCP:
```bash
# Create firewall rule
gcloud compute firewall-rules create allow-http-https \
  --allow tcp:80,tcp:443 \
  --source-ranges 0.0.0.0/0 \
  --target-tags ai-system-cluster
```

---

## Secrets Management

### 1. Create Environment Variables File

Create a `.env` file with all required secrets:

```bash
# Create .env file
cat > .env << 'EOF'
# Database Credentials
POSTGRES_USER=ai_user
POSTGRES_PASSWORD=CHANGE_ME_strong_password_123
POSTGRES_DB=ai_system

# Redis Password
REDIS_PASSWORD=CHANGE_ME_redis_password_456

# RabbitMQ Credentials
RABBITMQ_DEFAULT_USER=admin
RABBITMQ_DEFAULT_PASS=CHANGE_ME_rabbitmq_password_789
RABBITMQ_ERLANG_COOKIE=CHANGE_ME_erlang_cookie_abc

# LLM API Keys
OPENAI_API_KEY=sk-your-openai-api-key-here
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key-here

# Qdrant Vector Database
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-qdrant-api-key

# JWT Secrets
JWT_SECRET=CHANGE_ME_jwt_secret_very_long_random_string
JWT_REFRESH_SECRET=CHANGE_ME_refresh_secret_different_long_string

# Session Secret
SESSION_SECRET=CHANGE_ME_session_secret_another_random_string

# Application Secrets
API_KEY=CHANGE_ME_api_key_for_service_auth
ENCRYPTION_KEY=CHANGE_ME_32_char_encryption_key_here

# SMTP Configuration (for email notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-specific-password

# Slack Webhook (optional)
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL

# Let's Encrypt Email
LETSENCRYPT_EMAIL=admin@example.com
EOF

# Secure the file
chmod 600 .env
```

### 2. Create Kubernetes Secrets

#### Development Environment

```bash
# Create namespace
kubectl create namespace ai-system-dev

# Create secrets from .env file
kubectl create secret generic ai-system-secrets \
  --from-env-file=.env \
  -n ai-system-dev

# Verify secret creation
kubectl get secrets -n ai-system-dev
kubectl describe secret ai-system-secrets -n ai-system-dev
```

#### Staging Environment

```bash
# Create namespace
kubectl create namespace ai-system-staging

# Create secrets (use staging-specific .env file)
kubectl create secret generic ai-system-secrets \
  --from-env-file=.env.staging \
  -n ai-system-staging
```

#### Production Environment (with Sealed Secrets)

```bash
# Install Sealed Secrets Controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Install kubeseal CLI
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar -xvzf kubeseal-0.24.0-linux-amd64.tar.gz
sudo mv kubeseal /usr/local/bin/

# Create sealed secret
kubectl create secret generic ai-system-secrets \
  --from-env-file=.env \
  --dry-run=client \
  -o yaml | kubeseal -o yaml > sealed-secrets.yaml

# Apply sealed secret
kubectl apply -f sealed-secrets.yaml -n ai-system-prod
```

#### Alternative: External Secrets Operator (Recommended for Production)

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets-system \
  --create-namespace

# For AWS Secrets Manager
cat > external-secret.yaml << 'EOF'
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: ai-system-prod
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ai-system-secrets
  namespace: ai-system-prod
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: ai-system-secrets
    creationPolicy: Owner
  data:
  - secretKey: POSTGRES_PASSWORD
    remoteRef:
      key: ai-system/postgres-password
  - secretKey: OPENAI_API_KEY
    remoteRef:
      key: ai-system/openai-api-key
  # Add more secrets as needed
EOF

kubectl apply -f external-secret.yaml
```

### 3. TLS Certificate Setup

```bash
# Create ClusterIssuer for Let's Encrypt
cat > letsencrypt-issuer.yaml << 'EOF'
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com  # Change this
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          class: nginx
EOF

kubectl apply -f letsencrypt-issuer.yaml
```

---

## Database Initialization

### 1. Deploy PostgreSQL

```bash
# Navigate to k8s directory
cd devops/k8s/base

# Deploy PostgreSQL StatefulSet
kubectl apply -f postgres-statefulset.yaml -n ai-system-dev
kubectl apply -f postgres-service.yaml -n ai-system-dev

# Wait for PostgreSQL to be ready
kubectl wait --for=condition=ready pod -l app=postgres -n ai-system-dev --timeout=300s

# Verify deployment
kubectl get pods -n ai-system-dev | grep postgres
kubectl get pvc -n ai-system-dev
```

### 2. Initialize Database Schema

```bash
# Get PostgreSQL pod name
POSTGRES_POD=$(kubectl get pods -n ai-system-dev -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Create databases
kubectl exec -n ai-system-dev $POSTGRES_POD -- psql -U ai_user -d postgres -c "CREATE DATABASE ai_system;"

# Initialize schema (if you have SQL files)
kubectl exec -n ai-system-dev $POSTGRES_POD -- psql -U ai_user -d ai_system < database/schema.sql

# Or connect interactively
kubectl exec -it -n ai-system-dev $POSTGRES_POD -- psql -U ai_user -d ai_system
```

Sample schema:
```sql
-- Users table
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'viewer',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Approvals table
CREATE TABLE approvals (
  id SERIAL PRIMARY KEY,
  request_type VARCHAR(50) NOT NULL,
  request_data JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  requested_by INTEGER REFERENCES users(id),
  approved_by INTEGER REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  approved_at TIMESTAMP
);

-- Audit logs table
CREATE TABLE audit_logs (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_id INTEGER,
  details JSONB,
  ip_address VARCHAR(45),
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes
CREATE INDEX idx_approvals_status ON approvals(status);
CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
```

### 3. Deploy Redis

```bash
# Deploy Redis
kubectl apply -f redis-deployment.yaml -n ai-system-dev
kubectl apply -f redis-service.yaml -n ai-system-dev

# Wait for Redis to be ready
kubectl wait --for=condition=ready pod -l app=redis -n ai-system-dev --timeout=300s

# Test Redis connection
REDIS_POD=$(kubectl get pods -n ai-system-dev -l app=redis -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n ai-system-dev $REDIS_POD -- redis-cli ping
# Should return: PONG
```

### 4. Deploy RabbitMQ

```bash
# Deploy RabbitMQ StatefulSet
kubectl apply -f rabbitmq-statefulset.yaml -n ai-system-dev
kubectl apply -f rabbitmq-service.yaml -n ai-system-dev

# Wait for RabbitMQ to be ready
kubectl wait --for=condition=ready pod -l app=rabbitmq -n ai-system-dev --timeout=300s

# Verify RabbitMQ
kubectl get pods -n ai-system-dev | grep rabbitmq

# Access RabbitMQ Management UI (port-forward)
kubectl port-forward -n ai-system-dev svc/rabbitmq 15672:15672
# Open browser: http://localhost:15672
# Login: admin / (password from secrets)
```

### 5. Setup Qdrant Vector Database

Option A: Use Qdrant Cloud (Recommended)
```bash
# Sign up at https://qdrant.io/
# Get your cluster URL and API key
# Add to secrets (already done in secrets section)
```

Option B: Self-hosted Qdrant
```bash
# Deploy Qdrant
cat > qdrant-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: qdrant
  namespace: ai-system-dev
spec:
  serviceName: qdrant
  replicas: 1
  selector:
    matchLabels:
      app: qdrant
  template:
    metadata:
      labels:
        app: qdrant
    spec:
      containers:
      - name: qdrant
        image: qdrant/qdrant:v1.7.0
        ports:
        - containerPort: 6333
          name: http
        - containerPort: 6334
          name: grpc
        volumeMounts:
        - name: qdrant-storage
          mountPath: /qdrant/storage
  volumeClaimTemplates:
  - metadata:
      name: qdrant-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: qdrant
  namespace: ai-system-dev
spec:
  ports:
  - port: 6333
    name: http
  - port: 6334
    name: grpc
  selector:
    app: qdrant
EOF

kubectl apply -f qdrant-deployment.yaml
```

---

## Service Deployment

### 1. Build and Push Container Images

```bash
# Set container registry
export REGISTRY="your-dockerhub-username"  # or your registry URL
export VERSION="1.0.0"

# Build LLM Service
cd devops/services/llm-service
docker build -t ${REGISTRY}/llm-service:${VERSION} .
docker push ${REGISTRY}/llm-service:${VERSION}

# Build Worker Service
cd ../worker-service
docker build -t ${REGISTRY}/worker-service:${VERSION} .
docker push ${REGISTRY}/worker-service:${VERSION}

# Build Approval Backend
cd ../approval-dashboard/backend
docker build -t ${REGISTRY}/approval-backend:${VERSION} .
docker push ${REGISTRY}/approval-backend:${VERSION}

# Build Approval Frontend
cd ../frontend
docker build -t ${REGISTRY}/approval-frontend:${VERSION} .
docker push ${REGISTRY}/approval-frontend:${VERSION}
```

### 2. Update Image Tags in Manifests

```bash
# Update image references in deployment files
cd devops/k8s/base

# Update llm-service-deployment.yaml
sed -i "s|image: llm-service:latest|image: ${REGISTRY}/llm-service:${VERSION}|g" llm-service-deployment.yaml

# Update worker-service-deployment.yaml
sed -i "s|image: worker-service:latest|image: ${REGISTRY}/worker-service:${VERSION}|g" worker-service-deployment.yaml

# Update approval-backend-deployment.yaml
sed -i "s|image: approval-backend:latest|image: ${REGISTRY}/approval-backend:${VERSION}|g" approval-backend-deployment.yaml

# Update approval-frontend-deployment.yaml
sed -i "s|image: approval-frontend:latest|image: ${REGISTRY}/approval-frontend:${VERSION}|g" approval-frontend-deployment.yaml
```

### 3. Deploy Base Resources

```bash
cd devops/k8s/base

# Deploy namespace
kubectl apply -f namespace.yaml

# Deploy ConfigMap
kubectl apply -f configmap.yaml -n ai-system-dev

# Deploy RBAC
kubectl apply -f rbac.yaml -n ai-system-dev

# Deploy Network Policies
kubectl apply -f networkpolicy.yaml -n ai-system-dev
```

### 4. Deploy Application Services

```bash
# Deploy LLM Service
kubectl apply -f llm-service-deployment.yaml -n ai-system-dev
kubectl apply -f llm-service-service.yaml -n ai-system-dev

# Deploy Worker Service
kubectl apply -f worker-service-deployment.yaml -n ai-system-dev
kubectl apply -f worker-service-service.yaml -n ai-system-dev

# Deploy Approval Backend
kubectl apply -f approval-backend-deployment.yaml -n ai-system-dev
kubectl apply -f approval-backend-service.yaml -n ai-system-dev

# Deploy Approval Frontend
kubectl apply -f approval-frontend-deployment.yaml -n ai-system-dev
kubectl apply -f approval-frontend-service.yaml -n ai-system-dev
```

### 5. Deploy HPA (Horizontal Pod Autoscaler)

```bash
# Deploy HPA for all services
kubectl apply -f hpa.yaml -n ai-system-dev

# Verify HPA
kubectl get hpa -n ai-system-dev
```

### 6. Deploy Ingress

```bash
# Update domain names in ingress.yaml
sed -i 's/ai-system.example.com/your-domain.com/g' ingress.yaml
sed -i 's/api.ai-system.example.com/api.your-domain.com/g' ingress.yaml

# Deploy Ingress
kubectl apply -f ingress.yaml -n ai-system-dev

# Verify Ingress
kubectl get ingress -n ai-system-dev
kubectl describe ingress ai-system-ingress -n ai-system-dev
```

---

## Environment-Specific Deployment

### Using Kustomize for Environment Management

#### Development Deployment

```bash
cd devops/k8s/overlays/dev

# Review what will be applied
kubectl kustomize .

# Apply development configuration
kubectl apply -k .

# Verify deployment
kubectl get all -n ai-system-dev
```

#### Staging Deployment

```bash
cd devops/k8s/overlays/staging

# Create staging namespace
kubectl create namespace ai-system-staging

# Create staging secrets
kubectl create secret generic ai-system-secrets \
  --from-env-file=.env.staging \
  -n ai-system-staging

# Apply staging configuration
kubectl apply -k .

# Verify deployment
kubectl get all -n ai-system-staging
```

#### Production Deployment

```bash
cd devops/k8s/overlays/prod

# Create production namespace
kubectl create namespace ai-system-prod

# Setup production secrets (using Sealed Secrets or External Secrets)
# See Secrets Management section above

# Apply production configuration
kubectl apply -k .

# Verify deployment
kubectl get all -n ai-system-prod

# Check HPA status
kubectl get hpa -n ai-system-prod

# Check certificate status
kubectl get certificate -n ai-system-prod
```

### Using Helm for Deployment

```bash
cd devops/infrastructure/helm-charts/ai-system

# Install for development
helm install ai-system . \
  --namespace ai-system-dev \
  --create-namespace \
  --values values.yaml \
  --values values-dev.yaml

# Install for staging
helm install ai-system . \
  --namespace ai-system-staging \
  --create-namespace \
  --values values.yaml \
  --values values-staging.yaml

# Install for production
helm install ai-system . \
  --namespace ai-system-prod \
  --create-namespace \
  --values values.yaml \
  --values values-prod.yaml

# Verify installation
helm list --all-namespaces
```

---

## Post-Deployment Verification

### 1. Check Pod Status

```bash
# Check all pods in namespace
kubectl get pods -n ai-system-dev

# Expected output:
# NAME                                READY   STATUS    RESTARTS   AGE
# llm-service-xxxxx                   1/1     Running   0          2m
# worker-service-xxxxx                1/1     Running   0          2m
# approval-backend-xxxxx              1/1     Running   0          2m
# approval-frontend-xxxxx             1/1     Running   0          2m
# postgres-0                          1/1     Running   0          5m
# redis-xxxxx                         1/1     Running   0          5m
# rabbitmq-0                          1/1     Running   0          5m

# Check pod details if any issues
kubectl describe pod <pod-name> -n ai-system-dev

# Check pod logs
kubectl logs <pod-name> -n ai-system-dev
kubectl logs <pod-name> -n ai-system-dev --previous  # Previous container logs
```

### 2. Test Service Connectivity

```bash
# Test LLM Service health
kubectl port-forward -n ai-system-dev svc/llm-service 8000:8000 &
curl http://localhost:8000/health
# Expected: {"status":"healthy","service":"llm-service",...}

# Test Approval Backend
kubectl port-forward -n ai-system-dev svc/approval-backend 3000:3000 &
curl http://localhost:3000/health
# Expected: {"status":"ok"}

# Test database connectivity from pod
kubectl exec -n ai-system-dev <llm-service-pod> -- nc -zv postgres 5432
# Expected: postgres (10.x.x.x:5432) open

# Test Redis connectivity
kubectl exec -n ai-system-dev <llm-service-pod> -- nc -zv redis 6379
# Expected: redis (10.x.x.x:6379) open
```

### 3. Test API Endpoints

```bash
# Get Ingress IP/hostname
kubectl get ingress -n ai-system-dev

# Test LLM Service API
curl -X POST https://api.your-domain.com/llm/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello, how are you?"}
    ]
  }'

# Test approval backend login
curl -X POST https://api.your-domain.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "admin123"
  }'

# Access frontend
curl https://your-domain.com
# Should return HTML
```

### 4. Verify Monitoring

```bash
# Port-forward to Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
# Open browser: http://localhost:9090

# Port-forward to Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80 &
# Open browser: http://localhost:3000
# Login: admin / admin123

# Check if metrics are being collected
curl http://localhost:9090/api/v1/query?query=up
```

### 5. Verify Auto-Scaling

```bash
# Check HPA status
kubectl get hpa -n ai-system-dev

# Generate load to test scaling (optional)
kubectl run -n ai-system-dev load-generator \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://llm-service:8000/health; done"

# Watch HPA scaling
kubectl get hpa -n ai-system-dev --watch

# Clean up load generator
kubectl delete pod load-generator -n ai-system-dev
```

### 6. Create Initial Admin User

```bash
# Connect to PostgreSQL
POSTGRES_POD=$(kubectl get pods -n ai-system-dev -l app=postgres -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it -n ai-system-dev $POSTGRES_POD -- psql -U ai_user -d ai_system

# Create admin user (password should be hashed in real deployment)
INSERT INTO users (username, email, password_hash, role)
VALUES ('admin', 'admin@example.com', '$2b$10$hashed_password_here', 'admin');

# Exit psql
\q
```

---

## Rollback Procedures

### 1. Rollback with kubectl

```bash
# View deployment history
kubectl rollout history deployment/llm-service -n ai-system-dev

# Rollback to previous version
kubectl rollout undo deployment/llm-service -n ai-system-dev

# Rollback to specific revision
kubectl rollout undo deployment/llm-service -n ai-system-dev --to-revision=2

# Check rollback status
kubectl rollout status deployment/llm-service -n ai-system-dev
```

### 2. Rollback with Helm

```bash
# View release history
helm history ai-system -n ai-system-dev

# Rollback to previous version
helm rollback ai-system -n ai-system-dev

# Rollback to specific revision
helm rollback ai-system 2 -n ai-system-dev

# Verify rollback
helm status ai-system -n ai-system-dev
```

### 3. Database Rollback

```bash
# Restore from backup
POSTGRES_POD=$(kubectl get pods -n ai-system-dev -l app=postgres -o jsonpath='{.items[0].metadata.name}')

# Copy backup to pod
kubectl cp backup.sql ai-system-dev/$POSTGRES_POD:/tmp/backup.sql

# Restore database
kubectl exec -n ai-system-dev $POSTGRES_POD -- psql -U ai_user -d ai_system < /tmp/backup.sql

# Or use pg_restore for binary backups
kubectl exec -n ai-system-dev $POSTGRES_POD -- pg_restore -U ai_user -d ai_system /tmp/backup.dump
```

### 4. Emergency Rollback Procedure

```bash
# 1. Scale down problematic service
kubectl scale deployment/llm-service --replicas=0 -n ai-system-dev

# 2. Restore previous version from backup
kubectl apply -f backup/llm-service-deployment-v1.0.0.yaml -n ai-system-dev

# 3. Scale up
kubectl scale deployment/llm-service --replicas=3 -n ai-system-dev

# 4. Verify health
kubectl get pods -n ai-system-dev -l app=llm-service
kubectl logs -f deployment/llm-service -n ai-system-dev

# 5. Test endpoints
curl http://llm-service.ai-system-dev.svc.cluster.local:8000/health
```

---

## Troubleshooting

### Common Issues and Solutions

#### 1. Pods Not Starting

**Symptom**: Pods stuck in `Pending`, `CrashLoopBackOff`, or `ImagePullBackOff` state.

**Diagnosis**:
```bash
kubectl describe pod <pod-name> -n ai-system-dev
kubectl logs <pod-name> -n ai-system-dev
kubectl get events -n ai-system-dev --sort-by='.lastTimestamp'
```

**Solutions**:

**ImagePullBackOff**:
```bash
# Check image name and tag
kubectl get deployment <deployment-name> -n ai-system-dev -o yaml | grep image:

# Create image pull secret if using private registry
kubectl create secret docker-registry regcred \
  --docker-server=<registry-url> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n ai-system-dev

# Add to deployment
kubectl patch deployment <deployment-name> -n ai-system-dev \
  -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"regcred"}]}}}}'
```

**CrashLoopBackOff**:
```bash
# Check logs
kubectl logs <pod-name> -n ai-system-dev --previous

# Common causes:
# - Missing environment variables
# - Cannot connect to database
# - Invalid configuration

# Verify secrets
kubectl get secret ai-system-secrets -n ai-system-dev -o yaml

# Verify ConfigMap
kubectl get configmap ai-system-config -n ai-system-dev -o yaml
```

**Pending (Insufficient Resources)**:
```bash
# Check node resources
kubectl describe nodes

# Check pod resource requests
kubectl get pod <pod-name> -n ai-system-dev -o yaml | grep -A 5 resources:

# Solution: Scale up cluster or reduce resource requests
```

#### 2. Service Not Accessible

**Symptom**: Cannot access service through Ingress or port-forward.

**Diagnosis**:
```bash
# Check service endpoints
kubectl get endpoints -n ai-system-dev

# Check service selector matches pod labels
kubectl get svc <service-name> -n ai-system-dev -o yaml | grep selector:
kubectl get pods -n ai-system-dev --show-labels

# Check Ingress status
kubectl describe ingress ai-system-ingress -n ai-system-dev

# Check Ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/component=controller
```

**Solutions**:

```bash
# Verify service is routing to pods
kubectl port-forward -n ai-system-dev svc/<service-name> 8080:8000
curl http://localhost:8080/health

# Check network policies
kubectl get networkpolicy -n ai-system-dev
kubectl describe networkpolicy <policy-name> -n ai-system-dev

# Temporarily disable network policy for testing
kubectl delete networkpolicy --all -n ai-system-dev  # ONLY FOR TESTING
```

#### 3. Database Connection Issues

**Symptom**: Services cannot connect to PostgreSQL, Redis, or RabbitMQ.

**Diagnosis**:
```bash
# Test connectivity from pod
kubectl exec -n ai-system-dev <pod-name> -- nc -zv postgres 5432
kubectl exec -n ai-system-dev <pod-name> -- nc -zv redis 6379
kubectl exec -n ai-system-dev <pod-name> -- nc -zv rabbitmq 5672

# Check database pod status
kubectl get pods -n ai-system-dev | grep -E 'postgres|redis|rabbitmq'

# Check database logs
kubectl logs -n ai-system-dev postgres-0
kubectl logs -n ai-system-dev rabbitmq-0
```

**Solutions**:

```bash
# Verify secrets are correct
kubectl get secret ai-system-secrets -n ai-system-dev -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 -d

# Restart database pods
kubectl delete pod postgres-0 -n ai-system-dev  # Will be recreated by StatefulSet

# Check service DNS resolution
kubectl exec -n ai-system-dev <pod-name> -- nslookup postgres
kubectl exec -n ai-system-dev <pod-name> -- nslookup redis
```

#### 4. TLS Certificate Issues

**Symptom**: HTTPS not working or certificate errors.

**Diagnosis**:
```bash
# Check cert-manager logs
kubectl logs -n cert-manager -l app=cert-manager

# Check certificate status
kubectl get certificate -n ai-system-dev
kubectl describe certificate ai-system-tls -n ai-system-dev

# Check certificate request
kubectl get certificaterequest -n ai-system-dev
kubectl describe certificaterequest <request-name> -n ai-system-dev
```

**Solutions**:

```bash
# Delete and recreate certificate
kubectl delete certificate ai-system-tls -n ai-system-dev
kubectl apply -f ingress.yaml -n ai-system-dev

# Check ClusterIssuer
kubectl get clusterissuer
kubectl describe clusterissuer letsencrypt-prod

# Manual certificate debugging
kubectl get secret ai-system-tls -n ai-system-dev -o yaml
```

#### 5. High Resource Usage / OOMKilled

**Symptom**: Pods being killed due to out-of-memory errors.

**Diagnosis**:
```bash
# Check resource usage
kubectl top pods -n ai-system-dev
kubectl top nodes

# Check pod events
kubectl get events -n ai-system-dev | grep OOMKilled

# Check resource limits
kubectl get pod <pod-name> -n ai-system-dev -o yaml | grep -A 10 resources:
```

**Solutions**:

```bash
# Increase memory limits
kubectl set resources deployment/<deployment-name> \
  -n ai-system-dev \
  --limits=memory=4Gi \
  --requests=memory=2Gi

# Or edit deployment directly
kubectl edit deployment <deployment-name> -n ai-system-dev

# Scale up nodes if cluster-wide issue
# AWS EKS:
eksctl scale nodegroup --cluster=ai-system-cluster --name=standard-workers --nodes=5

# GKE:
gcloud container clusters resize ai-system-cluster --num-nodes=5 --zone=us-central1-a
```

#### 6. Monitoring Not Working

**Symptom**: Prometheus not collecting metrics, Grafana dashboards empty.

**Diagnosis**:
```bash
# Check Prometheus targets
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
curl http://localhost:9090/api/v1/targets

# Check ServiceMonitor
kubectl get servicemonitor -n ai-system-dev

# Check if services have proper annotations
kubectl get svc -n ai-system-dev -o yaml | grep prometheus
```

**Solutions**:

```bash
# Add prometheus annotations to services
kubectl annotate svc llm-service -n ai-system-dev \
  prometheus.io/scrape=true \
  prometheus.io/port=8000 \
  prometheus.io/path=/metrics

# Restart Prometheus
kubectl delete pod -n monitoring -l app.kubernetes.io/name=prometheus
```

### Debug Commands Cheatsheet

```bash
# Pod debugging
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> [-c <container-name>]
kubectl logs <pod-name> -n <namespace> --previous
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Service debugging
kubectl get svc -n <namespace>
kubectl get endpoints -n <namespace>
kubectl describe svc <service-name> -n <namespace>

# Network debugging
kubectl exec -n <namespace> <pod-name> -- nc -zv <host> <port>
kubectl exec -n <namespace> <pod-name> -- nslookup <service-name>
kubectl exec -n <namespace> <pod-name> -- curl <url>

# Resource debugging
kubectl top nodes
kubectl top pods -n <namespace>
kubectl describe node <node-name>

# Events debugging
kubectl get events -n <namespace> --sort-by='.lastTimestamp'
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Configuration debugging
kubectl get configmap -n <namespace>
kubectl get secret -n <namespace>
kubectl describe configmap <name> -n <namespace>
kubectl get secret <name> -n <namespace> -o yaml
```

### Getting Help

If you continue to experience issues:

1. **Check logs**: Always start with pod logs and events
2. **Review documentation**: Check official Kubernetes and service-specific docs
3. **Community support**: Ask in Kubernetes Slack or Stack Overflow
4. **Create an issue**: If it's a bug, create an issue in the project repository

---

## Conclusion

This deployment guide covers the complete process of deploying the AI-Driven Hybrid Kubernetes System from infrastructure setup to troubleshooting. Follow the steps in order for a successful deployment.

Key points to remember:
- Always secure your secrets properly
- Test in dev/staging before production
- Monitor your deployments closely
- Have rollback procedures ready
- Document any custom configurations

For ongoing operations and maintenance, refer to the OPERATIONS.md guide.
