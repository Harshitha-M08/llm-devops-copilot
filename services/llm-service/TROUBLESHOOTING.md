# LLM Service Troubleshooting & Fix Documentation

## Issue Summary
The LLM service was failing to start with the error: `ERROR: Error loading ASGI app. Could not import module "main"`

## Root Causes Identified

### 1. Incorrect Uvicorn Module Path
**Problem:** Dockerfile CMD was using `main:app` instead of `app.main:app`
- The application structure has `main.py` inside the `app/` directory
- Uvicorn couldn't find the module at the root level

**Solution:** Updated Dockerfile CMD from:
```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```
To:
```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

### 2. Incorrect Import Statements
**Problem:** `main.py` was using absolute imports instead of relative imports
- Import statements: `from llm_client import ...` and `from rag_pipeline import ...`
- These failed because Python couldn't resolve the module paths correctly

**Solution:** Changed to relative imports in `app/main.py`:
```python
from .llm_client import LLMClient, LLMConfig, LLMProvider
from .rag_pipeline import RAGPipeline, RAGConfig
```

### 3. RAG Pipeline Initialization at Module Import Time
**Problem:** RAG pipeline was initializing during module import (before uvicorn starts)
- Line 53 in main.py: `rag_pipeline = RAGPipeline(llm_client, rag_config)`
- This tried to connect to Qdrant immediately, causing connection failures
- Service would crash before even starting the HTTP server

**Solution (PENDING):** Need to move initialization to FastAPI startup event with retry logic

## Fixes Applied So Far

### In Codespaces Environment (`/workspaces/LLM DevOps Copilots`)

1. ✅ **Fixed Dockerfile**
   ```bash
   sed -i 's/"main:app"/"app.main:app"/' /workspaces/LLM DevOps Copilots/services/llm-service/Dockerfile
   ```

2. ✅ **Fixed Import Statements**
   ```bash
   sed -i 's/from llm_client import/from .llm_client import/' /workspaces/LLM DevOps Copilots/services/llm-service/app/main.py
   sed -i 's/from rag_pipeline import/from .rag_pipeline import/' /workspaces/LLM DevOps Copilots/services/llm-service/app/main.py
   ```

3. ✅ **Rebuilt Docker Image**
   ```bash
   docker-compose build --no-cache llm-service
   ```

4. ⏳ **Pending: Fix RAG Pipeline Initialization** (see next section)

## What You Need to Do Next

### Step 1: Fix the RAG Pipeline Initialization

The service is currently failing because it tries to connect to Qdrant during module import. You need to make this initialization lazy (happen in the startup event).

**Option A: Manual Edit**

Edit `/workspaces/LLM DevOps Copilots/services/llm-service/app/main.py`:

1. Comment out lines 50-53 (around these line numbers):
   ```python
   # llm_config = LLMConfig()
   # rag_config = RAGConfig()
   # llm_client = LLMClient(llm_config)
   # rag_pipeline = RAGPipeline(llm_client, rag_config)

   # Lazy initialization - will be set in startup event
   llm_config = None
   rag_config = None
   llm_client = None
   rag_pipeline = None
   ```

2. Update the `startup_event` function (around line 426):
   ```python
   @app.on_event("startup")
   async def startup_event():
       """Initialize services on startup"""
       global llm_config, rag_config, llm_client, rag_pipeline

       logger.info("Starting LLM Service...")

       # Initialize configurations
       llm_config = LLMConfig()
       rag_config = RAGConfig()

       # Initialize LLM client
       llm_client = LLMClient(llm_config)

       # Initialize RAG pipeline with retry logic
       import time
       for attempt in range(5):
           try:
               rag_pipeline = RAGPipeline(llm_client, rag_config)
               break
           except Exception as e:
               logger.warning(f"Failed to connect to Qdrant (attempt {attempt+1}/5): {e}")
               if attempt < 4:
                   time.sleep(2)
               else:
                   logger.error("Could not connect to Qdrant after 5 attempts")
                   raise

       logger.info(f"Configured providers: {list(llm_client.clients.keys())}")
       logger.info(f"Vector store collection: {rag_config.collection_name}")
       logger.info("LLM Service ready!")
   ```

**Option B: Use the Python Script**

Run the Python script provided in the previous message to automatically apply these changes.

### Step 2: Restart the Service

After making the changes:

```bash
# Navigate to your project
cd /workspaces/LLM DevOps Copilots

# Restart the service (no rebuild needed as code is volume-mounted)
docker-compose restart llm-service

# Wait for startup
sleep 10

# Check the logs
docker-compose logs llm-service | tail -30

# Check service status
docker-compose ps llm-service

# Test the health endpoint
curl http://localhost:8000/health
```

### Step 3: Verify Service is Running

You should see:
- Status: `Up X minutes (healthy)`
- Health endpoint returns: `{"status":"healthy","service":"llm-service",...}`
- No more connection errors in logs

## Current Service Status

```
STATUS: Unhealthy
REASON: RAG pipeline trying to connect to Qdrant during module import
IMPACT: Service keeps restarting, never becomes healthy
NEXT ACTION: Apply the lazy initialization fix (Step 1 above)
```

## Files Modified

### In Codespaces (`/workspaces/LLM DevOps Copilots`)
1. `services/llm-service/Dockerfile` - Fixed CMD instruction
2. `services/llm-service/app/main.py` - Fixed imports (lazy init pending)

### On Local Machine (Already Fixed)
1. `C:\Users\s.babu21\Downloads\node-new\node-v24.10.0-win-x64\devops\services\llm-service\Dockerfile`
2. `C:\Users\s.babu21\Downloads\node-new\node-v24.10.0-win-x64\devops\services\llm-service\app\main.py`

Note: The local fixes don't affect Codespaces. The two environments are separate.

## Architecture Overview

```
Directory Structure:
/workspaces/LLM DevOps Copilots/
├── services/
│   └── llm-service/
│       ├── Dockerfile          (Fixed: app.main:app)
│       ├── requirements.txt
│       └── app/
│           ├── main.py         (Fixed: relative imports, lazy init pending)
│           ├── llm_client.py
│           ├── rag_pipeline.py
│           └── __init__.py
└── docker-compose.yml

Dependencies:
llm-service → qdrant (vector database)
llm-service → redis (cache)
llm-service → rabbitmq (message queue)
```

## Testing After Fix

Once the service is healthy, test these endpoints:

```bash
# Health check
curl http://localhost:8000/health

# API documentation
curl http://localhost:8000/

# Swagger docs
open http://localhost:8000/docs
```

## Common Issues

1. **"Connection refused" to Qdrant**: Make sure Qdrant is running and healthy
   ```bash
   docker-compose ps qdrant
   curl http://localhost:6333/healthz
   ```

2. **Import errors**: Ensure relative imports are used (`.llm_client`, `.rag_pipeline`)

3. **Module not found**: Ensure Dockerfile CMD uses `app.main:app`

4. **Service keeps restarting**: This is the current issue - apply the lazy initialization fix

## Questions?

If you encounter any issues:
1. Check the logs: `docker-compose logs llm-service`
2. Check service status: `docker-compose ps`
3. Verify dependencies are healthy: `docker-compose ps`
4. Check network connectivity: `docker exec devops-llm-service curl http://qdrant:6333/healthz`
