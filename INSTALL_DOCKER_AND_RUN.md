# Install Docker Desktop & Run the Project

## Step 1: Install Docker Desktop

### Download & Install:

1. **Download Docker Desktop for Windows:**
   - Go to: https://www.docker.com/products/docker-desktop/
   - Click "Download for Windows"
   - Direct link: https://desktop.docker.com/win/main/amd64/Docker%20Desktop%20Installer.exe

2. **Run the Installer:**
   - Double-click the downloaded file
   - Follow the installation wizard
   - **Important:** Enable WSL 2 when prompted
   - Accept the license agreement
   - Click "Install"

3. **Restart Your Computer** (Required!)

4. **Start Docker Desktop:**
   - Find "Docker Desktop" in your Start Menu
   - Launch it
   - You may need to:
     - Accept Docker's terms of service
     - Create a Docker account (free) or skip
   - Wait for Docker to fully start (whale icon in system tray should be steady/green)

5. **Verify Installation:**
   Open a NEW terminal (PowerShell or Command Prompt) and run:
   ```powershell
   docker --version
   docker-compose --version
   ```

   You should see version numbers like:
   ```
   Docker version 24.x.x
   Docker Compose version v2.x.x
   ```

---

## Step 2: Configure API Keys

Before running the project, you MUST add your LLM API key:

### Option A: Use OpenAI (Recommended)

1. Get an API key from: https://platform.openai.com/api-keys
2. Open `devops\.env` in a text editor
3. Find line 79 and replace:
   ```
   OPENAI_API_KEY=your-openai-api-key-here
   ```
   With your actual key:
   ```
   OPENAI_API_KEY=sk-proj-xxxxxxxxxxxxxxxxxxxxx
   ```

### Option B: Use Google Gemini (Alternative)

1. Get an API key from: https://makersuite.google.com/app/apikey
2. Open `devops\.env` in a text editor
3. Add your key (around line 86):
   ```
   GEMINI_API_KEY=your-actual-gemini-key-here
   ```

**Save the file!**

---

## Step 3: Run the Project

### Using the Quick Start Script (Easiest):

Open PowerShell or Command Prompt in the `devops` folder and run:

```powershell
# Navigate to devops folder
cd C:\Users\s.babu21\Downloads\node-new\node-v24.10.0-win-x64\devops

# Run the startup script
.\start.bat
```

The script will:
- ✅ Check if Docker is running
- ✅ Check if .env exists
- ✅ Start all services
- ✅ Show you all the URLs

### Manual Method:

```powershell
cd C:\Users\s.babu21\Downloads\node-new\node-v24.10.0-win-x64\devops
docker-compose up -d
```

---

## Step 4: Access the Services

After starting (wait ~30 seconds for all services to fully start), access:

### Main Interfaces:
| Service | URL | Description |
|---------|-----|-------------|
| **Approval Dashboard** | http://localhost:3001 | Main web interface |
| **LLM Service API** | http://localhost:8000/docs | Interactive API docs |
| **Grafana Dashboards** | http://localhost:3002 | Monitoring |

### Additional Services:
| Service | URL | Credentials |
|---------|-----|-------------|
| RabbitMQ Management | http://localhost:15672 | guest / guest |
| Prometheus | http://localhost:9090 | - |
| Qdrant Dashboard | http://localhost:6333/dashboard | - |

### Default Login:
**Approval Dashboard:**
- Email: `admin@approvaldashboard.com`
- Password: `Admin@123`

**Grafana:**
- Username: `admin`
- Password: `admin`

---

## Step 5: Test the System

### Test 1: Open the Approval Dashboard
1. Go to: http://localhost:3001
2. Login with default credentials
3. Click "Create New Approval"
4. You should see the dashboard working!

### Test 2: Test the LLM API
Open a new PowerShell window:

```powershell
# Health check
curl http://localhost:8000/health

# Test chat (if API key is configured)
curl -X POST http://localhost:8000/api/v1/chat `
  -H "Content-Type: application/json" `
  -d '{
    "messages": [
      {"role": "user", "content": "Hello! Tell me about Kubernetes in one sentence."}
    ]
  }'
```

### Test 3: Check All Services
```powershell
docker-compose ps
```

You should see all services "Up" and healthy!

---

## Common Issues & Solutions

### Issue 1: "Docker is not running"
**Solution:**
- Make sure Docker Desktop is started
- Look for the whale icon in your system tray
- Wait for it to show "Docker Desktop is running"
- Try the command again

### Issue 2: "Port already in use"
**Solution:**
```powershell
# Find what's using the port (example: port 8000)
netstat -ano | findstr :8000

# Kill the process (replace PID with actual process ID)
taskkill /PID <PID> /F

# Or change ports in docker-compose.yml
```

### Issue 3: Services won't start
**Solution:**
```powershell
# View logs
docker-compose logs -f

# Restart everything
docker-compose down
docker-compose up -d
```

### Issue 4: "API key not configured" errors
**Solution:**
- Make sure you edited `.env` file
- Added your actual API key (starts with `sk-` for OpenAI)
- Saved the file
- Restart services:
  ```powershell
  docker-compose restart llm-service
  ```

---

## Useful Commands

```powershell
# Check service status
docker-compose ps

# View logs (all services)
docker-compose logs -f

# View logs (specific service)
docker-compose logs -f llm-service

# Restart a service
docker-compose restart llm-service

# Stop all services
docker-compose stop

# Stop and remove all services
docker-compose down

# Stop and remove everything (including volumes/data)
docker-compose down -v

# Rebuild and restart
docker-compose up -d --build

# See resource usage
docker stats
```

---

## Next Steps

Once everything is running:

1. **Explore the Approval Dashboard**
   - http://localhost:3001
   - Create approvals, test workflows

2. **Try the LLM API**
   - http://localhost:8000/docs
   - Test chat completions, RAG queries

3. **Check Monitoring**
   - http://localhost:3002 (Grafana)
   - View system metrics and dashboards

4. **Read the Documentation**
   - `GETTING_STARTED.md` - Detailed guide
   - `docs/ARCHITECTURE.md` - System design
   - `docs/API.md` - API documentation
   - `docs/DEPLOYMENT.md` - Production deployment

5. **Learn the Code**
   - Explore `services/` folder
   - Read the implementation
   - Make changes and experiment!

---

## Getting Help

If you encounter issues:

1. **Check Docker Desktop is running** (system tray icon)
2. **View logs:** `docker-compose logs -f`
3. **Restart services:** `docker-compose restart`
4. **Check documentation** in `docs/` folder
5. **File an issue** on GitHub

---

## Summary Checklist

- [ ] Docker Desktop installed and running
- [ ] API key added to `.env` file
- [ ] Ran `.\start.bat` or `docker-compose up -d`
- [ ] All services showing "Up" in `docker-compose ps`
- [ ] Opened http://localhost:3001 successfully
- [ ] Logged into Approval Dashboard
- [ ] Tested LLM API at http://localhost:8000/docs

**You're all set! Enjoy exploring the AI-Driven Kubernetes System! 🚀**
