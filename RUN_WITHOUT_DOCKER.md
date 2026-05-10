# Running the Project Without Docker

Since Docker is not available on your system, here are your options:

## Option 1: Install Docker Desktop (Recommended)

**Download Docker Desktop for Windows:**
https://www.docker.com/products/docker-desktop/

After installation:
```bash
cd devops
docker-compose up -d
```

This will run the complete stack with all services.

---

## Option 2: Run a Simplified Demo (Current System)

I'll create a minimal demo that shows the LLM Service working without dependencies.

### Quick Demo Setup

```bash
cd devops

# Install LLM Service dependencies
cd services/llm-service
pip install fastapi uvicorn openai pydantic pydantic-settings

# Set your OpenAI API key
$env:OPENAI_API_KEY="your-key-here"  # PowerShell
# or
set OPENAI_API_KEY=your-key-here  # CMD

# Run the LLM service (without RAG/database features)
python -c "
from fastapi import FastAPI
import uvicorn

app = FastAPI()

@app.get('/')
def home():
    return {'message': 'AI LLM Service - Demo Mode', 'status': 'running'}

@app.get('/health')
def health():
    return {'status': 'healthy', 'mode': 'demo'}

uvicorn.run(app, host='0.0.0.0', port=8000)
"
```

Then open: http://localhost:8000

---

## Option 3: Cloud Deployment

Deploy directly to a cloud Kubernetes cluster:

### AWS EKS
```bash
# Install AWS CLI and kubectl
aws eks update-kubeconfig --name your-cluster --region us-east-1

# Deploy with Helm
helm install ai-system ./infrastructure/helm-charts/ai-system
```

### Google Cloud GKE
```bash
# Install gcloud and kubectl
gcloud container clusters get-credentials your-cluster

# Deploy with Helm
helm install ai-system ./infrastructure/helm-charts/ai-system
```

---

## What You Need Docker For

The project requires these infrastructure components:
- ✅ **PostgreSQL** - Database for approvals and users
- ✅ **Redis** - Caching layer
- ✅ **RabbitMQ** - Message queue for async tasks
- ✅ **Qdrant** - Vector database for AI embeddings
- ✅ **All microservices** - LLM, Worker, Approval Backend/Frontend

**Without Docker**, you would need to:
1. Install PostgreSQL server locally
2. Install Redis server locally
3. Install RabbitMQ server locally
4. Install Qdrant locally or use Qdrant Cloud
5. Run each service manually in separate terminals

This is complex and error-prone, which is why Docker is recommended.

---

## Recommended Next Steps

1. **Install Docker Desktop** - Takes 5-10 minutes
2. **Restart your terminal**
3. **Run**: `cd devops && docker-compose up -d`
4. **Access**: http://localhost:3001 (Approval Dashboard)

That's it! Everything will work out of the box.

---

## Alternative: Use GitHub Codespaces

If you can't install Docker locally, use GitHub Codespaces:

1. Push this project to GitHub
2. Open in Codespaces (has Docker pre-installed)
3. Run `docker-compose up -d`
4. Access via port forwarding

---

## Contact

For help with Docker installation or deployment:
- Check `docs/DEPLOYMENT.md` for detailed guides
- See `GETTING_STARTED.md` for full instructions
