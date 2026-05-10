# [CLOUD_PROVIDER] Container Apps Deployment Guide - DevOps AI System

## Budget: $100 | Estimated Cost: $51-66/month

This guide will deploy your entire AI-powered DevOps system (8 services + infrastructure) to [CLOUD_PROVIDER] Container Apps without requiring local Docker Desktop.

---

## Prerequisites

1. [CLOUD_PROVIDER] Account with $100 credits
2. [CLOUD_PROVIDER] CLI installed (or use [CLOUD_PROVIDER] Cloud Shell)
3. Access to [CLOUD_PROVIDER] Cloud Shell (browser-based, comes with Docker pre-installed)

---

## Architecture Overview

### Services to Deploy (8 Container Apps):
1. **monitoring-agent** - K8s/Prometheus monitoring
2. **analyzer-agent** - LLM-powered analysis
3. **auto-response-agent** - Autonomous remediation
4. **notifier-agent** - Slack notifications
5. **memory-agent** - Incident storage & RAG
6. **llm-service** - Multi-provider LLM API
7. **approval-dashboard-backend** - REST API
8. **approval-dashboard-frontend** - React UI

### Infrastructure Components:
- **[CLOUD_PROVIDER] Container Registry** (ACR) - Image storage
- **[CLOUD_PROVIDER] Database for PostgreSQL** - Persistent storage
- **[CLOUD_PROVIDER] Cache for Redis** - Caching layer
- **RabbitMQ** (as Container App) - Message broker
- **Qdrant Cloud** - Vector database (external, free tier)

---

## Phase 1: Initial Setup & Resource Group Creation

### Step 1.1: Login and Set Subscription

```bash
# Login to [CLOUD_PROVIDER]
az login

# List subscriptions
az account list --output table

# Set the subscription with your $100 credits
az account set --subscription "YOUR_SUBSCRIPTION_ID"
```

### Step 1.2: Create Resource Group

```bash
# Create resource group in East US (cost-efficient region)
az group create \
  --name devops-ai-test-rg \
  --location eastus
```

### Step 1.3: Register Required Providers

```bash
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

---

## Phase 2: Container Registry Setup

### Step 2.1: Create [CLOUD_PROVIDER] Container Registry

```bash
# Create ACR (Basic SKU - $5/month)
az acr create \
  --resource-group devops-ai-test-rg \
  --name devopsairegistry \
  --sku Basic \
  --admin-enabled true

# Get ACR login credentials
az acr credential show \
  --name devopsairegistry \
  --resource-group devops-ai-test-rg

# Save these credentials for later:
# USERNAME: devopsairegistry
# PASSWORD: <output from command above>
```

### Step 2.2: Login to ACR

```bash
az acr login --name devopsairegistry
```

---

## Phase 3: Build Docker Images in [CLOUD_PROVIDER] (No Local Docker Required)

**IMPORTANT**: We'll use [CLOUD_PROVIDER] Container Registry Tasks to build images in the cloud.

### Step 3.1: Upload Project to [CLOUD_PROVIDER] Cloud Shell

Option A: Use [CLOUD_PROVIDER] Cloud Shell's file upload feature
Option B: Clone from Git repository

```bash
# If using Git (recommended)
cd ~
git clone <YOUR_REPO_URL>
cd <PROJECT_DIR>

# Navigate to devops folder
cd devops
```

### Step 3.2: Build Images Using ACR Tasks

```bash
# Build monitoring-agent
az acr build \
  --registry devopsairegistry \
  --image monitoring-agent:latest \
  --file services/monitoring-agent/Dockerfile \
  services/monitoring-agent

# Build analyzer-agent
az acr build \
  --registry devopsairegistry \
  --image analyzer-agent:latest \
  --file services/analyzer-agent/Dockerfile \
  services/analyzer-agent

# Build auto-response-agent
az acr build \
  --registry devopsairegistry \
  --image auto-response-agent:latest \
  --file services/auto-response-agent/Dockerfile \
  services/auto-response-agent

# Build notifier-agent
az acr build \
  --registry devopsairegistry \
  --image notifier-agent:latest \
  --file services/notifier-agent/Dockerfile \
  services/notifier-agent

# Build memory-agent
az acr build \
  --registry devopsairegistry \
  --image memory-agent:latest \
  --file services/memory-agent/Dockerfile \
  services/memory-agent

# Build llm-service
az acr build \
  --registry devopsairegistry \
  --image llm-service:latest \
  --file services/llm-service/Dockerfile \
  services/llm-service

# Build approval-dashboard-backend
az acr build \
  --registry devopsairegistry \
  --image approval-dashboard-backend:latest \
  --file services/approval-dashboard/backend/Dockerfile \
  services/approval-dashboard/backend

# Build approval-dashboard-frontend
az acr build \
  --registry devopsairegistry \
  --image approval-dashboard-frontend:latest \
  --file services/approval-dashboard/frontend/Dockerfile \
  services/approval-dashboard/frontend

# Build RabbitMQ (using official image, but tagging for tracking)
docker pull rabbitmq:3-management
docker tag rabbitmq:3-management devopsairegistry.azurecr.io/rabbitmq:3-management
az acr import \
  --name devopsairegistry \
  --source docker.io/rabbitmq:3-management \
  --image rabbitmq:3-management
```

### Step 3.3: Verify Images

```bash
az acr repository list --name devopsairegistry --output table
```

---

## Phase 4: Infrastructure Services Setup

### Step 4.1: Create PostgreSQL Database

```bash
# Create PostgreSQL Flexible Server (B1ms - $15/month)
az postgres flexible-server create \
  --resource-group devops-ai-test-rg \
  --name devops-ai-postgres \
  --location eastus \
  --admin-user dbadmin \
  --admin-password "YourSecurePassword123!" \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --version 14 \
  --storage-size 32 \
  --public-access 0.0.0.0

# Create databases
az postgres flexible-server db create \
  --resource-group devops-ai-test-rg \
  --server-name devops-ai-postgres \
  --database-name devops

az postgres flexible-server db create \
  --resource-group devops-ai-test-rg \
  --server-name devops-ai-postgres \
  --database-name approvals

# Allow [CLOUD_PROVIDER] services to access
az postgres flexible-server firewall-rule create \
  --resource-group devops-ai-test-rg \
  --name devops-ai-postgres \
  --rule-name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Get connection string
echo "PostgreSQL Connection String:"
echo "postgresql://dbadmin:YourSecurePassword123!@devops-ai-postgres.postgres.database.azure.com:5432/devops?sslmode=require"
```

### Step 4.2: Create Redis Cache

```bash
# Create Redis (Basic C0 - $16/month)
az redis create \
  --resource-group devops-ai-test-rg \
  --name devops-ai-redis \
  --location eastus \
  --sku Basic \
  --vm-size c0

# Get Redis connection details
az redis show \
  --resource-group devops-ai-test-rg \
  --name devops-ai-redis

az redis list-keys \
  --resource-group devops-ai-test-rg \
  --name devops-ai-redis

# Save the hostname and primary key for later
```

### Step 4.3: Setup Qdrant Cloud (External)

```bash
# Go to https://cloud.qdrant.io/
# Sign up for free tier (1GB cluster)
# Create a cluster and get:
# - QDRANT_URL: https://your-cluster.qdrant.io
# - QDRANT_API_KEY: your-api-key
```

---

## Phase 5: Container Apps Environment Setup

### Step 5.1: Create Log Analytics Workspace

```bash
az monitor log-analytics workspace create \
  --resource-group devops-ai-test-rg \
  --workspace-name devops-ai-logs \
  --location eastus

# Get workspace ID and key
LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group devops-ai-test-rg \
  --workspace-name devops-ai-logs \
  --query customerId \
  --output tsv)

LOG_ANALYTICS_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group devops-ai-test-rg \
  --workspace-name devops-ai-logs \
  --query primarySharedKey \
  --output tsv)
```

### Step 5.2: Create Container Apps Environment

```bash
az containerapp env create \
  --name devops-ai-env \
  --resource-group devops-ai-test-rg \
  --location eastus \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_ID \
  --logs-workspace-key $LOG_ANALYTICS_KEY
```

---

## Phase 6: Deploy RabbitMQ (Message Broker)

```bash
az containerapp create \
  --name rabbitmq \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/rabbitmq:3-management \
  --target-port 5672 \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    RABBITMQ_DEFAULT_USER=devops \
    RABBITMQ_DEFAULT_PASS=devops123 \
    RABBITMQ_DEFAULT_VHOST=devops \
  --cpu 0.5 \
  --memory 1.0Gi \
  --min-replicas 1 \
  --max-replicas 1

# Get RabbitMQ internal FQDN
az containerapp show \
  --name rabbitmq \
  --resource-group devops-ai-test-rg \
  --query properties.configuration.ingress.fqdn \
  --output tsv

# Save as: RABBITMQ_HOST (e.g., rabbitmq.internal.xxx.eastus.azurecontainerapps.io)
```

---

## Phase 7: Deploy Core Services

### Environment Variables Preparation

Create a file `env-vars.txt` with your actual credentials:

```bash
# Database
DATABASE_URL=postgresql://dbadmin:YourSecurePassword123!@devops-ai-postgres.postgres.database.azure.com:5432/devops?sslmode=require
POSTGRES_HOST=devops-ai-postgres.postgres.database.azure.com
POSTGRES_PORT=5432
POSTGRES_DB=devops
POSTGRES_USER=dbadmin
POSTGRES_PASSWORD=YourSecurePassword123!

# Redis
REDIS_HOST=devops-ai-redis.redis.cache.windows.net
REDIS_PORT=6380
REDIS_PASSWORD=<YOUR_REDIS_PRIMARY_KEY>
REDIS_SSL=true

# RabbitMQ
RABBITMQ_HOST=rabbitmq.internal.xxx.eastus.azurecontainerapps.io
RABBITMQ_PORT=5672
RABBITMQ_USER=devops
RABBITMQ_PASSWORD=devops123
RABBITMQ_VHOST=devops

# Qdrant
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-qdrant-api-key
QDRANT_COLLECTION_NAME=devops_incidents

# LLM API Keys
OPENAI_API_KEY=sk-your-openai-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key
GEMINI_API_KEY=your-gemini-key
LLM_PROVIDER=openai

# Slack
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SLACK_TOKEN=xoxb-your-slack-token
SLACK_CHANNEL=#devops-alerts

# Kubernetes (for monitoring agent - optional for testing)
K8S_IN_CLUSTER=false
KUBECONFIG_PATH=/config/kubeconfig
```

### Step 7.1: Deploy LLM Service (Foundation)

```bash
az containerapp create \
  --name llm-service \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/llm-service:latest \
  --target-port 8000 \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    OPENAI_API_KEY="<YOUR_OPENAI_KEY>" \
    ANTHROPIC_API_KEY="<YOUR_ANTHROPIC_KEY>" \
    GEMINI_API_KEY="<YOUR_GEMINI_KEY>" \
    LLM_PROVIDER=openai \
    REDIS_HOST=devops-ai-redis.redis.cache.windows.net \
    REDIS_PORT=6380 \
    REDIS_PASSWORD="<REDIS_KEY>" \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 2
```

### Step 7.2: Deploy Memory Agent

```bash
az containerapp create \
  --name memory-agent \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/memory-agent:latest \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    POSTGRES_HOST=devops-ai-postgres.postgres.database.azure.com \
    POSTGRES_PORT=5432 \
    POSTGRES_DB=devops \
    POSTGRES_USER=dbadmin \
    POSTGRES_PASSWORD="YourSecurePassword123!" \
    RABBITMQ_HOST=<RABBITMQ_FQDN> \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    QDRANT_URL="<QDRANT_URL>" \
    QDRANT_API_KEY="<QDRANT_KEY>" \
    QDRANT_COLLECTION_NAME=devops_incidents \
    LLM_SERVICE_URL=http://llm-service.internal.xxx.eastus.azurecontainerapps.io \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1
```

### Step 7.3: Deploy Monitoring Agent

```bash
az containerapp create \
  --name monitoring-agent \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/monitoring-agent:latest \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    RABBITMQ_HOST=<RABBITMQ_FQDN> \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    PROMETHEUS_URL=http://localhost:9090 \
    K8S_IN_CLUSTER=false \
    ANOMALY_DETECTION_SENSITIVITY=2.0 \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1
```

### Step 7.4: Deploy Analyzer Agent

```bash
az containerapp create \
  --name analyzer-agent \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/analyzer-agent:latest \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    RABBITMQ_HOST=<RABBITMQ_FQDN> \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    LLM_SERVICE_URL=http://llm-service.internal.xxx.eastus.azurecontainerapps.io \
    REDIS_HOST=devops-ai-redis.redis.cache.windows.net \
    REDIS_PORT=6380 \
    REDIS_PASSWORD="<REDIS_KEY>" \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 2
```

### Step 7.5: Deploy Auto-Response Agent

```bash
az containerapp create \
  --name auto-response-agent \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/auto-response-agent:latest \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    RABBITMQ_HOST=<RABBITMQ_FQDN> \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    APPROVAL_API_URL=http://approval-dashboard-backend.internal.xxx.eastus.azurecontainerapps.io \
    K8S_IN_CLUSTER=false \
    AUTO_EXECUTE_THRESHOLD=0.9 \
    REQUIRES_APPROVAL_THRESHOLD=0.7 \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1
```

### Step 7.6: Deploy Notifier Agent

```bash
az containerapp create \
  --name notifier-agent \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/notifier-agent:latest \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    RABBITMQ_HOST=<RABBITMQ_FQDN> \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    SLACK_WEBHOOK_URL="<YOUR_SLACK_WEBHOOK>" \
    SLACK_TOKEN="<YOUR_SLACK_TOKEN>" \
    SLACK_CHANNEL="#devops-alerts" \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1
```

### Step 7.7: Deploy Approval Dashboard Backend

```bash
az containerapp create \
  --name approval-dashboard-backend \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/approval-dashboard-backend:latest \
  --target-port 3000 \
  --ingress internal \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    DATABASE_URL="postgresql://dbadmin:YourSecurePassword123!@devops-ai-postgres.postgres.database.azure.com:5432/approvals?sslmode=require" \
    JWT_SECRET="your-jwt-secret-change-this" \
    RABBITMQ_HOST=<RABBITMQ_FQDN> \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    NODE_ENV=production \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1
```

### Step 7.8: Deploy Approval Dashboard Frontend (Public Access)

```bash
az containerapp create \
  --name approval-dashboard-frontend \
  --resource-group devops-ai-test-rg \
  --environment devops-ai-env \
  --image devopsairegistry.azurecr.io/approval-dashboard-frontend:latest \
  --target-port 80 \
  --ingress external \
  --registry-server devopsairegistry.azurecr.io \
  --registry-username devopsairegistry \
  --registry-password "<ACR_PASSWORD>" \
  --env-vars \
    REACT_APP_API_URL=http://approval-dashboard-backend.internal.xxx.eastus.azurecontainerapps.io \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 2

# Get the public URL
az containerapp show \
  --name approval-dashboard-frontend \
  --resource-group devops-ai-test-rg \
  --query properties.configuration.ingress.fqdn \
  --output tsv
```

---

## Phase 8: Verification & Testing

### Step 8.1: Check All Services Status

```bash
# List all container apps
az containerapp list \
  --resource-group devops-ai-test-rg \
  --output table

# Check individual service health
az containerapp show \
  --name monitoring-agent \
  --resource-group devops-ai-test-rg \
  --query properties.runningStatus

# Repeat for: analyzer-agent, auto-response-agent, notifier-agent, memory-agent, llm-service, approval-dashboard-backend, approval-dashboard-frontend, rabbitmq
```

### Step 8.2: View Logs

```bash
# View logs for monitoring-agent
az containerapp logs show \
  --name monitoring-agent \
  --resource-group devops-ai-test-rg \
  --follow

# Repeat for other services as needed
```

### Step 8.3: Test Service Communication

```bash
# Exec into llm-service to test connectivity
az containerapp exec \
  --name llm-service \
  --resource-group devops-ai-test-rg \
  --command "/bin/sh"

# Inside the container:
# - Test RabbitMQ: telnet <RABBITMQ_HOST> 5672
# - Test PostgreSQL: psql postgresql://...
# - Test Redis: redis-cli -h <REDIS_HOST> -p 6380 --tls ping
```

### Step 8.4: Test LLM Service

```bash
# Get llm-service internal URL
LLM_URL=$(az containerapp show \
  --name llm-service \
  --resource-group devops-ai-test-rg \
  --query properties.configuration.ingress.fqdn \
  --output tsv)

# Test health endpoint (from [CLOUD_PROVIDER] Cloud Shell)
curl http://$LLM_URL/health
curl http://$LLM_URL/ready
```

### Step 8.5: Access Approval Dashboard

```bash
# Get frontend URL
FRONTEND_URL=$(az containerapp show \
  --name approval-dashboard-frontend \
  --resource-group devops-ai-test-rg \
  --query properties.configuration.ingress.fqdn \
  --output tsv)

echo "Approval Dashboard URL: https://$FRONTEND_URL"
```

### Step 8.6: Monitor Costs

```bash
# Check current cost
az consumption usage list \
  --start-date 2025-01-01 \
  --end-date 2025-12-31 \
  --query "[?contains(instanceName, 'devops-ai')].{Name:instanceName, Cost:pretaxCost}" \
  --output table

# Set budget alert
az consumption budget create \
  --resource-group devops-ai-test-rg \
  --budget-name devops-ai-budget \
  --amount 100 \
  --time-grain Monthly \
  --start-date 2025-01-01 \
  --end-date 2025-12-31
```

---

## Phase 9: Functional Testing Checklist

### Test 1: End-to-End Flow
1. Monitoring agent detects anomaly → publishes to RabbitMQ
2. Analyzer agent consumes event → analyzes with LLM → searches RAG
3. Auto-response agent receives recommendation → validates → executes or requests approval
4. Memory agent stores incident → creates embedding in Qdrant
5. Notifier agent sends Slack notification

### Test 2: Approval Flow
1. Navigate to approval dashboard
2. Create a test approval request
3. Verify WebSocket real-time update
4. Approve/reject action
5. Check action execution logs

### Test 3: RAG Search
1. Store multiple incidents with memory-agent
2. Query similar incidents via analyzer-agent
3. Verify Qdrant returns relevant results

---

## Phase 10: Cleanup & Resource Deletion

### Option A: Delete Entire Resource Group (Recommended)

```bash
# Delete everything at once
az group delete \
  --name devops-ai-test-rg \
  --yes \
  --no-wait

# Verify deletion
az group exists --name devops-ai-test-rg
```

### Option B: Delete Individual Resources (Selective)

```bash
# Delete Container Apps
az containerapp delete --name monitoring-agent --resource-group devops-ai-test-rg --yes
az containerapp delete --name analyzer-agent --resource-group devops-ai-test-rg --yes
az containerapp delete --name auto-response-agent --resource-group devops-ai-test-rg --yes
az containerapp delete --name notifier-agent --resource-group devops-ai-test-rg --yes
az containerapp delete --name memory-agent --resource-group devops-ai-test-rg --yes
az containerapp delete --name llm-service --resource-group devops-ai-test-rg --yes
az containerapp delete --name approval-dashboard-backend --resource-group devops-ai-test-rg --yes
az containerapp delete --name approval-dashboard-frontend --resource-group devops-ai-test-rg --yes
az containerapp delete --name rabbitmq --resource-group devops-ai-test-rg --yes

# Delete Container Apps Environment
az containerapp env delete --name devops-ai-env --resource-group devops-ai-test-rg --yes

# Delete PostgreSQL
az postgres flexible-server delete --name devops-ai-postgres --resource-group devops-ai-test-rg --yes

# Delete Redis
az redis delete --name devops-ai-redis --resource-group devops-ai-test-rg --yes

# Delete ACR
az acr delete --name devopsairegistry --resource-group devops-ai-test-rg --yes

# Delete Log Analytics
az monitor log-analytics workspace delete --workspace-name devops-ai-logs --resource-group devops-ai-test-rg --yes

# Finally delete resource group
az group delete --name devops-ai-test-rg --yes
```

### Verify All Costs Stopped

```bash
# Check remaining resources
az resource list --resource-group devops-ai-test-rg

# Check billing
az consumption usage list --output table
```

---

## Cost Optimization Tips

1. **Scale to Zero**: Container Apps can scale to 0 replicas when not in use
2. **Stop Services When Not Testing**: Use `az containerapp update --min-replicas 0`
3. **Delete When Not Needed**: Don't leave resources running overnight
4. **Use Spot Instances**: For AKS (if you switch), use spot node pools
5. **Monitor Daily**: Check [CLOUD_PROVIDER] portal > Cost Management daily

---

## Troubleshooting

### Issue: Container App Won't Start

```bash
# Check logs
az containerapp logs show --name <APP_NAME> --resource-group devops-ai-test-rg --tail 100

# Check environment variables
az containerapp show --name <APP_NAME> --resource-group devops-ai-test-rg --query properties.template.containers[0].env
```

### Issue: Services Can't Communicate

```bash
# Verify all services are in same Container Apps Environment
az containerapp list --resource-group devops-ai-test-rg --query "[].{Name:name, Env:properties.environmentId}" --output table

# Check if ingress is internal vs external
az containerapp show --name <APP_NAME> --resource-group devops-ai-test-rg --query properties.configuration.ingress
```

### Issue: Database Connection Fails

```bash
# Check firewall rules
az postgres flexible-server firewall-rule list --resource-group devops-ai-test-rg --name devops-ai-postgres

# Test connection from Cloud Shell
psql "postgresql://dbadmin:YourSecurePassword123!@devops-ai-postgres.postgres.database.azure.com:5432/devops?sslmode=require"
```

### Issue: RabbitMQ Connection Fails

```bash
# Check RabbitMQ logs
az containerapp logs show --name rabbitmq --resource-group devops-ai-test-rg --tail 50

# Verify environment variables
az containerapp show --name rabbitmq --resource-group devops-ai-test-rg --query properties.template.containers[0].env
```

---

## Next Steps After Deployment

1. **Configure Slack Webhook**: Update notifier-agent with your Slack webhook URL
2. **Add LLM API Keys**: Update llm-service with your OpenAI/Anthropic/Gemini keys
3. **Setup Qdrant Cloud**: Create account and update memory-agent with credentials
4. **Initialize Database Schema**: Run database migrations (if not auto-applied)
5. **Create Test Incidents**: Manually trigger monitoring-agent to generate test incidents
6. **Monitor Logs**: Watch logs in real-time to ensure event flow works

---

## Summary

**Total Services**: 8 Container Apps + 4 Infrastructure Services
**Estimated Monthly Cost**: $51-66
**Deployment Time**: ~45-60 minutes
**Teardown Time**: ~5 minutes (delete resource group)

**Key Benefits**:
- No local Docker required (ACR Tasks builds in cloud)
- Automatic scaling and managed infrastructure
- Built-in observability with Log Analytics
- Easy teardown to stop costs immediately
- Well within $100 budget

---

## Support & References

- [CLOUD_PROVIDER] Container Apps Docs: https://learn.microsoft.com/azure/container-apps/
- ACR Tasks: https://learn.microsoft.com/azure/container-registry/container-registry-tasks-overview
- Cost Calculator: https://azure.microsoft.com/pricing/calculator/

---

**Questions?** Check the troubleshooting section or review logs using the commands provided above.
