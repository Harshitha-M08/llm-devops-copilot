# Getting Started with AI-Driven Hybrid Kubernetes System

## Table of Contents
- [Quick Start](#quick-start)
- [Understanding the Project](#understanding-the-project)
- [Running the Project](#running-the-project)
- [Architecture Overview](#architecture-overview)
- [Development Workflow](#development-workflow)
- [Testing](#testing)
- [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)
- [Future Enhancements](#future-enhancements)

---

## Quick Start

### Prerequisites Checklist

Before running this project, ensure you have:

**Required:**
- [ ] Docker Desktop (v20.10+) and Docker Compose
- [ ] Git (v2.30+)
- [ ] Python 3.11+
- [ ] Node.js 20+ and npm 9+
- [ ] kubectl (v1.24+) - for Kubernetes deployment
- [ ] Helm 3.x - for Kubernetes deployment

**For Production:**
- [ ] Kubernetes cluster (AWS EKS, GCP GKE, or on-premises)
- [ ] OpenAI API key (or Google Gemini API key)
- [ ] SMTP credentials (for email notifications)

**Optional:**
- [ ] Qdrant Cloud account (free tier) or self-host with Docker
- [ ] Slack webhook URL (for notifications)
- [ ] GitHub account (for CI/CD)

---

## Understanding the Project

### What This Project Does

This is a **production-ready AI microservices platform** that provides:

1. **LLM Integration** - Chat with OpenAI GPT-4 or Google Gemini
2. **RAG (Retrieval-Augmented Generation)** - AI answers enriched with your document knowledge base
3. **Human-in-the-Loop Approvals** - Web dashboard for approving critical AI actions
4. **Async Task Processing** - Background workers for long-running AI tasks
5. **Real-time Notifications** - WebSocket updates and email/Slack alerts
6. **Enterprise Monitoring** - Prometheus, Grafana, and ELK stack

### Project Structure - A Guided Tour

```
devops/
├── 📖 README.md                    ← Start here: Project overview
├── 📘 GETTING_STARTED.md           ← You are here!
│
├── 🎯 services/                    ← The microservices (your application)
│   ├── llm-service/                ← AI/LLM API (Python FastAPI)
│   │   ├── app/
│   │   │   ├── main.py             ← REST API endpoints
│   │   │   ├── llm_client.py       ← OpenAI/Gemini integration
│   │   │   └── rag_pipeline.py     ← RAG with vector search
│   │   ├── tests/                  ← Unit tests
│   │   └── Dockerfile              ← Container image
│   │
│   ├── worker-service/             ← Background task processor
│   │   ├── app/
│   │   │   ├── main.py             ← Worker entry point
│   │   │   ├── consumers.py        ← RabbitMQ message consumer
│   │   │   └── tasks.py            ← Task handlers
│   │   └── Dockerfile
│   │
│   ├── approval-dashboard/         ← Human approval interface
│   │   ├── backend/                ← Node.js/Express API
│   │   │   ├── src/server.js       ← API server
│   │   │   └── src/routes/         ← REST endpoints
│   │   └── frontend/               ← React UI
│   │       ├── src/pages/          ← Dashboard pages
│   │       └── src/components/     ← UI components
│   │
│   └── vector-store/               ← Qdrant configuration
│       └── config/                 ← Vector DB setup
│
├── ☸️ k8s/                         ← Kubernetes manifests
│   ├── base/                       ← Base deployments/services
│   └── overlays/                   ← Environment configs (dev/staging/prod)
│
├── ⚙️ infrastructure/              ← Infrastructure as Code
│   ├── helm-charts/                ← Helm charts for deployment
│   └── terraform/                  ← Cloud infrastructure (optional)
│
├── 🚀 ci-cd/                       ← CI/CD pipelines
│   ├── .github/workflows/          ← GitHub Actions
│   └── scripts/                    ← Build/deploy scripts
│
├── 📊 monitoring/                  ← Observability stack
│   ├── prometheus/                 ← Metrics collection
│   ├── grafana/                    ← Dashboards
│   └── elk/                        ← Centralized logging
│
├── 📚 docs/                        ← Comprehensive documentation
│   ├── ARCHITECTURE.md             ← System design
│   ├── API.md                      ← API documentation
│   ├── DEPLOYMENT.md               ← Deploy guide
│   ├── OPERATIONS.md               ← Operations manual
│   └── SECURITY.md                 ← Security guide
│
└── 🛠️ Support Files
    ├── docker-compose.yml          ← Local development stack
    ├── Makefile                    ← Automation commands
    ├── .env.example                ← Environment template
    └── .gitignore
```

### How to Learn This Project

**Recommended Learning Path:**

1. **Start Small** (30 minutes)
   - Read `README.md` for overview
   - Browse `docs/ARCHITECTURE.md` (just the diagrams)
   - Look at `docker-compose.yml` to see all services

2. **Run Locally** (1 hour)
   - Follow "Running Locally" section below
   - Test the LLM Service API with curl
   - Open the Approval Dashboard UI
   - Check Grafana dashboards

3. **Understand Code** (2-3 hours)
   - Read `services/llm-service/app/main.py` - see the REST API
   - Read `services/worker-service/app/tasks.py` - see task processing
   - Read `services/approval-dashboard/backend/src/server.js` - see the backend
   - Browse the React frontend components

4. **Deep Dive** (1-2 days)
   - Read full documentation in `docs/`
   - Explore Kubernetes manifests in `k8s/`
   - Study CI/CD pipelines in `ci-cd/`
   - Review monitoring setup in `monitoring/`

5. **Deploy** (2-4 hours)
   - Deploy to local Kubernetes (minikube/kind)
   - Then deploy to cloud Kubernetes
   - Set up CI/CD pipeline

---

## Running the Project

### Option 1: Local Development with Docker Compose (Recommended for Learning)

This runs all services on your local machine using Docker.

#### Step 1: Clone and Setup

```bash
# Navigate to the devops directory
cd devops

# Copy environment template
cp .env.example .env

# Edit .env with your API keys
# At minimum, set:
# - OPENAI_API_KEY=sk-your-key-here
# - Or GEMINI_API_KEY=your-key-here
```

**Windows Users:**
```powershell
copy .env.example .env
notepad .env
```

#### Step 2: Configure Environment Variables

Open `.env` and set these critical values:

```bash
# LLM Provider (choose one or both)
OPENAI_API_KEY=sk-your-actual-key-here
GEMINI_API_KEY=your-gemini-key-here

# Vector Store (choose one)
# Option A: Use Qdrant Cloud (free tier)
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-qdrant-api-key

# Option B: Use local Qdrant (already in docker-compose)
# QDRANT_URL=http://qdrant:6333
# QDRANT_API_KEY=  # Leave empty for local

# Email (optional - for notifications)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password

# Slack (optional)
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

#### Step 3: Start All Services

```bash
# Using Make (recommended)
make start

# Or using Docker Compose directly
docker-compose up -d

# Check service health
make health

# View logs
make logs
```

#### Step 4: Verify Services are Running

```bash
# Check all containers
docker-compose ps

# You should see these services running:
# ✓ postgres (port 5432)
# ✓ redis (port 6379)
# ✓ rabbitmq (port 5672, 15672)
# ✓ qdrant (port 6333)
# ✓ llm-service (port 8000)
# ✓ worker-service
# ✓ approval-backend (port 3000)
# ✓ approval-frontend (port 3001)
# ✓ nginx (port 80)
# ✓ prometheus (port 9090)
# ✓ grafana (port 3002)
```

#### Step 5: Access the Services

**Web Interfaces:**
- **Approval Dashboard UI:** http://localhost:3001
- **LLM Service API Docs:** http://localhost:8000/docs
- **RabbitMQ Management:** http://localhost:15672 (guest/guest)
- **Grafana Dashboards:** http://localhost:3002 (admin/admin)
- **Prometheus:** http://localhost:9090
- **Qdrant UI:** http://localhost:6333/dashboard

**Default Login Credentials:**
- **Approval Dashboard:**
  - Email: `admin@approvaldashboard.com`
  - Password: `Admin@123`

#### Step 6: Test the System

**Test 1: LLM Service API**
```bash
# Health check
curl http://localhost:8000/health

# Chat completion
curl -X POST http://localhost:8000/api/v1/chat \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [
      {"role": "user", "content": "What is Kubernetes?"}
    ]
  }'

# Ingest a document for RAG
curl -X POST http://localhost:8000/api/v1/documents/ingest \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Kubernetes is an open-source container orchestration platform...",
    "metadata": {"source": "documentation"}
  }'

# RAG query
curl -X POST http://localhost:8000/api/v1/rag/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is Kubernetes?"
  }'
```

**Test 2: Approval Dashboard**
1. Open http://localhost:3001
2. Login with admin credentials
3. Create a new approval request
4. Approve or reject it
5. Check real-time notifications

**Test 3: Worker Service**
```bash
# Submit an async task
curl -X POST http://localhost:8000/api/v1/tasks/submit \
  -H "Content-Type: application/json" \
  -d '{
    "task_type": "process_llm_request",
    "payload": {
      "prompt": "Explain microservices"
    }
  }'

# Check worker logs
docker-compose logs -f worker-service
```

#### Step 7: Stop Services

```bash
# Stop all services
make stop

# Or stop and remove volumes (clean slate)
make down
```

---

### Option 2: Running Individual Services (Development)

For active development on a specific service:

**LLM Service:**
```bash
cd services/llm-service

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export OPENAI_API_KEY=sk-your-key
export QDRANT_URL=http://localhost:6333

# Run the service
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Or use Make
make dev-llm
```

**Approval Dashboard Backend:**
```bash
cd services/approval-dashboard/backend

# Install dependencies
npm install

# Set environment variables
cp .env.example .env
# Edit .env with your values

# Run the service
npm run dev

# Or use Make
make dev-backend
```

**Approval Dashboard Frontend:**
```bash
cd services/approval-dashboard/frontend

# Install dependencies
npm install

# Set environment variables
cp .env.example .env
# Edit REACT_APP_API_URL=http://localhost:3000/api

# Run the service
npm start

# Or use Make
make dev-frontend
```

**Worker Service:**
```bash
cd services/worker-service

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export RABBITMQ_URL=amqp://guest:guest@localhost:5672

# Run the service
python -m app.main

# Or use Make
make dev-worker
```

---

### Option 3: Kubernetes Deployment (Production)

#### Prerequisites
- Kubernetes cluster (minikube for local, or AWS EKS/GCP GKE for cloud)
- kubectl configured
- Helm 3 installed

#### Deploy Using Kubectl + Kustomize

```bash
# Create namespace
kubectl create namespace ai-system

# Create secrets (update with your actual values)
kubectl create secret generic ai-secrets \
  --from-literal=openai-api-key=sk-your-key \
  --from-literal=gemini-api-key=your-key \
  --from-literal=qdrant-api-key=your-key \
  --from-literal=postgres-password=changeme \
  --from-literal=rabbitmq-password=changeme \
  --from-literal=redis-password=changeme \
  -n ai-system

# Deploy to development
kubectl apply -k k8s/overlays/dev/

# Check deployment status
kubectl get pods -n ai-system
kubectl get svc -n ai-system
kubectl get ingress -n ai-system

# View logs
kubectl logs -f deployment/llm-service -n ai-system

# Or use Make
make k8s-deploy
```

#### Deploy Using Helm

```bash
# Navigate to helm chart
cd infrastructure/helm-charts

# Install for development
helm install ai-system ./ai-system \
  -f ./ai-system/values-dev.yaml \
  --namespace ai-system \
  --create-namespace

# Install for staging
helm install ai-system ./ai-system \
  -f ./ai-system/values-staging.yaml \
  --namespace ai-system-staging \
  --create-namespace

# Install for production
helm install ai-system ./ai-system \
  -f ./ai-system/values-prod.yaml \
  --namespace ai-system-prod \
  --create-namespace

# Check status
helm status ai-system -n ai-system

# Upgrade
helm upgrade ai-system ./ai-system \
  -f ./ai-system/values-dev.yaml \
  -n ai-system

# Rollback
helm rollback ai-system -n ai-system
```

#### Access Services in Kubernetes

```bash
# Port-forward to access services locally
kubectl port-forward svc/llm-service 8000:8000 -n ai-system
kubectl port-forward svc/approval-frontend 3001:80 -n ai-system
kubectl port-forward svc/grafana 3002:80 -n ai-system

# Or use Ingress (if configured)
kubectl get ingress -n ai-system
# Access via: http://your-domain.com
```

---

## Architecture Overview

### System Components

```
┌─────────────────────────────────────────────────────────────────┐
│                          User / Client                           │
└───────────────────────────────┬─────────────────────────────────┘
                                │
                    ┌───────────▼───────────┐
                    │   Ingress / Gateway   │
                    │   (NGINX / Traefik)   │
                    └───────────┬───────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
┌───────▼────────┐    ┌────────▼────────┐    ┌────────▼─────────┐
│  LLM Service   │    │  Approval UI    │    │  Approval API    │
│  (FastAPI)     │    │  (React)        │    │  (Express)       │
│  Port: 8000    │    │  Port: 3001     │    │  Port: 3000      │
└───────┬────────┘    └─────────────────┘    └────────┬─────────┘
        │                                               │
        │        ┌──────────────────┐                  │
        └───────►│   RabbitMQ       │◄─────────────────┘
                 │   Message Queue  │
                 │   Port: 5672     │
                 └────────┬─────────┘
                          │
                  ┌───────▼────────┐
                  │ Worker Service │
                  │   (Python)     │
                  └───────┬────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
┌───────▼────────┐ ┌─────▼──────┐ ┌────────▼─────────┐
│  PostgreSQL    │ │   Redis    │ │  Qdrant Vector   │
│  (Database)    │ │  (Cache)   │ │  (Embeddings)    │
│  Port: 5432    │ │ Port: 6379 │ │  Port: 6333      │
└────────────────┘ └────────────┘ └──────────────────┘
```

### Request Flow Examples

**1. Simple Chat Request:**
```
User → Ingress → LLM Service → OpenAI/Gemini API → Response
```

**2. RAG Query (with context):**
```
User → LLM Service → Qdrant (retrieve docs) → LLM Service (augment prompt)
     → OpenAI/Gemini API → Response with context
```

**3. Async Task with Approval:**
```
User → LLM Service → RabbitMQ → Worker Service → Create Approval Request
     → PostgreSQL → WebSocket → Approval Dashboard → Human Approves
     → Worker Service continues → Execute Task → Send Notification
```

### Technology Decisions

| Component | Technology | Why? |
|-----------|------------|------|
| **LLM API** | FastAPI | Fast, async, auto-docs, Python ecosystem |
| **Worker** | Python + Celery/RQ | Async task processing, retry logic |
| **Backend API** | Node.js/Express | Fast, mature, great WebSocket support |
| **Frontend** | React + Material-UI | Modern, component-based, rich UI library |
| **Database** | PostgreSQL | Reliable, ACID, great for structured data |
| **Cache** | Redis | Fast, widely used, good for sessions |
| **Queue** | RabbitMQ | Mature, reliable, flexible routing |
| **Vector DB** | Qdrant | Fast, free tier, easy to use |
| **Orchestration** | Kubernetes | Industry standard, auto-scaling, self-healing |
| **Monitoring** | Prometheus + Grafana | De-facto standard, rich ecosystem |

---

## Development Workflow

### Making Changes

1. **Create a feature branch**
```bash
git checkout -b feature/your-feature-name
```

2. **Make your changes**
- Edit code in `services/`
- Update tests
- Update documentation

3. **Test locally**
```bash
# Run tests
make test

# Run linting
make lint

# Start services
make start
```

4. **Commit with conventional commits**
```bash
git add .
git commit -m "feat: add new LLM endpoint for summarization"
```

5. **Push and create PR**
```bash
git push origin feature/your-feature-name
# Create PR on GitHub
```

### Code Style

**Python:**
- Follow PEP 8
- Use Black for formatting: `make format-python`
- Type hints required
- Docstrings for all functions

**JavaScript/TypeScript:**
- Use Prettier: `make format-node`
- ESLint rules enforced
- JSDoc comments

---

## Testing

### Run All Tests

```bash
# All tests
make test

# Python tests only
make test-python

# Node tests only
make test-node

# With coverage
make test-coverage

# Integration tests
make test-integration

# E2E tests
make test-e2e
```

### Test Individual Services

**LLM Service:**
```bash
cd services/llm-service
pytest tests/ -v
pytest tests/test_llm_client.py::test_chat_completion -v
```

**Worker Service:**
```bash
cd services/worker-service
pytest tests/ -v --cov=app
```

**Approval Backend:**
```bash
cd services/approval-dashboard/backend
npm test
npm run test:watch
```

**Approval Frontend:**
```bash
cd services/approval-dashboard/frontend
npm test
npm run test:coverage
```

---

## Deployment

### Local Kubernetes (minikube)

```bash
# Start minikube
minikube start --cpus 4 --memory 8192

# Enable addons
minikube addons enable ingress
minikube addons enable metrics-server

# Deploy
kubectl apply -k k8s/overlays/dev/

# Access via minikube
minikube service llm-service -n ai-system
minikube service approval-frontend -n ai-system
```

### AWS EKS

See `docs/DEPLOYMENT.md` for full AWS deployment guide.

```bash
# Create EKS cluster (if not exists)
# Use Terraform: cd infrastructure/terraform/aws
# terraform init && terraform apply

# Configure kubectl
aws eks update-kubeconfig --name ai-system-cluster --region us-east-1

# Deploy with Helm
helm install ai-system ./infrastructure/helm-charts/ai-system \
  -f ./infrastructure/helm-charts/ai-system/values-prod.yaml \
  --namespace ai-system-prod \
  --create-namespace

# Check status
kubectl get all -n ai-system-prod
```

### GCP GKE

```bash
# Create GKE cluster (if not exists)
gcloud container clusters create ai-system-cluster \
  --region us-central1 \
  --num-nodes 3

# Get credentials
gcloud container clusters get-credentials ai-system-cluster --region us-central1

# Deploy
helm install ai-system ./infrastructure/helm-charts/ai-system \
  -f ./infrastructure/helm-charts/ai-system/values-prod.yaml \
  --namespace ai-system-prod \
  --create-namespace
```

---

## Troubleshooting

### Common Issues

**1. Services won't start in Docker Compose**
```bash
# Check logs
docker-compose logs -f service-name

# Common causes:
# - Missing .env file: cp .env.example .env
# - Port conflicts: Check if ports are already in use
# - Docker resources: Increase Docker memory (8GB recommended)
```

**2. LLM Service returns 500 errors**
```bash
# Check if API keys are set
docker-compose exec llm-service env | grep API_KEY

# Check Qdrant connection
curl http://localhost:6333/collections

# View detailed logs
docker-compose logs -f llm-service
```

**3. Approval Dashboard not loading**
```bash
# Check if backend is running
curl http://localhost:3000/api/health

# Check frontend build
docker-compose logs -f approval-frontend

# Check browser console for errors
```

**4. Worker not processing tasks**
```bash
# Check RabbitMQ connection
curl http://localhost:15672 (guest/guest)

# Check worker logs
docker-compose logs -f worker-service

# Check queue has messages
# Go to http://localhost:15672 → Queues
```

**5. Kubernetes pods not starting**
```bash
# Check pod status
kubectl get pods -n ai-system

# Describe pod for details
kubectl describe pod <pod-name> -n ai-system

# Check logs
kubectl logs <pod-name> -n ai-system

# Common issues:
# - Image pull errors: Check image registry
# - Resource limits: Check cluster resources
# - Config/secrets: Verify secrets exist
```

### Debug Commands

```bash
# Check service health
make health

# View all logs
make logs

# Shell into containers
docker-compose exec llm-service bash
docker-compose exec postgres psql -U postgres

# Database access
docker-compose exec postgres psql -U postgres -d ai_system
# \dt  - list tables
# SELECT * FROM approvals;

# RabbitMQ check
docker-compose exec rabbitmq rabbitmqctl list_queues

# Check network connectivity
docker-compose exec llm-service ping postgres
```

---

## Future Enhancements

### Phase 1: Core Improvements (Next 1-2 Months)

**1. Enhanced LLM Features**
- [ ] Add support for Claude, Llama, and other models
- [ ] Implement conversation memory and context management
- [ ] Add fine-tuning capabilities
- [ ] Support for image generation (DALL-E, Stable Diffusion)
- [ ] Multi-modal inputs (images + text)
- [ ] Token cost tracking and budgeting per user

**2. Advanced RAG**
- [ ] Multi-vector store support (Pinecone, Weaviate, Chroma)
- [ ] Hybrid search (semantic + keyword)
- [ ] Document chunking strategies (semantic, hierarchical)
- [ ] Re-ranking models for better retrieval
- [ ] Citation and source tracking
- [ ] Automatic document ingestion from various sources (PDF, DOCX, web scraping)

**3. Approval Workflow Enhancements**
- [ ] Multi-level approval chains (sequential, parallel)
- [ ] Conditional approval rules engine
- [ ] Approval templates and workflows
- [ ] Time-based auto-approval/rejection
- [ ] Mobile app for approvals (React Native)
- [ ] Approval analytics dashboard
- [ ] Integration with JIRA, ServiceNow for ticketing

**4. Security Hardening**
- [ ] OAuth 2.0 / OIDC integration (Auth0, Okta, Keycloak)
- [ ] Multi-factor authentication (MFA)
- [ ] API rate limiting per user/team
- [ ] Data encryption at rest
- [ ] Audit logging for all actions
- [ ] GDPR compliance features (data export, deletion)
- [ ] SOC 2 compliance documentation

### Phase 2: Scale & Performance (Months 3-4)

**5. Performance Optimization**
- [ ] Response caching layer
- [ ] LLM response streaming optimization
- [ ] Database query optimization and indexing
- [ ] CDN for frontend assets
- [ ] API response compression
- [ ] Connection pooling tuning
- [ ] Load testing and benchmarking

**6. Scalability**
- [ ] Multi-region deployment support
- [ ] Database read replicas
- [ ] Horizontal pod autoscaling refinement
- [ ] Cluster autoscaling (KEDA for event-driven)
- [ ] Distributed tracing (Jaeger, OpenTelemetry)
- [ ] Service mesh implementation (Istio, Linkerd)

**7. Observability**
- [ ] Custom Grafana dashboards for business metrics
- [ ] APM integration (New Relic, Datadog, Dynatrace)
- [ ] User behavior analytics
- [ ] Cost monitoring and optimization
- [ ] Anomaly detection alerts
- [ ] SLO/SLA tracking

### Phase 3: Advanced Features (Months 5-6)

**8. AI Agent Framework**
- [ ] LangGraph integration for complex workflows
- [ ] Tool calling and function execution
- [ ] Multi-agent collaboration
- [ ] Autonomous agent with guardrails
- [ ] Agent memory and personalization
- [ ] Agent performance evaluation

**9. Enterprise Features**
- [ ] Multi-tenancy support (org/team isolation)
- [ ] Usage quotas and billing
- [ ] White-label customization
- [ ] SSO integration (SAML, LDAP)
- [ ] Advanced RBAC with custom permissions
- [ ] Compliance certifications (HIPAA, FedRAMP)

**10. Developer Experience**
- [ ] SDK for Python, JavaScript, Go, Java
- [ ] CLI tool for management
- [ ] Postman/OpenAPI collections
- [ ] Interactive API playground
- [ ] Code generation from natural language
- [ ] Plugin system for extensions

**11. Integration Ecosystem**
- [ ] Zapier/Make integration
- [ ] Slack bot for chat interface
- [ ] Microsoft Teams integration
- [ ] Google Workspace integration
- [ ] Salesforce connector
- [ ] Webhook system for custom integrations

### Phase 4: Innovation (Months 7+)

**12. Advanced AI Capabilities**
- [ ] Model routing based on task complexity
- [ ] Automatic prompt optimization
- [ ] AI model comparison and benchmarking
- [ ] Custom model deployment (self-hosted LLMs)
- [ ] Federated learning support
- [ ] AI explainability features

**13. Data & Analytics**
- [ ] Business intelligence dashboard
- [ ] Predictive analytics for approval patterns
- [ ] A/B testing framework for prompts
- [ ] User sentiment analysis
- [ ] ROI tracking and reporting

**14. Edge Computing**
- [ ] Edge deployment for low-latency regions
- [ ] Offline mode support
- [ ] Progressive Web App (PWA)
- [ ] Local LLM inference options

### Technical Debt & Maintenance

**Continuous Improvements:**
- [ ] Dependency updates (automated with Dependabot)
- [ ] Security patching
- [ ] Code refactoring and cleanup
- [ ] Test coverage improvements (target: 90%+)
- [ ] Documentation updates
- [ ] Performance profiling and optimization
- [ ] Database migration strategies
- [ ] Disaster recovery drills

---

## Community & Support

### Getting Help

- **Documentation:** Start with `docs/` folder
- **Issues:** Report bugs and request features on GitHub
- **Discussions:** Ask questions in GitHub Discussions
- **Email:** support@yourproject.com (configure your own)

### Contributing

See `CONTRIBUTING.md` for detailed guidelines.

Quick contribution checklist:
1. Fork the repository
2. Create a feature branch
3. Make your changes with tests
4. Ensure all tests pass (`make test`)
5. Submit a pull request

### License

This project is licensed under the MIT License - see `LICENSE` file.

---

## Quick Reference

### Essential Commands

```bash
# Start everything
make start

# Stop everything
make stop

# Run tests
make test

# Deploy to Kubernetes
make k8s-deploy

# View logs
make logs

# Clean everything
make clean-all

# Health check
make health

# Open monitoring
make monitor
```

### Important URLs (Local Development)

| Service | URL | Credentials |
|---------|-----|-------------|
| Approval Dashboard | http://localhost:3001 | admin@approvaldashboard.com / Admin@123 |
| LLM Service API | http://localhost:8000/docs | - |
| RabbitMQ Management | http://localhost:15672 | guest / guest |
| Grafana | http://localhost:3002 | admin / admin |
| Prometheus | http://localhost:9090 | - |
| Qdrant Dashboard | http://localhost:6333/dashboard | - |

### File Locations

| Purpose | Location |
|---------|----------|
| Service code | `services/<service-name>/app/` |
| Tests | `services/<service-name>/tests/` |
| Kubernetes manifests | `k8s/` |
| Helm charts | `infrastructure/helm-charts/` |
| CI/CD pipelines | `ci-cd/.github/workflows/` |
| Documentation | `docs/` |
| Environment config | `.env` (create from `.env.example`) |

---

## Summary

This project provides a **complete, production-ready AI platform** with:
- ✅ Multiple LLM providers (OpenAI, Gemini)
- ✅ RAG for knowledge-enhanced responses
- ✅ Human-in-the-loop approval workflows
- ✅ Async task processing
- ✅ Real-time notifications
- ✅ Enterprise monitoring and logging
- ✅ Kubernetes-native deployment
- ✅ Full CI/CD automation
- ✅ Comprehensive security

**Start with local Docker Compose**, understand the architecture, then deploy to Kubernetes for production.

For detailed information, see:
- `docs/ARCHITECTURE.md` - System design
- `docs/DEPLOYMENT.md` - Production deployment
- `docs/API.md` - API documentation
- `docs/OPERATIONS.md` - Day-2 operations

**Happy coding! 🚀**
