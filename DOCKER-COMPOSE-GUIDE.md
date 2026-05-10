# Docker Compose Startup Guide

## 🎯 Overview

This guide will help you start the entire AI-Driven Hybrid Kubernetes System locally using Docker Compose. The platform consists of multiple microservices that work together to provide LLM operations with human-in-the-loop approval workflows.

## 📋 Prerequisites

Before starting, ensure you have the following installed:

- **Docker Desktop** (version 20.10+ recommended)
  - Windows: https://docs.docker.com/desktop/install/windows-install/
  - Verify installation: `docker --version`
- **Docker Compose** (usually included with Docker Desktop)
  - Verify installation: `docker-compose --version`
- **Python 3.11+** (for local testing)
- **Node.js 20+** (for local testing)
- **Valid API Keys**:
  - OpenAI API key from https://platform.openai.com/api-keys
  - Anthropic API key from https://console.anthropic.com/

## 🔧 Configuration

### Step 1: Update Environment Variables

Edit the `.env` file in the `devops/` directory and configure the following critical variables:

```bash
# IMPORTANT: Replace with your actual API keys
OPENAI_API_KEY=sk-proj-YOUR_ACTUAL_OPENAI_KEY_HERE
ANTHROPIC_API_KEY=sk-ant-YOUR_ACTUAL_ANTHROPIC_KEY_HERE

# Optional: Configure email notifications
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM=noreply@devops-platform.com

# Optional: Configure Slack notifications
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SLACK_BOT_TOKEN=xoxb-your-slack-bot-token
```

### Step 2: Verify Docker Compose Configuration

Navigate to the devops directory and validate the configuration:

```bash
cd devops
docker-compose config
```

If there are any errors, they will be displayed here. Fix them before proceeding.

## 🚀 Starting Services

### Option 1: Start All Services (Recommended for first-time setup)

Start all services in detached mode (background):

```bash
cd devops
docker-compose up -d
```

This will start the following services:
1. **PostgreSQL** (port 5432) - Main database
2. **Redis** (port 6379) - Cache and session store
3. **RabbitMQ** (ports 5672, 15672) - Message queue
4. **Qdrant** (ports 6333, 6334) - Vector database for RAG
5. **LLM Service** (port 8000) - Python FastAPI service
6. **Worker Service** - Background task processor
7. **Approval Backend** (port 3000) - Node.js/Express API
8. **Approval Frontend** (port 3001) - React dashboard
9. **NGINX** (ports 80, 443) - Reverse proxy

### Option 2: Start Services with Logs Visible

If you want to see logs in real-time:

```bash
cd devops
docker-compose up
```

Press `Ctrl+C` to stop all services.

### Option 3: Start Individual Services

Start only specific services:

```bash
# Start database services only
docker-compose up -d postgres redis rabbitmq qdrant

# Start application services
docker-compose up -d llm-service worker-service approval-backend approval-frontend

# Start monitoring stack
docker-compose up -d prometheus grafana
```

## 📊 Monitoring Service Status

### Check Running Containers

```bash
docker-compose ps
```

Expected output (when all services are healthy):

```
Name                   Command                  State           Ports
-----------------------------------------------------------------------------------
devops-postgres        docker-entrypoint.sh postgres   Up   5432/tcp
devops-redis           docker-entrypoint.sh redis-...  Up   6379/tcp
devops-rabbitmq        docker-entrypoint.sh rabbi...  Up   5672/tcp, 15672/tcp
devops-qdrant          ./qdrant --config-path con...  Up   6333/tcp, 6334/tcp
devops-llm-service     uvicorn main:app --host 0....  Up   8000/tcp
devops-worker          python main.py                  Up
devops-approval-backend node server.js                Up   3000/tcp
devops-approval-frontend npm start                    Up   3001/tcp
devops-nginx           nginx -g daemon off;           Up   80/tcp, 443/tcp
```

### View Logs

View logs for all services:

```bash
docker-compose logs -f
```

View logs for a specific service:

```bash
docker-compose logs -f llm-service
docker-compose logs -f worker-service
docker-compose logs -f approval-backend
```

## 🔍 Health Checks

Once services are running, verify they are working correctly:

### 1. PostgreSQL Database

```bash
docker-compose exec postgres psql -U devops -d devops_db -c "SELECT version();"
```

Expected: PostgreSQL version information

### 2. Redis Cache

```bash
docker-compose exec redis redis-cli ping
```

Expected: `PONG`

### 3. RabbitMQ Management UI

Open in browser: http://localhost:15672
- Username: `devops`
- Password: `devops123`

Expected: RabbitMQ management dashboard

### 4. Qdrant Vector Database

```bash
curl http://localhost:6333/
```

Expected: Qdrant API response with version information

### 5. LLM Service API

```bash
curl http://localhost:8000/health
```

Expected response:

```json
{
  "status": "healthy",
  "service": "llm-service",
  "version": "1.0.0",
  "timestamp": "2025-10-18T..."
}
```

Test chat completion (requires valid API key):

```bash
curl -X POST http://localhost:8000/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello!"}
    ],
    "provider": "openai"
  }'
```

### 6. Approval Backend API

```bash
curl http://localhost:3000/health
```

Expected response:

```json
{
  "status": "healthy",
  "service": "approval-backend",
  "version": "1.0.0"
}
```

### 7. Approval Frontend

Open in browser: http://localhost:3001

Expected: React application login page

**Default Test Credentials:**
- Admin: `admin@devops.local` / `Admin@123`
- Approver: `approver@devops.local` / `Approver@123`
- Developer: `developer@devops.local` / `Developer@123`
- Viewer: `viewer@devops.local` / `Viewer@123`

## 🧪 Testing the Complete Workflow

### Test 1: User Authentication

```bash
# Login as admin
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@devops.local",
    "password": "Admin@123"
  }'
```

Save the returned JWT token for subsequent requests.

### Test 2: LLM Chat Completion

```bash
curl -X POST http://localhost:8000/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Explain Kubernetes in one sentence."}
    ],
    "provider": "openai",
    "max_tokens": 100
  }'
```

### Test 3: Document Ingestion for RAG

```bash
curl -X POST http://localhost:8000/api/v1/documents/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Kubernetes is an open-source container orchestration platform that automates deployment, scaling, and management of containerized applications.",
    "metadata": {
      "source": "documentation",
      "category": "kubernetes"
    }
  }'
```

### Test 4: RAG Query

```bash
curl -X POST http://localhost:8000/api/v1/rag/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Kubernetes?",
    "top_k": 3
  }'
```

### Test 5: Create Approval Request

```bash
curl -X POST http://localhost:3000/api/approvals \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -d '{
    "title": "Deploy New Feature",
    "description": "Request approval to deploy feature X to production",
    "priority": "high",
    "metadata": {
      "environment": "production",
      "service": "api-gateway"
    }
  }'
```

## 🛠️ Troubleshooting

### Services Not Starting

1. **Check Docker resources**: Ensure Docker Desktop has sufficient memory (at least 8GB recommended)
2. **Check port conflicts**: Ensure ports 80, 443, 3000, 3001, 5432, 5672, 6333, 6379, 8000, 15672 are not in use
3. **View service logs**: `docker-compose logs [service-name]`

### Database Connection Errors

```bash
# Restart PostgreSQL
docker-compose restart postgres

# Check database logs
docker-compose logs postgres

# Verify database is initialized
docker-compose exec postgres psql -U devops -d devops_db -c "\dt"
```

### RabbitMQ Issues

```bash
# Restart RabbitMQ
docker-compose restart rabbitmq

# Check queue status
docker-compose exec rabbitmq rabbitmqctl list_queues

# Re-initialize queues
docker-compose exec rabbitmq bash /docker-entrypoint-initdb.d/init-queues.sh
```

### LLM Service API Errors

If you see `Incorrect API key` errors:

1. Verify your API keys in `.env` are correct
2. Ensure API keys start with correct prefixes:
   - OpenAI: `sk-proj-` or `sk-`
   - Anthropic: `sk-ant-`
3. Restart LLM service: `docker-compose restart llm-service`

### Worker Service Not Processing Tasks

```bash
# Check worker logs
docker-compose logs worker-service

# Verify RabbitMQ connection
docker-compose exec rabbitmq rabbitmqctl list_connections

# Restart worker
docker-compose restart worker-service
```

## 🔄 Restarting Services

Restart all services:

```bash
docker-compose restart
```

Restart specific service:

```bash
docker-compose restart llm-service
```

## 🛑 Stopping Services

Stop all services (preserves data):

```bash
docker-compose stop
```

Stop and remove all containers (preserves volumes):

```bash
docker-compose down
```

Stop, remove containers, and delete all data:

```bash
docker-compose down -v
```

**WARNING**: The `-v` flag will delete all database data, including users and approvals!

## 🧹 Cleanup

Remove all stopped containers, unused networks, and dangling images:

```bash
docker system prune -a
```

## 📈 Monitoring & Observability

### Prometheus Metrics

Access Prometheus: http://localhost:9090

Available metrics:
- LLM Service: http://localhost:8000/metrics
- Application metrics from all services

### Grafana Dashboards

Access Grafana: http://localhost:3002
- Username: `admin`
- Password: `admin`

Pre-configured dashboards:
- System Overview
- LLM Service Metrics
- Database Performance
- RabbitMQ Queue Status

### RabbitMQ Management

Access RabbitMQ Management: http://localhost:15672
- Username: `devops`
- Password: `devops123`

Monitor:
- Queue lengths
- Message rates
- Connections
- Consumer count

### Qdrant Dashboard

Access Qdrant: http://localhost:6333/dashboard

Monitor:
- Collection statistics
- Vector count
- Search performance

## 🔐 Security Considerations

### For Development:
- Default credentials are provided in `.env`
- SMTP and Slack integrations are optional
- HTTPS is configured but uses self-signed certificates

### For Production:
- **CHANGE ALL DEFAULT PASSWORDS** in `.env`
- Obtain valid SSL/TLS certificates
- Use secrets management (e.g., HashiCorp Vault, AWS Secrets Manager)
- Enable authentication for all services
- Configure firewall rules
- Use environment-specific `.env` files

## 📚 Additional Resources

- **API Documentation**: http://localhost:8000/docs (Swagger UI)
- **Project Architecture**: See `ARCHITECTURE.md`
- **Deployment Guide**: See `DEPLOYMENT.md`
- **Troubleshooting**: See `TROUBLESHOOTING.md`

## 🎉 Success Indicators

You'll know everything is working when:

1. ✅ All 9+ containers show as "Up" in `docker-compose ps`
2. ✅ Health check endpoints return `{"status": "healthy"}`
3. ✅ You can login to the frontend at http://localhost:3001
4. ✅ LLM chat completion works (with valid API keys)
5. ✅ RabbitMQ management UI shows active queues
6. ✅ Database contains test users and sample data
7. ✅ Worker service processes tasks from RabbitMQ
8. ✅ WebSocket notifications work in the frontend

## 🆘 Getting Help

If you encounter issues:

1. Check service logs: `docker-compose logs [service-name]`
2. Verify configuration: `docker-compose config`
3. Restart problematic service: `docker-compose restart [service-name]`
4. Review environment variables in `.env`
5. Consult project documentation
6. Check GitHub issues or create a new one

---

**Last Updated**: 2025-10-18
**Version**: 1.0.0
**Status**: Phase 1 Complete - All services implemented and ready for testing
