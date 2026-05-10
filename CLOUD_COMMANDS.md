# Azure Cloud Commands - DevOps AI Agent System

**Purpose**: Step-by-step Azure CLI commands to deploy the DevOps AI Agent System to Azure Container Apps

**Target Audience**: Anyone cloning the GitHub repository and deploying to Azure

**Prerequisites**:
- Azure subscription (Azure for Students or Pay-As-You-Go)
- Azure CLI installed or Azure Cloud Shell access
- Docker images pushed to Docker Hub (Harshitha-M08/*)
- All required API keys (OpenAI, Anthropic, Qdrant, Slack)

---

## 📋 Table of Contents

1. [Initial Setup & Login](#1-initial-setup--login)
2. [Create Resource Group](#2-create-resource-group)
3. [Create Container Apps Environment](#3-create-container-apps-environment)
4. [Deploy PostgreSQL Database](#4-deploy-postgresql-database)
5. [Deploy Redis Cache](#5-deploy-redis-cache)
6. [Deploy RabbitMQ](#6-deploy-rabbitmq)
7. [Deploy Microservices](#7-deploy-microservices)
8. [Fix RabbitMQ Connections](#8-fix-rabbitmq-connections)
9. [Verify Deployment](#9-verify-deployment)
10. [Testing & Monitoring](#10-testing--monitoring)
11. [Cleanup (When Done)](#11-cleanup-when-done)

---

## 1. Initial Setup & Login

### Login to Azure
```bash
# Login to Azure account
az login

# Set subscription (if you have multiple)
az account set --subscription "Azure for Students"

# Verify current subscription
az account show
```

### Set Default Location
```bash
# Set default location to Central India
az configure --defaults location=centralindia
```

---

## 2. Create Resource Group

```bash
# Create resource group in Central India
az group create \
  --name devops-india-rg \
  --location centralindia

# Verify resource group created
az group show --name devops-india-rg
```

**Expected Output**: JSON showing resource group details with `"provisioningState": "Succeeded"`

---

## 3. Create Container Apps Environment

```bash
# Create Container Apps Environment
az containerapp env create \
  --name devops-ai-env \
  --resource-group devops-india-rg \
  --location centralindia

# Verify environment created
az containerapp env show \
  --name devops-ai-env \
  --resource-group devops-india-rg
```

**Expected Output**: Environment with `"provisioningState": "Succeeded"`

**Note**: This creates the internal network where all services will communicate.

---

## 4. Deploy PostgreSQL Database

```bash
# Create PostgreSQL Flexible Server
az postgres flexible-server create \
  --name devops-postgres \
  --resource-group devops-india-rg \
  --location centralindia \
  --admin-user devopsadmin \
  --admin-password "DevOps@123456" \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --storage-size 32 \
  --version 14 \
  --public-access 0.0.0.0-255.255.255.255

# Create database
az postgres flexible-server db create \
  --resource-group devops-india-rg \
  --server-name devops-postgres \
  --database-name devops_db

# Verify database created
az postgres flexible-server db show \
  --resource-group devops-india-rg \
  --server-name devops-postgres \
  --database-name devops_db
```

**Important Notes**:
- Username: `devopsadmin`
- Password: `DevOps@123456`
- Database: `devops_db`
- Port: `5432` (default)
- Public access enabled for Azure services (adjust for production)

---

## 5. Deploy Redis Cache

```bash
# Create Azure Cache for Redis (Basic C0 tier)
az redis create \
  --name devops-ai-redis \
  --resource-group devops-india-rg \
  --location centralindia \
  --sku Basic \
  --vm-size c0 \
  --enable-non-ssl-port false

# Get Redis connection string
az redis show \
  --name devops-ai-redis \
  --resource-group devops-india-rg \
  --query "hostName" -o tsv

# Get Redis access key
az redis list-keys \
  --name devops-ai-redis \
  --resource-group devops-india-rg \
  --query "primaryKey" -o tsv
```

**Important Notes**:
- Host: `devops-ai-redis.redis.cache.windows.net`
- Port: `6380` (SSL)
- Save the access key - you'll need it for environment variables

---

## 6. Deploy RabbitMQ

```bash
# Deploy RabbitMQ as a container app (internal ingress only)
az containerapp create \
  --name rabbitmq \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image rabbitmq:3-management \
  --target-port 5672 \
  --ingress internal \
  --transport tcp \
  --min-replicas 1 \
  --max-replicas 1 \
  --cpu 0.5 \
  --memory 1Gi \
  --env-vars \
    RABBITMQ_DEFAULT_USER=devops \
    RABBITMQ_DEFAULT_PASS=devops123 \
    RABBITMQ_DEFAULT_VHOST=devops

# Verify RabbitMQ is running
az containerapp show \
  --name rabbitmq \
  --resource-group devops-india-rg \
  --query "properties.runningStatus" -o tsv
```

**Important Notes**:
- **CRITICAL**: Other services must use `RABBITMQ_HOST=rabbitmq` (short name, NOT full FQDN!)
- User: `devops`
- Password: `devops123`
- VHost: `devops`
- Port: `5672` (AMQP)
- Ingress: Internal only (no external access)

---

## 7. Deploy Microservices

### 7.1 Deploy monitoring-agent

```bash
az containerapp create \
  --name monitoring-agent \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/monitoring-agent:latest \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    POSTGRES_HOST=devops-postgres.postgres.database.azure.com \
    POSTGRES_PORT=5432 \
    POSTGRES_DB=devops_db \
    POSTGRES_USER=devopsadmin \
    POSTGRES_PASSWORD=DevOps@123456 \
    RABBITMQ_HOST=rabbitmq \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    PROMETHEUS_URL=https://prometheus.lemonsea-56412705.centralindia.azurecontainerapps.io \
    K8S_IN_CLUSTER=false \
    ENVIRONMENT=production \
    LOG_LEVEL=INFO
```

**CRITICAL**:
- Set `PROMETHEUS_URL` to the **external HTTPS URL** of Prometheus (replace with your actual Prometheus FQDN)
- Set `K8S_IN_CLUSTER=false` since we're not running on Kubernetes
- Do NOT use internal FQDNs - they cause timeout issues in Azure Container Apps

### 7.2 Deploy analyzer-agent

```bash
az containerapp create \
  --name analyzer-agent \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/analyzer-agent:latest \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    POSTGRES_HOST=devops-postgres.postgres.database.azure.com \
    POSTGRES_PORT=5432 \
    POSTGRES_DB=devops_db \
    POSTGRES_USER=devopsadmin \
    POSTGRES_PASSWORD=DevOps@123456 \
    RABBITMQ_HOST=rabbitmq \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    QDRANT_URL=https://YOUR-QDRANT-CLUSTER.cloud.qdrant.io \
    QDRANT_API_KEY=YOUR_QDRANT_API_KEY \
    QDRANT_COLLECTION_NAME=incident_embeddings \
    LLM_SERVICE_URL=http://llm-service:8000 \
    ENVIRONMENT=production \
    LOG_LEVEL=INFO
```

**Note**: Replace `YOUR-QDRANT-CLUSTER` and `YOUR_QDRANT_API_KEY` with your actual Qdrant Cloud credentials.

### 7.3 Deploy auto-response-agent

```bash
az containerapp create \
  --name auto-response-agent \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/auto-response-agent:latest \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    RABBITMQ_HOST=rabbitmq \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    APPROVAL_API_URL=http://approval-dashboard-backend:5000 \
    K8S_IN_CLUSTER=false \
    AUTO_EXECUTE_THRESHOLD=90 \
    REQUIRES_APPROVAL_THRESHOLD=50 \
    ENVIRONMENT=production \
    LOG_LEVEL=INFO
```

**Important**: `AUTO_EXECUTE_THRESHOLD` must be an integer (0-100), NOT a float like 0.9!

### 7.4 Deploy notifier-agent

```bash
az containerapp create \
  --name notifier-agent \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/notifier-agent:latest \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    POSTGRES_HOST=devops-postgres.postgres.database.azure.com \
    POSTGRES_PORT=5432 \
    POSTGRES_DB=devops_db \
    POSTGRES_USER=devopsadmin \
    POSTGRES_PASSWORD=DevOps@123456 \
    RABBITMQ_HOST=rabbitmq \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    SLACK_BOT_TOKEN=xoxb-YOUR-SLACK-BOT-TOKEN \
    SLACK_CHANNEL=#devopsalerts \
    ENVIRONMENT=production \
    LOG_LEVEL=INFO
```

**Note**: Replace `YOUR-SLACK-BOT-TOKEN` with your actual Slack Bot Token from https://api.slack.com/apps

### 7.5 Deploy memory-agent (Optional - Has Known Issues)

```bash
# ⚠️ WARNING: memory-agent currently has OpenAI SDK version conflict
# Deploy at your own risk - will crash on startup

az containerapp create \
  --name memory-agent \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/memory-agent:latest \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    POSTGRES_HOST=devops-postgres.postgres.database.azure.com \
    POSTGRES_PORT=5432 \
    POSTGRES_DB=devops_db \
    POSTGRES_USER=devopsadmin \
    POSTGRES_PASSWORD=DevOps@123456 \
    RABBITMQ_HOST=rabbitmq \
    RABBITMQ_PORT=5672 \
    RABBITMQ_USER=devops \
    RABBITMQ_PASSWORD=devops123 \
    RABBITMQ_VHOST=devops \
    QDRANT_HOST=YOUR-QDRANT-CLUSTER.cloud.qdrant.io \
    QDRANT_API_KEY=YOUR_QDRANT_API_KEY \
    QDRANT_COLLECTION_NAME=incident_embeddings \
    OPENAI_API_KEY=sk-proj-YOUR-OPENAI-API-KEY \
    ENVIRONMENT=production \
    LOG_LEVEL=INFO
```

**Known Issue**: OpenAI SDK incompatibility. Agent will crash with `TypeError: AsyncClient.__init__() got an unexpected keyword argument 'proxies'`. See IMPLEMENTATION_TRACKER_2.md for fix.

### 7.6 Deploy llm-service

```bash
az containerapp create \
  --name llm-service \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/llm-service:latest \
  --target-port 8000 \
  --ingress internal \
  --cpu 0.5 \
  --memory 1Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    OPENAI_API_KEY=secretref:openai-api-key \
    ANTHROPIC_API_KEY=secretref:anthropic-api-key \
    QDRANT_URL=http://qdrant:6333 \
    QDRANT_API_KEY="" \
    LLM_PROVIDER=openai \
    LLM_MODEL=gpt-4 \
  --secrets \
    openai-api-key=sk-proj-YOUR-OPENAI-API-KEY \
    anthropic-api-key=sk-ant-YOUR-ANTHROPIC-API-KEY
```

**Important Notes:**
- Set `min-replicas 1` to avoid cold start timeouts when analyzer-agent calls it
- RAG pipeline is lazy-loaded - service starts without Qdrant
- `/api/v1/chat` endpoint works without Qdrant (used by analyzer-agent)
- Replace API keys with your actual keys

### 7.7 Deploy approval-dashboard-backend (Optional - Has Known Issues)

```bash
# ⚠️ WARNING: approval-dashboard-backend currently has missing User model file
# Deploy at your own risk - will crash on startup

az containerapp create \
  --name approval-dashboard-backend \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/approval-dashboard-backend:latest \
  --target-port 5000 \
  --ingress internal \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    POSTGRES_HOST=devops-postgres.postgres.database.azure.com \
    POSTGRES_PORT=5432 \
    POSTGRES_DB=devops_db \
    POSTGRES_USER=devopsadmin \
    POSTGRES_PASSWORD=DevOps@123456 \
    JWT_SECRET=your-super-secret-jwt-key-change-in-production \
    ENVIRONMENT=production \
    LOG_LEVEL=INFO
```

**Known Issue**: Missing `models/User.js` file. Backend will crash with `Error: Cannot find module '../models/User'`. See IMPLEMENTATION_TRACKER_2.md for fix.

### 7.8 Deploy approval-dashboard-frontend

```bash
az containerapp create \
  --name approval-dashboard-frontend \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/approval-dashboard-frontend:latest \
  --target-port 3000 \
  --ingress external \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1 \
  --env-vars \
    REACT_APP_API_URL=http://approval-dashboard-backend:5000

# Get the public URL
az containerapp show \
  --name approval-dashboard-frontend \
  --resource-group devops-india-rg \
  --query "properties.configuration.ingress.fqdn" -o tsv
```

**Note**: Save the frontend URL - this is your dashboard access point!

### 7.9 Deploy test-app

```bash
az containerapp create \
  --name test-app \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/test-app:latest \
  --target-port 8080 \
  --ingress external \
  --cpu 0.25 \
  --memory 0.5Gi \
  --min-replicas 1 \
  --max-replicas 1

# Get the public URL
az containerapp show \
  --name test-app \
  --resource-group devops-india-rg \
  --query "properties.configuration.ingress.fqdn" -o tsv
```

**Note**: Save the test-app URL - you'll use this to trigger chaos endpoints for testing!

### 7.10 Deploy Prometheus

```bash
# Deploy Prometheus with external ingress
az containerapp create \
  --name prometheus \
  --resource-group devops-india-rg \
  --environment devops-ai-env \
  --image Harshitha-M08/prometheus:latest \
  --target-port 9090 \
  --ingress external \
  --cpu 0.5 \
  --memory 1Gi \
  --min-replicas 1 \
  --max-replicas 1

# Get Prometheus URL
az containerapp show \
  --name prometheus \
  --resource-group devops-india-rg \
  --query "properties.configuration.ingress.fqdn" -o tsv
```

**Note**: Prometheus is deployed with **external ingress** to avoid internal networking timeout issues. The custom prometheus-azure.yml config file (baked into the Docker image) scrapes test-app metrics via its external FQDN.

---

## 8. Fix Monitoring & Prometheus Connectivity

### 8.1 Fix monitoring-agent Prometheus Connection

If monitoring-agent can't reach Prometheus (timeout errors), update the PROMETHEUS_URL to use the external HTTPS URL:

```bash
# Get Prometheus external URL first
PROM_URL=$(az containerapp show \
  --name prometheus \
  --resource-group devops-india-rg \
  --query "properties.configuration.ingress.fqdn" -o tsv)

# Update monitoring-agent with correct Prometheus URL
az containerapp update \
  --name monitoring-agent \
  --resource-group devops-india-rg \
  --set-env-vars PROMETHEUS_URL="https://${PROM_URL}"

# Restart monitoring-agent
az containerapp revision restart \
  --resource-group devops-india-rg \
  --name monitoring-agent \
  --revision $(az containerapp revision list \
    --name monitoring-agent \
    --resource-group devops-india-rg \
    --query "[0].name" \
    --output tsv)

# Verify monitoring-agent can now query Prometheus (wait 30 seconds)
az containerapp logs show \
  --name monitoring-agent \
  --resource-group devops-india-rg \
  --tail 30
```

**Look for**: `Prometheus client initialized: https://prometheus...` and **no more** "Unexpected error querying Prometheus" errors.

### 8.2 Fix analyzer-agent LLM Service Connection

If analyzer-agent times out when connecting to llm-service, use the internal FQDN:

```bash
# Update analyzer-agent with llm-service internal FQDN
az containerapp update \
  --name analyzer-agent \
  --resource-group devops-india-rg \
  --set-env-vars LLM_SERVICE_URL="http://llm-service.internal.lemonsea-56412705.centralindia.azurecontainerapps.io:8000"

# Restart analyzer-agent
az containerapp revision restart \
  --resource-group devops-india-rg \
  --name analyzer-agent \
  --revision $(az containerapp revision list \
    --name analyzer-agent \
    --resource-group devops-india-rg \
    --query "[0].name" \
    --output tsv)
```

---

## 9. Fix RabbitMQ Connections

**CRITICAL FIX**: After initial deployment, some agents may fail to connect to RabbitMQ due to using full FQDN instead of short name. Apply these fixes:

### 9.1 Fix monitoring-agent (if needed)

```bash
# Update RABBITMQ_HOST to short name
az containerapp update \
  --name monitoring-agent \
  --resource-group devops-india-rg \
  --set-env-vars RABBITMQ_HOST=rabbitmq

# Verify connection in logs (wait 30 seconds after update)
az containerapp logs show \
  --name monitoring-agent \
  --resource-group devops-india-rg \
  --type console --tail 20
```

**Look for**: `✓ Connected to RabbitMQ exchange: agent_events`

###9.2 Fix auto-response-agent (if needed)

```bash
# Update RABBITMQ_HOST and fix AUTO_EXECUTE_THRESHOLD
az containerapp update \
  --name auto-response-agent \
  --resource-group devops-india-rg \
  --set-env-vars \
    RABBITMQ_HOST=rabbitmq \
    AUTO_EXECUTE_THRESHOLD=90

# Verify connection in logs (wait 30 seconds after update)
az containerapp logs show \
  --name auto-response-agent \
  --resource-group devops-india-rg \
  --type console --tail 20
```

**Look for**:
- `Auto-execute threshold: 90%`
- `✓ Event consumer connected to RabbitMQ`

---

## 9. Verify Deployment

### 9.1 Check All Services Status

```bash
# List all container apps
az containerapp list \
  --resource-group devops-india-rg \
  --query "[].{Name:name, Status:properties.runningStatus, FQDN:properties.configuration.ingress.fqdn}" \
  --output table
```

**Expected Output**: All services should show `Status: Running`

### 9.2 Verify RabbitMQ Connections

```bash
# Check monitoring-agent
az containerapp logs show \
  --name monitoring-agent \
  --resource-group devops-india-rg \
  --type console --tail 20 | grep -i "connected"

# Check analyzer-agent
az containerapp logs show \
  --name analyzer-agent \
  --resource-group devops-india-rg \
  --type console --tail 20 | grep -i "connected"

# Check auto-response-agent
az containerapp logs show \
  --name auto-response-agent \
  --resource-group devops-india-rg \
  --type console --tail 20 | grep -i "connected"

# Check notifier-agent
az containerapp logs show \
  --name notifier-agent \
  --resource-group devops-india-rg \
  --type console --tail 20 | grep -i "connected"
```

**Expected**: Each agent should show `✓ Connected to RabbitMQ` or similar success message.

### 9.3 Check Database Connectivity

```bash
# Connect to PostgreSQL and verify tables
az postgres flexible-server execute \
  --name devops-postgres \
  --admin-user devopsadmin \
  --admin-password "DevOps@123456" \
  --database-name devops_db \
  --querytext "SELECT table_name FROM information_schema.tables WHERE table_schema='public';"
```

**Expected**: Should show tables created by the agents (incidents, analyses, actions, etc.)

---

## 10. Testing & Monitoring

### 10.1 Get Test App URL

```bash
# Get test-app URL for triggering chaos
az containerapp show \
  --name test-app \
  --resource-group devops-india-rg \
  --query "properties.configuration.ingress.fqdn" -o tsv
```

### 10.2 Trigger Chaos Endpoints

```bash
# Replace <test-app-url> with the URL from previous command

# Trigger CPU spike
curl -X POST https://<test-app-url>/api/chaos/cpu

# Trigger memory spike
curl -X POST https://<test-app-url>/api/chaos/memory

# Trigger error rate spike
curl -X POST https://<test-app-url>/api/chaos/errors
```

### 10.3 Monitor Workflow in Real-Time

Open 4 separate terminals and run these commands:

**Terminal 1 - monitoring-agent**:
```bash
az containerapp logs show \
  --name monitoring-agent \
  --resource-group devops-india-rg \
  --type console --tail 50 --follow
```

**Terminal 2 - analyzer-agent**:
```bash
az containerapp logs show \
  --name analyzer-agent \
  --resource-group devops-india-rg \
  --type console --tail 50 --follow
```

**Terminal 3 - auto-response-agent**:
```bash
az containerapp logs show \
  --name auto-response-agent \
  --resource-group devops-india-rg \
  --type console --tail 50 --follow
```

**Terminal 4 - notifier-agent**:
```bash
az containerapp logs show \
  --name notifier-agent \
  --resource-group devops-india-rg \
  --type console --tail 50 --follow
```

**Expected Flow**:
1. monitoring-agent detects: `🚨 Incident detected`
2. analyzer-agent processes: `Analysis complete`
3. auto-response-agent decides: `Action generated`
4. notifier-agent alerts: `Slack notification sent`

### 10.4 Check Slack Notifications

Open your Slack workspace and check the `#devopsalerts` channel for incident notifications.

---

## 11. Stop/Start Services (Cost Management)

### 11.1 Stop All Services (Deactivate Revisions - Saves Money)

**Why**: Deactivating revisions stops all containers but keeps configuration intact. Costs drop from ~$4-5/day to ~$0.50/day (only database + Redis).

**Step 1: Get Current Revision Names**

```bash
# Get revision names for all services
az containerapp revision list --name monitoring-agent --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name analyzer-agent --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name auto-response-agent --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name notifier-agent --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name memory-agent --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name llm-service --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name approval-dashboard-backend --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name approval-dashboard-frontend --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name test-app --resource-group devops-india-rg --query "[0].name" -o tsv
az containerapp revision list --name rabbitmq --resource-group devops-india-rg --query "[0].name" -o tsv
```

**Step 2: Deactivate All Revisions** (use the revision names from Step 1)

```bash
# Deactivate monitoring-agent (replace with your revision name)
az containerapp revision deactivate --name monitoring-agent --resource-group devops-india-rg --revision monitoring-agent--0000004

# Deactivate analyzer-agent
az containerapp revision deactivate --name analyzer-agent --resource-group devops-india-rg --revision analyzer-agent--0000001

# Deactivate auto-response-agent
az containerapp revision deactivate --name auto-response-agent --resource-group devops-india-rg --revision auto-response-agent--0000002

# Deactivate notifier-agent
az containerapp revision deactivate --name notifier-agent --resource-group devops-india-rg --revision notifier-agent--0000001

# Deactivate memory-agent
az containerapp revision deactivate --name memory-agent --resource-group devops-india-rg --revision memory-agent--0000004

# Deactivate llm-service
az containerapp revision deactivate --name llm-service --resource-group devops-india-rg --revision llm-service--mxmofdo

# Deactivate approval-dashboard-backend
az containerapp revision deactivate --name approval-dashboard-backend --resource-group devops-india-rg --revision approval-dashboard-backend--wyn7ogl

# Deactivate approval-dashboard-frontend
az containerapp revision deactivate --name approval-dashboard-frontend --resource-group devops-india-rg --revision approval-dashboard-frontend--ggam14u

# Deactivate test-app
az containerapp revision deactivate --name test-app --resource-group devops-india-rg --revision test-app--0000001

# Deactivate rabbitmq
az containerapp revision deactivate --name rabbitmq --resource-group devops-india-rg --revision rabbitmq--2oyckb0
```

**Expected Output**: `"Deactivate succeeded"` for each service.

---

### 11.2 Restart All Services (Reactivate or Create New Revisions)

**Option A: Reactivate Existing Revisions** (Fastest - 1 minute)

```bash
# Reactivate revisions (use the same revision names from deactivate step)
az containerapp revision activate --name rabbitmq --resource-group devops-india-rg --revision rabbitmq--2oyckb0

# Wait 30 seconds for RabbitMQ to start, then activate agents
sleep 30

az containerapp revision activate --name monitoring-agent --resource-group devops-india-rg --revision monitoring-agent--0000004
az containerapp revision activate --name analyzer-agent --resource-group devops-india-rg --revision analyzer-agent--0000001
az containerapp revision activate --name auto-response-agent --resource-group devops-india-rg --revision auto-response-agent--0000002
az containerapp revision activate --name notifier-agent --resource-group devops-india-rg --revision notifier-agent--0000001
az containerapp revision activate --name llm-service --resource-group devops-india-rg --revision llm-service--mxmofdo
az containerapp revision activate --name test-app --resource-group devops-india-rg --revision test-app--0000001
```

**Option B: Create New Revisions** (If reactivate doesn't work)

```bash
# Force new revisions by updating min/max replicas
az containerapp update --name rabbitmq --resource-group devops-india-rg --min-replicas 1 --max-replicas 1

# Wait 30 seconds for RabbitMQ
sleep 30

az containerapp update --name monitoring-agent --resource-group devops-india-rg --min-replicas 1 --max-replicas 1
az containerapp update --name analyzer-agent --resource-group devops-india-rg --min-replicas 1 --max-replicas 1
az containerapp update --name auto-response-agent --resource-group devops-india-rg --min-replicas 1 --max-replicas 1
az containerapp update --name notifier-agent --resource-group devops-india-rg --min-replicas 1 --max-replicas 1
az containerapp update --name llm-service --resource-group devops-india-rg --min-replicas 1 --max-replicas 1
az containerapp update --name test-app --resource-group devops-india-rg --min-replicas 1 --max-replicas 1
```

---

### 11.3 Verify Services Are Running

```bash
# Check all services status
az containerapp list \
  --resource-group devops-india-rg \
  --query "[].{Name:name, Status:properties.runningStatus}" \
  --output table
```

**Expected**: All services should show `Status: Running`

---

### 11.4 Delete Entire Resource Group (IRREVERSIBLE!)

```bash
# ⚠️ WARNING: This deletes EVERYTHING in the resource group!
# All data, services, and configurations will be permanently lost!

az group delete \
  --name devops-india-rg \
  --yes \
  --no-wait

# Check deletion status
az group show --name devops-india-rg
```

**Expected**: After a few minutes, you'll get an error saying resource group doesn't exist (deletion complete).

---

## 📝 Quick Reference

### Essential URLs

| Service | URL Format | Access |
|---------|-----------|--------|
| Frontend Dashboard | `https://approval-dashboard-frontend.<random>.centralindia.azurecontainerapps.io` | Public |
| Test App | `https://test-app.<random>.centralindia.azurecontainerapps.io` | Public |
| Backend API | `http://approval-dashboard-backend.internal.<random>.centralindia.azurecontainerapps.io` | Internal only |
| RabbitMQ | `rabbitmq:5672` | Internal only |
| PostgreSQL | `devops-postgres.postgres.database.azure.com:5432` | Azure services |
| Redis | `devops-ai-redis.redis.cache.windows.net:6380` | Azure services |

### Environment Variables Summary

**Required for All Agents**:
- `POSTGRES_HOST`, `POSTGRES_PORT`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`
- `RABBITMQ_HOST=rabbitmq` (CRITICAL: use short name!)
- `RABBITMQ_PORT=5672`
- `RABBITMQ_USER=devops`
- `RABBITMQ_PASSWORD=devops123`
- `RABBITMQ_VHOST=devops`

**Agent-Specific**:
- **analyzer-agent, memory-agent**: `QDRANT_URL`, `QDRANT_API_KEY`, `QDRANT_COLLECTION_NAME`
- **llm-service**: `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`
- **notifier-agent**: `SLACK_BOT_TOKEN`, `SLACK_CHANNEL`
- **auto-response-agent**: `AUTO_EXECUTE_THRESHOLD=90` (integer!), `K8S_IN_CLUSTER=false`
- **approval-dashboard-backend**: `JWT_SECRET`

### Cost Estimates (Azure for Students)

| Resource | Daily Cost | Monthly Cost |
|----------|-----------|--------------|
| Container Apps Environment | ~$0.50 | ~$15 |
| PostgreSQL (B1ms) | ~$1.00 | ~$30 |
| Redis (Basic C0) | ~$0.50 | ~$15 |
| Container Apps (9 services) | ~$2-3 | ~$60-90 |
| **TOTAL** | **~$4-5/day** | **~$120-150/month** |

**⚠️ WARNING**: This exceeds the $100 Azure for Students credit! Monitor spending daily and scale down when not testing.

---

## 🐛 Common Issues & Solutions

### Issue 1: RabbitMQ Connection Timeout
**Symptom**: `[Errno 110] Connect call failed`
**Solution**: Use `RABBITMQ_HOST=rabbitmq` (short name), NOT full FQDN

### Issue 2: auto-response-agent Config Error
**Symptom**: `Input should be a valid integer, unable to parse string as an integer`
**Solution**: Set `AUTO_EXECUTE_THRESHOLD=90` (integer), NOT `0.9` (float)

### Issue 3: memory-agent Crashes
**Symptom**: `TypeError: AsyncClient.__init__() got an unexpected keyword argument 'proxies'`
**Solution**: OpenAI SDK version conflict - needs code fix (see IMPLEMENTATION_TRACKER_2.md)

### Issue 4: Dashboard Backend Crashes
**Symptom**: `Error: Cannot find module '../models/User'`
**Solution**: Missing file in Docker image - needs rebuild (see IMPLEMENTATION_TRACKER_2.md)

### Issue 5: PostgreSQL Connection Refused
**Symptom**: `Connection refused` or timeout
**Solution**: Check firewall rules - ensure Azure services can access PostgreSQL

---

## 📚 Additional Resources

- **GitHub Repository**: https://github.com/Harshitha-M08/LLM DevOps Copilot
- **Docker Hub**: https://hub.docker.com/u/Harshitha-M08
- **Implementation Tracker**: `devops/IMPLEMENTATION_TRACKER_2.md`
- **Session Summary**: `devops/SESSION_SUMMARY_OCT26.md`
- **Azure Container Apps Docs**: https://learn.microsoft.com/en-us/azure/container-apps/

---

## 🎯 Success Criteria

Your deployment is successful when:
- ✅ All 9 services show `Status: Running`
- ✅ 4 agents connected to RabbitMQ (monitoring, analyzer, auto-response, notifier)
- ✅ Triggering chaos endpoint generates logs in all agents
- ✅ Slack notifications received in #devopsalerts channel
- ✅ Dashboard frontend loads (even if backend is broken)

---

**Document Version**: 1.0
**Created**: October 26, 2025
**Last Updated**: October 26, 2025 16:45 UTC
**Tested On**: Azure Container Apps (Central India)
**Status**: Production-ready (except memory-agent and dashboard backend)
