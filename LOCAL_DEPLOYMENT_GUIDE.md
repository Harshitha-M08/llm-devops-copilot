# 🚀 Local Deployment Guide - DevOps Agent Platform

**Complete guide for running the entire platform locally on Windows/Linux/Mac**

---

## 📋 Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start (5 minutes)](#quick-start)
3. [Architecture Overview](#architecture-overview)
4. [Step-by-Step Deployment](#step-by-step-deployment)
5. [Accessing Services](#accessing-services)
6. [Verification & Testing](#verification--testing)
7. [Troubleshooting](#troubleshooting)
8. [Stopping & Cleanup](#stopping--cleanup)

---

## ✅ Prerequisites

### Required Software

| Software | Version | Purpose | Installation |
|----------|---------|---------|-------------|
| **Docker Desktop** | 20.10+ | Container runtime | [Download](https://www.docker.com/products/docker-desktop/) |
| **Docker Compose** | 2.0+ | Multi-container orchestration | Included with Docker Desktop |
| **Python** | 3.11+ | Running agents locally (optional) | [Download](https://www.python.org/downloads/) |
| **Node.js** | 18+ | Dashboard development (optional) | [Download](https://nodejs.org/) |
| **kubectl** | 1.25+ | Kubernetes CLI (optional for K8s) | [Install Guide](https://kubernetes.io/docs/tasks/tools/) |
| **Kind** | 0.20+ | Local Kubernetes (optional) | [Install Guide](https://kind.sigs.k8s.io/docs/user/quick-start/) |

### System Requirements

- **RAM**: 8GB minimum, 16GB recommended
- **CPU**: 4 cores minimum, 8 cores recommended
- **Disk**: 20GB free space
- **OS**: Windows 10/11, macOS 10.15+, Ubuntu 20.04+

### API Keys Required

You need at least ONE of these LLM provider API keys:

- **OpenAI API Key**: Get from [platform.openai.com](https://platform.openai.com/api-keys)
- **Anthropic API Key**: Get from [console.anthropic.com](https://console.anthropic.com/)

---

## 🚀 Quick Start (5 minutes)

### Option 1: Docker Compose Only (Easiest)

```bash
# 1. Clone the repository
git clone <your-repo-url>
cd LLM DevOps Copilot-main

# 2. Set up environment variables
cp .env.example .env
# Edit .env and add your OpenAI/Anthropic API keys

# 3. Start all services
docker-compose up -d

# 4. Wait for services to be ready (2-3 minutes)
docker-compose ps

# 5. Open dashboards
# - Approval Dashboard: http://localhost:3001
# - Ops Dashboard: http://localhost:3003
# - Grafana: http://localhost:3002 (admin/admin)
# - Prometheus: http://localhost:9090
# - RabbitMQ: http://localhost:15672 (devops/devops123)
```

### Option 2: Docker Compose + Kubernetes

```bash
# Follow steps 1-4 from Option 1, then:

# 5. Create local Kubernetes cluster
cd infrastructure/kind
../../bin/kind.exe create cluster --config kind-config-custom.yaml

# 6. Deploy to Kubernetes
kubectl apply -f ../kubernetes/base/namespace.yaml
kubectl apply -f ../kubernetes/base/rbac.yaml
kubectl apply -f ../kubernetes/base/
kubectl apply -f ../monitoring/

# 7. Verify Kubernetes pods
kubectl get pods -n devops-agent

# 8. Port forward Kubernetes services
kubectl port-forward -n devops-agent svc/prometheus 9091:9090 &
kubectl port-forward -n devops-agent svc/grafana 3004:3000 &
```

---

## 🏗️ Architecture Overview

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    LOCAL DEPLOYMENT STACK                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │           DOCKER COMPOSE SERVICES (16 containers)         │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │                                                            │  │
│  │  Infrastructure:                                           │  │
│  │  • PostgreSQL (port 5432)                                 │  │
│  │  • RabbitMQ (ports 5672, 15672)                          │  │
│  │  • Redis (port 6379)                                      │  │
│  │  • Qdrant (port 6333)                                     │  │
│  │  • Prometheus (port 9090)                                 │  │
│  │  • Grafana (port 3002)                                    │  │
│  │                                                            │  │
│  │  Agents:                                                   │  │
│  │  • Monitoring Agent                                        │  │
│  │  • Analyzer Agent                                          │  │
│  │  • Auto-Response Agent                                     │  │
│  │  • Notifier Agent                                          │  │
│  │  • Worker Service                                          │  │
│  │                                                            │  │
│  │  Services:                                                 │  │
│  │  • LLM Service (port 8000)                                │  │
│  │  • Approval Backend (port 3000)                           │  │
│  │  • Approval Frontend (port 3001)                          │  │
│  │  • Ops Dashboard Backend (port 8001)                      │  │
│  │  • Ops Dashboard Frontend (port 3003)                     │  │
│  │  • Test App (port 8080)                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         KUBERNETES CLUSTER (Optional - 8 pods)            │  │
│  ├──────────────────────────────────────────────────────────┤  │
│  │                                                            │  │
│  │  • Kind cluster: 1 control-plane + 2 workers             │  │
│  │  • RabbitMQ StatefulSet                                   │  │
│  │  • Nginx Deployment (3 replicas)                          │  │
│  │  • Python Backend (2 replicas)                            │  │
│  │  • Prometheus (port 9091)                                 │  │
│  │  • Grafana (port 3004)                                    │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📝 Step-by-Step Deployment

### Step 1: Environment Setup

1. **Clone Repository**
   ```bash
   git clone <your-repo-url>
   cd LLM DevOps Copilot-main
   ```

2. **Configure Environment Variables**
   ```bash
   # Copy example environment file
   cp .env.example .env

   # Edit .env file
   nano .env  # or use your preferred editor
   ```

3. **Required Configuration**

   Edit these critical variables in `.env`:

   ```ini
   # LLM Configuration (REQUIRED - Choose one)
   OPENAI_API_KEY=sk-your-openai-key-here
   # OR
   ANTHROPIC_API_KEY=sk-ant-your-anthropic-key-here

   # LLM Provider Selection
   LLM_PROVIDER=openai  # or 'anthropic'
   LLM_MODEL=gpt-4-turbo  # or 'claude-3-opus-20240229'

   # Optional: Slack Notifications
   SLACK_BOT_TOKEN=xoxb-your-slack-token  # Leave empty to disable
   SLACK_CHANNEL=#devops-alerts
   ```

### Step 2: Start Infrastructure Services

```bash
# Start core infrastructure first
docker-compose up -d postgres rabbitmq redis qdrant prometheus grafana

# Wait 30 seconds for databases to initialize
sleep 30

# Check if services are healthy
docker-compose ps
```

Expected output:
```
NAME                 STATUS                    PORTS
devops-postgres      Up (healthy)              5432
devops-rabbitmq      Up (healthy)              5672, 15672
devops-redis         Up (healthy)              6379
devops-qdrant        Up                        6333
devops-prometheus    Up                        9090
devops-grafana       Up                        3002
```

### Step 3: Initialize Database

```bash
# Run database migrations (if applicable)
docker-compose exec postgres psql -U devops -d devops_db -c "\dt"

# The database schema should be auto-created by the agents on first run
```

### Step 4: Start Agent Services

```bash
# Start all agents
docker-compose up -d llm-service monitoring-agent analyzer-agent auto-response-agent notifier-agent worker

# Check agent logs
docker-compose logs -f analyzer-agent
```

Look for these success messages:
```
✓ Analyzer Agent initialized (version 1.0.0)
🚀 Starting Analyzer Agent...
✓ Connected to RabbitMQ
✓ Connected to LLM Service
📊 Listening for incidents on routing key: monitoring.incident.*
```

### Step 5: Start Dashboard Services

```bash
# Start approval dashboard
docker-compose up -d approval-backend approval-frontend

# Start ops dashboard
docker-compose up -d ops-dashboard-backend ops-dashboard-frontend

# Start test application
docker-compose up -d test-app
```

### Step 6: Verify All Services

```bash
# Check all containers are running
docker-compose ps

# Should show 16 services running
# If any show "unhealthy", check logs:
docker-compose logs <service-name>
```

---

## 🌐 Accessing Services

### Primary Dashboards

| Dashboard | URL | Credentials | Purpose |
|-----------|-----|-------------|---------|
| **Approval Dashboard** | http://localhost:3001 | No auth | Review and approve remediation actions |
| **Ops Dashboard** | http://localhost:3003 | No auth | Real-time incident monitoring |
| **Grafana** | http://localhost:3002 | admin/admin | Metrics visualization |
| **Prometheus** | http://localhost:9090 | None | Metrics database & queries |
| **RabbitMQ Management** | http://localhost:15672 | devops/devops123 | Message queue monitoring |

### API Endpoints

| Service | Endpoint | Purpose |
|---------|----------|---------|
| **LLM Service** | http://localhost:8000/docs | LLM API documentation (FastAPI) |
| **Approval Backend** | http://localhost:3000/api | Approval API |
| **Ops Backend** | http://localhost:8001/api | Ops Dashboard API |
| **Test App** | http://localhost:8080 | Sample application for testing |

### Health Checks

```bash
# Check LLM Service
curl http://localhost:8000/health

# Check Approval Backend
curl http://localhost:3000/api/health

# Check Ops Dashboard Backend
curl http://localhost:8001/api/health

# Check Prometheus
curl http://localhost:9090/-/healthy
```

---

## ✅ Verification & Testing

### 1. Verify Infrastructure

```bash
# PostgreSQL
docker-compose exec postgres psql -U devops -d devops_db -c "SELECT version();"

# RabbitMQ
curl -u devops:devops123 http://localhost:15672/api/overview

# Redis
docker-compose exec redis redis-cli PING

# Qdrant
curl http://localhost:6333/collections
```

### 2. Verify Agents

```bash
# Check agent logs for successful startup
docker-compose logs monitoring-agent | grep "Starting"
docker-compose logs analyzer-agent | grep "initialized"
docker-compose logs auto-response-agent | grep "ready"
docker-compose logs notifier-agent | grep "Connected"
```

### 3. Test Incident Workflow

#### Option A: Trigger Test Incident via API

```bash
# Trigger a CPU spike in test app
curl -X POST http://localhost:8080/spike/cpu

# Watch the incident flow through agents
docker-compose logs -f monitoring-agent analyzer-agent auto-response-agent notifier-agent
```

#### Option B: Manual Incident Injection

```bash
# Publish test incident to RabbitMQ
docker-compose exec rabbitmq rabbitmqadmin publish \
  exchange=agent_events \
  routing_key=monitoring.incident.detected \
  payload='{"incident_id":"test-123","severity":"high","data":{"metric":"cpu_usage_percent","current_value":95,"threshold":80}}'
```

### 4. Monitor in Dashboards

1. **Open Ops Dashboard**: http://localhost:3003
   - Should show real-time incidents
   - WebSocket connection indicator should be green

2. **Open Approval Dashboard**: http://localhost:3001
   - Should show pending approvals (if any)

3. **Open Grafana**: http://localhost:3002
   - Navigate to "DevOps Agent Dashboard"
   - Should show metrics from monitoring agent

### 5. Run Test Suite

```bash
# Run all tests
powershell -File run_tests.ps1

# Run specific service tests
cd services/analyzer-agent
pytest tests/ -v -m unit --cov=app

cd ../auto-response-agent
pytest tests/ -v -m unit --cov=app
```

---

## 🐛 Troubleshooting

### Common Issues

#### Issue 1: Containers Show "Unhealthy"

**Symptom**: `docker-compose ps` shows services as "unhealthy"

**Solution**:
```bash
# Check health endpoint manually
curl http://localhost:8001/api/health

# If it responds with 200 OK, the service is fine
# Increase health check timeout in docker-compose.yml:
# healthcheck:
#   interval: 30s
#   timeout: 10s  # Increase this
#   retries: 5
```

#### Issue 2: Agents Can't Connect to RabbitMQ

**Symptom**: Logs show "Connection refused" or "Connection timeout"

**Solution**:
```bash
# Restart RabbitMQ
docker-compose restart rabbitmq

# Wait for it to be fully ready
docker-compose logs -f rabbitmq | grep "Server startup complete"

# Then restart agents
docker-compose restart monitoring-agent analyzer-agent auto-response-agent
```

#### Issue 3: LLM Service Returns Errors

**Symptom**: "Unauthorized" or "API key invalid"

**Solution**:
```bash
# Verify API key is set
docker-compose exec llm-service env | grep OPENAI_API_KEY

# If empty, add to .env file:
echo "OPENAI_API_KEY=sk-your-key" >> .env

# Restart LLM service
docker-compose restart llm-service
```

#### Issue 4: Dashboard Shows "No Data"

**Symptom**: Ops Dashboard or Grafana show no incidents/metrics

**Solution**:
```bash
# Check if monitoring agent is generating incidents
docker-compose logs monitoring-agent | grep "Incident detected"

# Manually trigger an incident
curl -X POST http://localhost:8080/spike/cpu

# Verify RabbitMQ is receiving messages
curl -u devops:devops123 http://localhost:15672/api/queues/%2Fdevops
```

#### Issue 5: Port Already in Use

**Symptom**: "bind: address already in use"

**Solution**:
```bash
# Find what's using the port (example for port 3000)
# Windows
netstat -ano | findstr :3000

# Linux/Mac
lsof -i :3000

# Kill the process or change port in docker-compose.yml
```

#### Issue 6: Out of Memory / Disk Space

**Symptom**: Containers crashing, Docker errors

**Solution**:
```bash
# Clean up Docker
docker system prune -a --volumes

# Increase Docker Desktop memory:
# Docker Desktop > Settings > Resources > Memory: 8GB+
```

### Viewing Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f analyzer-agent

# Last 100 lines
docker-compose logs --tail 100 analyzer-agent

# Follow errors only
docker-compose logs -f | grep ERROR
```

---

## 🛑 Stopping & Cleanup

### Stop All Services

```bash
# Stop all containers (keeps data)
docker-compose down

# Stop and remove volumes (DELETES DATA)
docker-compose down -v

# Stop specific service
docker-compose stop analyzer-agent
```

### Stop Kubernetes Cluster

```bash
# Delete Kind cluster
cd infrastructure/kind
../../bin/kind.exe delete cluster --name devops-agent
```

### Complete Cleanup

```bash
# Stop all containers
docker-compose down -v

# Remove all images
docker-compose down --rmi all

# Clean up Docker system
docker system prune -a --volumes -f

# Delete Kind cluster
kind delete cluster --name devops-agent
```

---

## 📊 Monitoring Your Local Deployment

### Key Metrics to Watch

1. **RabbitMQ Queues** (http://localhost:15672)
   - Message rates should be flowing
   - No messages stuck in queues

2. **Prometheus Targets** (http://localhost:9090/targets)
   - All targets should show "UP"

3. **Agent Logs**
   ```bash
   # Should see incident detection and analysis
   docker-compose logs -f monitoring-agent analyzer-agent
   ```

4. **Database Connections**
   ```bash
   # Check PostgreSQL connections
   docker-compose exec postgres psql -U devops -d devops_db -c "SELECT count(*) FROM pg_stat_activity;"
   ```

---

## 🚀 Next Steps

Now that your local deployment is running:

1. ✅ **Explore Dashboards**: Visit http://localhost:3001 and http://localhost:3003
2. ✅ **Trigger Incidents**: Use the test app at http://localhost:8080
3. ✅ **Review Logs**: Watch agent behavior with `docker-compose logs -f`
4. ✅ **Check Grafana**: Visualize metrics at http://localhost:3002
5. ✅ **Test Approvals**: Approve/reject actions in approval dashboard
6. ✅ **Run Tests**: Execute test suite with `powershell -File run_tests.ps1`

---

## 📚 Additional Resources

- **Project README**: `README.md`
- **Testing Guide**: `tests/README.md`
- **API Documentation**: http://localhost:8000/docs (LLM Service)
- **Architecture Diagrams**: `docs/architecture/`
- **Troubleshooting**: Check `docker-compose logs` for errors

---

## ✅ Success Criteria

Your local deployment is successful when:

- ✅ All 16 Docker containers show "Up" status
- ✅ Dashboards are accessible (ports 3001, 3003)
- ✅ Grafana shows metrics (port 3002)
- ✅ RabbitMQ management UI works (port 15672)
- ✅ Test incident triggers automated response
- ✅ Agent logs show incident detection → analysis → action → notification
- ✅ No error messages in logs (except expected "unhealthy" timeouts)

---

## 🎉 You're Ready!

Your **Agentic DevOps Platform** is now running locally!

**What's happening behind the scenes:**
- Monitoring agent watches for incidents
- Analyzer agent performs LLM-powered root cause analysis
- Auto-response agent executes or requests approval for fixes
- Notifier agent sends alerts (if Slack configured)
- Memory agent learns from incidents for future reference

**Try triggering an incident**: `curl -X POST http://localhost:8080/spike/cpu`

Then watch the magic happen in the logs and dashboards! 🎯
