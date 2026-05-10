# GitHub Codespaces Setup Guide

## 🚀 Quick Start

This project is fully configured for [SERVICE_PROVIDER] Codespaces, providing a complete cloud development environment with all dependencies pre-installed.

### What is [SERVICE_PROVIDER] Codespaces?

[SERVICE_PROVIDER] Codespaces is a cloud-based development environment that allows you to code from anywhere using VS Code in your browser. All dependencies, tools, and services are pre-configured and ready to use.

---

## 📋 Prerequisites

1. **[SERVICE_PROVIDER] Account**: You need a [SERVICE_PROVIDER] account with Codespaces access
2. **API Keys**: Obtain the following API keys:
   - OpenAI API Key: https://platform.openai.com/api-keys
   - Anthropic API Key (optional): https://console.anthropic.com/

---

## 🎯 Step-by-Step Setup

### Step 1: Fork or Clone the Repository

If you don't already have the repository:

1. Go to your [SERVICE_PROVIDER] repository
2. Click the **"Code"** button
3. Select **"Codespaces"** tab
4. Click **"Create codespace on main"**

### Step 2: Wait for Codespace to Build

The first time you create a Codespace, it will:
- Build the devcontainer (~5-10 minutes)
- Install Python 3.12 and Node.js 20
- Install all project dependencies
- Start database services (PostgreSQL, Redis, RabbitMQ, Qdrant)
- Run the setup script automatically

You'll see progress in the terminal.

### Step 3: Configure API Keys

Once the Codespace is ready:

1. **Create/Update .env file**:
   ```bash
   cd /workspace/devops
   # Edit .env file
   nano .env
   ```

2. **Add your API keys**:
   ```bash
   # Update these lines in .env:
   OPENAI_API_KEY=sk-proj-YOUR_ACTUAL_OPENAI_KEY_HERE
   ANTHROPIC_API_KEY=sk-ant-YOUR_ACTUAL_ANTHROPIC_KEY_HERE  # Optional
   ```

3. **Save and exit** (`Ctrl+O`, `Enter`, `Ctrl+X` in nano)

### Step 4: Start All Services

```bash
cd /workspace/devops
docker-compose up -d
```

This will start:
- ✅ LLM Service (Python FastAPI)
- ✅ Worker Service (Python)
- ✅ Approval Backend (Node.js/Express)
- ✅ Approval Frontend (React)
- ✅ NGINX (Reverse Proxy)

### Step 5: Verify Services

Check all services are running:

```bash
docker-compose ps
```

Test the LLM Service:

```bash
cd /workspace/devops/services/llm-service
python test_complete.py
```

Expected output:
```
[PASS] LLM Client Init
[PASS] Chat Completion
[PASS] Embeddings
[PASS] RAG Pipeline  # Will pass now that Qdrant is running!
```

---

## 🌐 Accessing Services

[SERVICE_PROVIDER] Codespaces automatically forwards ports. You can access services through:

### Web Interfaces

| Service | Port | URL | Credentials |
|---------|------|-----|-------------|
| **Approval Frontend** | 3001 | Auto-forwarded | See below |
| **Approval Backend API** | 3000 | Auto-forwarded | N/A |
| **LLM Service API Docs** | 8000 | Auto-forwarded | N/A |
| **RabbitMQ Management** | 15672 | Auto-forwarded | devops / devops123 |
| **Grafana** | 3002 | Auto-forwarded | admin / admin |
| **Prometheus** | 9090 | Auto-forwarded | N/A |

### Default Application Credentials

For the Approval Dashboard (http://localhost:3001):

| Role | Email | Password |
|------|-------|----------|
| Admin | admin@devops.local | Admin@123 |
| Approver | approver@devops.local | Approver@123 |
| Developer | developer@devops.local | Developer@123 |
| Viewer | viewer@devops.local | Viewer@123 |

### Database Services

These run internally and can be accessed from within the Codespace:

| Service | Host | Port | Credentials |
|---------|------|------|-------------|
| PostgreSQL | localhost | 5432 | devops / devops123 |
| Redis | localhost | 6379 | Password: redis123 |
| RabbitMQ | localhost | 5672 | devops / devops123 |
| Qdrant | localhost | 6333 | N/A |

---

## 🔧 Development Workflow

### Project Structure

```
/workspace/
└── devops/
    ├── .devcontainer/          # Codespaces configuration
    │   ├── devcontainer.json   # Main configuration
    │   ├── Dockerfile          # Base image
    │   ├── docker-compose.yml  # Services for Codespaces
    │   └── setup.sh           # Auto-setup script
    ├── services/
    │   ├── llm-service/       # Python FastAPI
    │   ├── worker-service/    # Python worker
    │   └── approval-dashboard/
    │       ├── backend/       # Node.js Express
    │       └── frontend/      # React
    ├── infrastructure/        # Database & queue config
    ├── kubernetes/           # K8s manifests
    └── terraform/            # Infrastructure as Code
```

### Running Services Individually

#### LLM Service (Python FastAPI)

```bash
cd /workspace/devops/services/llm-service
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Access at: http://localhost:8000/docs

#### Worker Service

```bash
cd /workspace/devops/services/worker-service
python app/main.py
```

#### Approval Backend

```bash
cd /workspace/devops/services/approval-dashboard/backend
npm run dev
```

#### Approval Frontend

```bash
cd /workspace/devops/services/approval-dashboard/frontend
npm start
```

Access at: http://localhost:3001

### Testing

#### Test LLM Service

```bash
cd /workspace/devops/services/llm-service
python test_complete.py
```

#### Test with curl

```bash
# Health check
curl http://localhost:8000/health

# Chat completion
curl -X POST http://localhost:8000/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "Hello!"}
    ],
    "provider": "openai"
  }'

# Document ingestion
curl -X POST http://localhost:8000/api/v1/documents/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Kubernetes is a container orchestration platform.",
    "metadata": {"source": "docs"}
  }'

# RAG query
curl -X POST http://localhost:8000/api/v1/rag/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Kubernetes?"
  }'
```

---

## 🐛 Troubleshooting

### Services Not Starting

**Problem**: Docker services fail to start

**Solution**:
```bash
# Check service logs
docker-compose logs [service-name]

# Restart all services
docker-compose restart

# Rebuild if needed
docker-compose down
docker-compose up -d --build
```

### Database Connection Errors

**Problem**: Cannot connect to PostgreSQL

**Solution**:
```bash
# Check if PostgreSQL is running
docker-compose ps postgres

# Restart PostgreSQL
docker-compose restart postgres

# Test connection
PGPASSWORD=devops123 psql -h localhost -U devops -d devops_db -c "SELECT version();"
```

### API Key Issues

**Problem**: LLM Service returns 401 errors

**Solution**:
1. Verify API key format:
   - OpenAI: Must start with `sk-proj-` or `sk-`
   - Anthropic: Must start with `sk-ant-`
2. Check `.env` file:
   ```bash
   cat .env | grep API_KEY
   ```
3. Restart LLM service:
   ```bash
   docker-compose restart llm-service
   ```

### Port Already in Use

**Problem**: Error "port already allocated"

**Solution**:
```bash
# Find and stop conflicting processes
sudo lsof -i :[PORT_NUMBER]
sudo kill -9 [PID]

# Or change ports in docker-compose.yml
```

### Codespace Running Slow

**Problem**: Performance issues

**Solution**:
1. Upgrade Codespace machine type:
   - Click Codespace name in bottom left
   - Select "Change Machine Type"
   - Choose 4-core or 8-core
2. Rebuild container:
   - Press `F1` or `Ctrl+Shift+P`
   - Type "Rebuild Container"
   - Select "Codespaces: Rebuild Container"

---

## 🔐 Security Best Practices

### Protecting Secrets

1. **Never commit .env files**
   - Already in .gitignore
   - Use [SERVICE_PROVIDER] Secrets for production

2. **Use [SERVICE_PROVIDER] Codespaces Secrets**
   - Go to: Settings → Codespaces → Secrets
   - Add: `OPENAI_API_KEY` and `ANTHROPIC_API_KEY`
   - These will be auto-injected into your Codespace

3. **Rotate Keys Regularly**
   - Generate new API keys monthly
   - Update in Codespaces Secrets

4. **Use Different Keys Per Environment**
   - Development: Separate API keys
   - Staging: Different keys
   - Production: Most secure keys

---

## 📊 Monitoring & Debugging

### View Logs

```bash
# All services
docker-compose logs -f

# Specific service
docker-compose logs -f llm-service
docker-compose logs -f worker-service
docker-compose logs -f approval-backend

# Last 100 lines
docker-compose logs --tail=100 llm-service
```

### Monitor Resources

```bash
# Docker stats
docker stats

# System resources
htop

# Disk usage
df -h
```

### Database Queries

```bash
# Connect to PostgreSQL
PGPASSWORD=devops123 psql -h localhost -U devops -d devops_db

# Common queries:
# List tables
\dt

# View users
SELECT * FROM users;

# View approvals
SELECT * FROM approvals ORDER BY created_at DESC LIMIT 10;

# Exit
\q
```

### RabbitMQ Monitoring

Access RabbitMQ Management UI:
- URL: http://localhost:15672
- Username: `devops`
- Password: `devops123`

Check:
- Queue lengths
- Message rates
- Consumer status
- Connection health

---

## 🚢 Deployment from Codespaces

### To Kubernetes

```bash
# Set up kubectl (if targeting external cluster)
kubectl config use-context your-cluster

# Deploy
cd /workspace/devops/kubernetes
kubectl apply -f base/

# Verify
kubectl get pods -n devops
kubectl get services -n devops
```

### To Docker Hub

```bash
# Build and push images
cd /workspace/devops

# LLM Service
docker build -t yourusername/llm-service:latest services/llm-service
docker push yourusername/llm-service:latest

# Worker Service
docker build -t yourusername/worker-service:latest services/worker-service
docker push yourusername/worker-service:latest

# Approval Backend
docker build -t yourusername/approval-backend:latest services/approval-dashboard/backend
docker push yourusername/approval-backend:latest
```

---

## 🎓 Learning Resources

### Documentation

- **Architecture**: `/workspace/devops/README.md`
- **Local Setup**: `/workspace/devops/DOCKER-COMPOSE-GUIDE.md`
- **Test Results**: `/workspace/devops/TEST-RESULTS.md`
- **Phase 1 Progress**: `/workspace/devops/UPGRADE-PART1.md`
- **Phase 2 Roadmap**: `/workspace/devops/UPGRADE-PART2.md`
- **Phase 3 Plans**: `/workspace/devops/UPGRADE-PART3.md`

### API Documentation

- **LLM Service**: http://localhost:8000/docs (Swagger UI)
- **LLM Service (Alternative)**: http://localhost:8000/redoc (ReDoc)

### Testing

- **LLM Service Tests**: `python services/llm-service/test_complete.py`
- **Integration Tests**: Coming in Phase 2

---

## 💡 Tips & Tricks

### Faster Development

1. **Use VS Code Extensions** (auto-installed):
   - Python
   - ESLint
   - Prettier
   - Docker
   - Kubernetes

2. **Terminal Shortcuts**:
   ```bash
   # Create aliases (add to ~/.bashrc)
   alias dc='docker-compose'
   alias dps='docker-compose ps'
   alias dlog='docker-compose logs -f'
   alias llm='cd /workspace/devops/services/llm-service'
   alias worker='cd /workspace/devops/services/worker-service'
   alias backend='cd /workspace/devops/services/approval-dashboard/backend'
   alias frontend='cd /workspace/devops/services/approval-dashboard/frontend'
   ```

3. **Hot Reload**:
   - Python: Use `--reload` flag with uvicorn
   - Node.js: Use `npm run dev` (nodemon)
   - React: `npm start` (auto hot-reload)

### Saving Costs

1. **Stop Codespace When Not in Use**
   - Auto-stops after 30 minutes of inactivity
   - Manually stop: Codespaces menu → Stop

2. **Use Prebuilds** (for organization repos)
   - Faster startup times
   - Reduced compute costs

3. **Delete Old Codespaces**
   - Keep only active Codespaces
   - Delete from: https://github.com/codespaces

---

## 🆘 Getting Help

### Common Commands Reference

```bash
# Start all services
cd /workspace/devops && docker-compose up -d

# Stop all services
docker-compose down

# View all services
docker-compose ps

# View logs
docker-compose logs -f [service-name]

# Restart service
docker-compose restart [service-name]

# Test LLM Service
cd /workspace/devops/services/llm-service && python test_complete.py

# Run database migrations
PGPASSWORD=devops123 psql -h localhost -U devops -d devops_db -f infrastructure/database/init/001_initial_schema.sql

# Access PostgreSQL CLI
PGPASSWORD=devops123 psql -h localhost -U devops -d devops_db

# Check [SERVICE_PROVIDER] Codespaces secrets
gh secret list

# View environment variables
printenv | grep -E '(OPENAI|ANTHROPIC|DATABASE|REDIS|RABBITMQ)'
```

### Support Channels

- **Documentation**: Check `/workspace/devops/*.md` files
- **[SERVICE_PROVIDER] Issues**: Create issue in your repository
- **Codespaces Docs**: https://docs.github.com/en/codespaces

---

## ✅ Success Checklist

After setup, verify:

- [ ] Codespace is running
- [ ] All Docker services are up (`docker-compose ps`)
- [ ] PostgreSQL accessible (`psql` connection works)
- [ ] Redis accessible (`redis-cli ping` returns PONG)
- [ ] RabbitMQ Management UI accessible (port 15672)
- [ ] Qdrant accessible (port 6333)
- [ ] API keys configured in `.env`
- [ ] LLM Service tests pass (`python test_complete.py`)
- [ ] Approval Frontend loads (port 3001)
- [ ] Can login to dashboard with default credentials

---

## 🎉 You're Ready!

Your [SERVICE_PROVIDER] Codespace is fully configured and ready for development!

**Next Steps:**
1. ✅ Start building features
2. ✅ Run tests regularly
3. ✅ Monitor service health
4. ✅ Deploy to production (Phase 2)

**Happy Coding! 🚀**

---

**Last Updated**: 2025-10-18
**Version**: 1.0.0
**Status**: Production Ready for [SERVICE_PROVIDER] Codespaces
