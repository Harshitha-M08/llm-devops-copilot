# [CLOUD_PROVIDER] Deployment - Quick Start Guide

## 🚀 Deploy in 3 Steps (Without Docker Desktop)

### Prerequisites
- [CLOUD_PROVIDER] account with $100 credits
- Access to [CLOUD_PROVIDER] Cloud Shell (browser-based, no installation needed)

---

## Step 1: Prepare Configuration

### 1.1 Get Required API Keys

Before deployment, sign up and get these API keys:

| Service | Sign Up URL | What You Need |
|---------|------------|---------------|
| **OpenAI** | https://platform.openai.com/api-keys | API Key (starts with `sk-`) |
| **Qdrant Cloud** | https://cloud.qdrant.io/ | Free tier cluster + API key |
| **Slack** | https://api.slack.com/messaging/webhooks | Webhook URL |

### 1.2 Edit Deployment Script

Open `deploy-to-azure.sh` and update the CONFIGURATION section:

```bash
# [CLOUD_PROVIDER] Configuration
SUBSCRIPTION_ID="your-subscription-id"  # Get with: az account list
RESOURCE_GROUP="devops-ai-test-rg"
LOCATION="eastus"
ACR_NAME="devopsairegistry"

# LLM API Keys (REQUIRED)
OPENAI_API_KEY="sk-your-actual-openai-key"
ANTHROPIC_API_KEY="sk-ant-your-anthropic-key"  # Optional
GEMINI_API_KEY="your-gemini-key"  # Optional

# Qdrant Cloud (REQUIRED)
QDRANT_URL="https://your-cluster.qdrant.io"
QDRANT_API_KEY="your-qdrant-api-key"

# Slack Configuration (REQUIRED)
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
SLACK_CHANNEL="#devops-alerts"
```

---

## Step 2: Upload & Deploy

### 2.1 Open [CLOUD_PROVIDER] Cloud Shell

1. Go to https://portal.azure.com
2. Click the Cloud Shell icon (top right)
3. Select **Bash** environment

### 2.2 Upload Project Files

In [CLOUD_PROVIDER] Cloud Shell:

```bash
# Option A: Upload via Cloud Shell UI
# Click "Upload/Download files" button → Select your devops/ folder

# Option B: Clone from Git (if you have a repo)
git clone <YOUR_REPO_URL>
cd <PROJECT_DIR>/devops
```

### 2.3 Run Deployment Script

```bash
# Make script executable
chmod +x deploy-to-azure.sh

# Run deployment (takes ~20-30 minutes)
./deploy-to-azure.sh
```

The script will automatically:
- ✅ Create all [CLOUD_PROVIDER] resources
- ✅ Build Docker images in the cloud (no local Docker needed!)
- ✅ Deploy 8 services + infrastructure
- ✅ Configure networking and environment variables

---

## Step 3: Verify & Test

### 3.1 Check Deployment Status

```bash
# Run verification script
chmod +x verify-deployment.sh
./verify-deployment.sh
```

### 3.2 Access Approval Dashboard

After deployment completes, you'll see:

```
Approval Dashboard: https://approval-dashboard-frontend.xxx.eastus.azurecontainerapps.io
```

Open this URL in your browser to access the dashboard.

### 3.3 View Logs

```bash
# Monitor monitoring-agent logs
az containerapp logs show \
  --name monitoring-agent \
  --resource-group devops-ai-test-rg \
  --follow

# Monitor analyzer-agent logs
az containerapp logs show \
  --name analyzer-agent \
  --resource-group devops-ai-test-rg \
  --follow
```

---

## 📊 What Gets Deployed

### Services (8 Container Apps)
1. **monitoring-agent** - Monitors Prometheus metrics & K8s resources
2. **analyzer-agent** - LLM-powered root cause analysis
3. **auto-response-agent** - Autonomous remediation
4. **notifier-agent** - Slack notifications
5. **memory-agent** - Incident storage & RAG
6. **llm-service** - Multi-provider LLM API
7. **approval-dashboard-backend** - REST API
8. **approval-dashboard-frontend** - React UI

### Infrastructure
- **[CLOUD_PROVIDER] Container Registry** - Image storage
- **PostgreSQL** (B1ms) - Database
- **Redis** (Basic C0) - Cache
- **RabbitMQ** - Message broker
- **Log Analytics** - Observability

---

## 💰 Cost Breakdown

| Service | SKU | Monthly Cost |
|---------|-----|--------------|
| Container Apps | Consumption | $10-20 |
| PostgreSQL | B1ms | $15 |
| Redis | Basic C0 | $16 |
| Container Registry | Basic | $5 |
| Log Analytics | Pay-as-you-go | $5-10 |
| **Total** | | **$51-66** |

✅ Well under $100 budget!

---

## 🧹 Cleanup (When Done Testing)

### Quick Cleanup (Delete Everything)

```bash
# Run cleanup script
chmod +x cleanup-azure.sh
./cleanup-azure.sh
```

This will:
- Delete the entire resource group
- Remove all services and data
- Stop all costs immediately

### Manual Cleanup

```bash
# Delete resource group (everything in it)
az group delete --name devops-ai-test-rg --yes --no-wait

# Verify deletion
az group exists --name devops-ai-test-rg
```

---

## 🔧 Troubleshooting

### Issue: "Container App won't start"

```bash
# Check logs
az containerapp logs show \
  --name <SERVICE_NAME> \
  --resource-group devops-ai-test-rg \
  --tail 100

# Check environment variables
az containerapp show \
  --name <SERVICE_NAME> \
  --resource-group devops-ai-test-rg \
  --query properties.template.containers[0].env
```

### Issue: "Services can't communicate"

All services must be in the same Container Apps Environment. Verify:

```bash
az containerapp list \
  --resource-group devops-ai-test-rg \
  --query "[].{Name:name, Env:properties.environmentId}" \
  --output table
```

### Issue: "Database connection fails"

Check firewall rules:

```bash
az postgres flexible-server firewall-rule list \
  --resource-group devops-ai-test-rg \
  --name devops-ai-postgres
```

### Issue: "ACR build fails"

Make sure you're in the `devops/` directory:

```bash
pwd  # Should show: .../devops
ls services/  # Should list: monitoring-agent, analyzer-agent, etc.
```

---

## 📚 Additional Resources

- **Full Deployment Guide**: See `AZURE_DEPLOYMENT_GUIDE.md` for detailed documentation
- **Environment Variables**: See `.env.template` for all configuration options
- **[CLOUD_PROVIDER] Container Apps Docs**: https://learn.microsoft.com/azure/container-apps/
- **ACR Tasks**: https://learn.microsoft.com/azure/container-registry/container-registry-tasks-overview

---

## 🎯 Testing Checklist

After deployment, test these flows:

### 1. End-to-End Flow
- [ ] Monitoring agent detects anomaly
- [ ] Analyzer agent analyzes with LLM
- [ ] Auto-response agent validates action
- [ ] Memory agent stores incident
- [ ] Notifier agent sends Slack message

### 2. Approval Flow
- [ ] Open approval dashboard
- [ ] Create test approval request
- [ ] Approve/reject action
- [ ] Check execution logs

### 3. RAG Search
- [ ] Store multiple incidents
- [ ] Query similar incidents
- [ ] Verify Qdrant returns results

---

## ❓ Questions?

Check the troubleshooting section in `AZURE_DEPLOYMENT_GUIDE.md` or review logs:

```bash
# View all services
az containerapp list --resource-group devops-ai-test-rg --output table

# Get specific service details
az containerapp show --name <SERVICE_NAME> --resource-group devops-ai-test-rg
```

---

## 🎉 Success!

You should now have a fully functional AI-powered DevOps system running on [CLOUD_PROVIDER]!

**Next Steps:**
1. Configure Slack webhook
2. Add real Prometheus metrics
3. Connect to actual Kubernetes cluster (optional)
4. Test incident detection and response

**When done testing:**
```bash
./cleanup-azure.sh
```

---

**Estimated Setup Time:** 30-45 minutes
**Estimated Cleanup Time:** 5 minutes
**Monthly Cost:** $51-66 (under $100 budget ✅)
