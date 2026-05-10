# 🚀 Quick Start - 5 Minutes to Running

Get your DevOps Agent Platform running locally in 5 minutes!

---

## ⚡ Super Quick Start

```powershell
# 1. Make sure Docker Desktop is running

# 2. Add your API key to .env file
# Edit .env and add: OPENAI_API_KEY=sk-your-key-here

# 3. Start everything
powershell -File start.ps1

# 4. Open dashboards
# - Approval Dashboard: http://localhost:3001
# - Ops Dashboard: http://localhost:3003
# - Grafana: http://localhost:3002 (admin/admin)

# 5. Test it works
curl -X POST http://localhost:8080/spike/cpu
```

---

## 📚 Files You Need

| File | Purpose | When to Use |
|------|---------|-------------|
| **start.ps1** | Start all services | Every time you want to run the platform |
| **stop.ps1** | Stop all services | When you're done for the day |
| **verify.ps1** | Check everything is working | If something seems broken |
| **run_tests.ps1** | Run test suite | Before deploying or after code changes |
| **LOCAL_DEPLOYMENT_GUIDE.md** | Full documentation | If you need detailed info or troubleshooting |

---

## 🎯 Common Commands

### Start the Platform
```powershell
# Basic start (just Docker Compose)
powershell -File start.ps1

# Start with Kubernetes
powershell -File start.ps1 -WithKubernetes

# Quick start (skip waits - faster but may have issues)
powershell -File start.ps1 -Quick
```

### Check Status
```powershell
# Full health check
powershell -File verify.ps1

# Quick status
docker-compose ps

# View logs
docker-compose logs -f
```

### Stop the Platform
```powershell
# Stop (keeps your data)
powershell -File stop.ps1

# Stop and DELETE everything
powershell -File stop.ps1 -RemoveData

# Stop with Kubernetes
powershell -File stop.ps1 -WithKubernetes
```

### Test the System
```powershell
# Run all tests
powershell -File run_tests.ps1

# Trigger a test incident
curl -X POST http://localhost:8080/spike/cpu

# Watch what happens
docker-compose logs -f analyzer-agent
```

---

## 🌐 Access Your Services

After running `start.ps1`, open these URLs:

### Main Dashboards
- **Approval Dashboard**: http://localhost:3001
  - Review and approve AI-suggested fixes

- **Ops Dashboard**: http://localhost:3003
  - Watch incidents in real-time

- **Grafana**: http://localhost:3002
  - Username: `admin`
  - Password: `admin`
  - View metrics and create custom dashboards

### Monitoring Tools
- **Prometheus**: http://localhost:9090
  - Query metrics directly

- **RabbitMQ**: http://localhost:15672
  - Username: `devops`
  - Password: `devops123`
  - See message queues

### APIs
- **LLM Service**: http://localhost:8000/docs
  - FastAPI auto-generated docs

- **Test App**: http://localhost:8080
  - Simple app to trigger test incidents

---

## 🧪 Testing the Platform

### 1. Trigger a CPU Incident
```powershell
curl -X POST http://localhost:8080/spike/cpu
```

**What happens:**
1. Monitoring Agent detects high CPU
2. Analyzer Agent performs root cause analysis using LLM
3. Auto-Response Agent decides to scale or request approval
4. Notifier Agent sends alert (if Slack configured)
5. You see everything in the dashboards!

### 2. Watch the Logs
```powershell
# Watch all agents
docker-compose logs -f monitoring-agent analyzer-agent auto-response-agent

# Watch specific agent
docker-compose logs -f analyzer-agent
```

### 3. Check Dashboards
- Open http://localhost:3003 (Ops Dashboard)
- You should see the incident appear in real-time
- Watch it flow through: Detection → Analysis → Action

---

## ❓ Troubleshooting

### "Container shows unhealthy"
```powershell
# Check if it's actually working
powershell -File verify.ps1

# Most "unhealthy" warnings are false alarms due to health check timeouts
# If verify.ps1 shows ✓ for the service, it's fine!
```

### "Port already in use"
```powershell
# Find what's using the port (example: port 3000)
netstat -ano | findstr :3000

# Kill the process or change the port in docker-compose.yml
```

### "No incidents showing in dashboard"
```powershell
# 1. Trigger a test incident
curl -X POST http://localhost:8080/spike/cpu

# 2. Check monitoring agent is detecting it
docker-compose logs monitoring-agent | Select-String "Incident"

# 3. Check RabbitMQ has messages
# Open http://localhost:15672 and look for queues with messages
```

### "LLM Service not working"
```powershell
# Check API key is set
docker-compose exec llm-service env | Select-String "OPENAI_API_KEY"

# If empty, add to .env:
# OPENAI_API_KEY=sk-your-key-here

# Restart LLM service
docker-compose restart llm-service
```

---

## 📖 Need More Help?

- **Full Guide**: See `LOCAL_DEPLOYMENT_GUIDE.md`
- **Architecture**: See `README.md`
- **Testing**: See `tests/README.md`
- **Logs**: Run `docker-compose logs <service-name>`

---

## 🎉 Success Checklist

Your platform is working when:

- ✅ `powershell -File verify.ps1` shows all ✓ green checkmarks
- ✅ http://localhost:3001 and http://localhost:3003 load
- ✅ `curl -X POST http://localhost:8080/spike/cpu` triggers an incident
- ✅ You see incident detection in logs: `docker-compose logs analyzer-agent`
- ✅ Dashboards show real-time updates

---

## 💡 Tips

1. **First Time Setup**: Takes 3-5 minutes for all services to fully start
2. **Health Checks**: Some services show "unhealthy" but work fine - use `verify.ps1` to check
3. **API Keys**: You need either OpenAI or Anthropic API key in `.env`
4. **Logs**: Always check logs if something seems wrong: `docker-compose logs -f`
5. **Kubernetes**: Optional - only use `-WithKubernetes` if you want to test K8s features

---

## 🚀 What's Next?

1. Explore the dashboards
2. Trigger different types of incidents
3. Watch how the AI analyzes and responds
4. Try approving/rejecting actions in the Approval Dashboard
5. Create custom Grafana dashboards
6. Run the test suite: `powershell -File run_tests.ps1`

---

**Ready? Let's go!**

```powershell
powershell -File start.ps1
```

Then open http://localhost:3003 and watch the magic happen! ✨
