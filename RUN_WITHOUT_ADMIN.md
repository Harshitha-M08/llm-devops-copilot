# Running Without Docker (No Admin Rights Required!)

Since you don't have admin rights to install Docker, here's how to run a **simplified demo** using just Python and Node.js.

## What You Can Run Without Docker

✅ **LLM Service** - Python FastAPI (works standalone)
✅ **Simple Demo Dashboard** - Node.js/React
❌ Full system with databases (requires Docker or cloud deployment)

---

## Quick Demo Setup (5 minutes)

### Step 1: Install Python Dependencies

```powershell
cd C:\Users\s.babu21\Downloads\node-new\node-v24.10.0-win-x64\devops\services\llm-service

# Install required packages
pip install fastapi uvicorn openai pydantic pydantic-settings tenacity --user
```

### Step 2: Create a Simple Demo Script

I'll create a simplified version that works without databases:

```powershell
# Stay in llm-service directory
cd C:\Users\s.babu21\Downloads\node-new\node-v24.10.0-win-x64\devops\services\llm-service
```

### Step 3: Set Your API Key

```powershell
# PowerShell
$env:OPENAI_API_KEY="your-openai-key-here"

# OR for Command Prompt
set OPENAI_API_KEY=your-openai-key-here
```

### Step 4: Run the Demo

```powershell
python demo_standalone.py
```

Then open: **http://localhost:8000**

---

## Alternative: Use Cloud Services (No Local Installation)

### Option 1: GitHub Codespaces (Recommended - Free!)

1. **Push your project to GitHub**
2. **Open in Codespaces** (has Docker pre-installed!)
3. **Run:** `docker-compose up -d`
4. **Access via port forwarding**

**Steps:**
```bash
# In Codespaces terminal:
cd devops
cp .env.example .env
# Edit .env with your API key
docker-compose up -d
```

Codespaces gives you a full Linux VM with Docker - no admin rights needed!

### Option 2: Google Cloud Shell (Free!)

1. Go to: https://shell.cloud.google.com
2. Upload your project
3. Run with Docker (pre-installed)

### Option 3: Replit (Free, Browser-Based)

1. Go to: https://replit.com
2. Create new Repl
3. Upload your code
4. Run directly in browser

### Option 4: AWS Cloud9 (Free Tier)

1. AWS Cloud9 IDE (browser-based)
2. Has Docker pre-installed
3. No admin rights needed

---

## Option: Run Services Individually (Manual Setup)

Since you have Python 3.12 and Node.js 24, you can run services one by one:

### Running LLM Service Standalone

```powershell
cd devops\services\llm-service

# Set API key
$env:OPENAI_API_KEY="sk-your-key"

# Run
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

Access: http://localhost:8000/docs

**Note:** This will work but won't have:
- ❌ Vector database (Qdrant) - RAG features disabled
- ❌ Message queue (RabbitMQ) - No async tasks
- ❌ Database (PostgreSQL) - No data persistence

But you CAN test:
- ✅ Chat completions with OpenAI/Gemini
- ✅ Basic LLM functionality
- ✅ API endpoints

---

## Best Solution: GitHub Codespaces (Recommended!)

**This is your best option without admin rights:**

### Step 1: Create GitHub Account (Free)
Go to: https://github.com/signup

### Step 2: Create New Repository
- Click "New repository"
- Name: `ai-kubernetes-system`
- Set to Public or Private
- Click "Create repository"

### Step 3: Upload Your Code

```bash
cd C:\Users\s.babu21\Downloads\node-new\node-v24.10.0-win-x64\devops

# Initialize git
git init
git add .
git commit -m "Initial commit"

# Push to GitHub
git remote add origin https://github.com/YOUR-USERNAME/ai-kubernetes-system.git
git branch -M main
git push -u origin main
```

### Step 4: Open in Codespaces
1. On GitHub, click your repository
2. Click the green "Code" button
3. Click "Codespaces" tab
4. Click "Create codespace on main"

**A browser-based VS Code opens with Docker installed!**

### Step 5: Run the Project
In Codespaces terminal:
```bash
cd devops
cp .env.example .env
# Edit .env and add your API key (click the file in left sidebar)

# Start everything!
docker-compose up -d

# Access services (Codespaces will auto-forward ports)
```

---

## Portable Docker Alternative: Docker Desktop Portable

**If your IT allows USB drives:**

Some organizations allow running portable apps from USB:

1. **Docker Desktop Portable** (unofficial builds exist)
2. **Podman Desktop** - Docker alternative, sometimes allowed
3. **WSL2 + Docker** - Sometimes works without admin if WSL2 is pre-installed

**Check with IT first!**

---

## Summary of Options

| Option | Admin Required? | Effort | Full Features |
|--------|----------------|--------|---------------|
| **GitHub Codespaces** | ❌ No | Low | ✅ Yes |
| Google Cloud Shell | ❌ No | Low | ✅ Yes |
| Replit | ❌ No | Medium | ⚠️ Limited |
| AWS Cloud9 | ❌ No | Medium | ✅ Yes |
| Python Standalone | ❌ No | Low | ❌ No (demo only) |
| Docker Desktop | ✅ Yes | Low | ✅ Yes |

---

## My Recommendation

**Use GitHub Codespaces!**

Why:
- ✅ No admin rights needed
- ✅ Free (60 hours/month)
- ✅ Has Docker pre-installed
- ✅ Full Linux environment
- ✅ Access from any browser
- ✅ All features work perfectly
- ✅ Can show to others via URL

**It's literally made for this exact situation!**

---

## Quick Start: GitHub Codespaces

1. **Sign up:** https://github.com/signup (free)
2. **Create repo** (public or private)
3. **Upload devops folder**
4. **Open in Codespaces**
5. **Run:** `docker-compose up -d`
6. **Done!**

---

## Need Help Setting Up Codespaces?

Let me know and I'll create a step-by-step guide with screenshots!

---

**Bottom line:** You can't run the full stack locally without Docker (needs databases), but **GitHub Codespaces gives you Docker in the cloud for free!** 🚀
