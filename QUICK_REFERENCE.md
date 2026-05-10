# Quick Reference Guide - DevOps Copilot

**Fast commands for common operations**

---

## 🚀 Starting the System

```bash
cd "C:\Users\Harshitha Gowda\OneDrive\Desktop\hhagents\LLM DevOps Copilot-main"
docker compose up -d
```

**Wait 2 minutes for all services to start**, then verify:
```bash
docker ps | grep devops
```

---

## 🔍 Monitoring the System

### **Watch Live Logs (All Agents)**
```bash
docker compose logs -f monitoring-agent analyzer-agent auto-response-agent notifier-agent llm-service
```

### **Check Specific Agent**
```bash
# Monitoring agent (detects problems)
docker logs -f devops-monitoring-agent

# Analyzer agent (AI analysis)
docker logs -f devops-analyzer-agent

# Auto-response agent (executes fixes)
docker logs -f devops-auto-response-agent

# LLM service (Anthropic API)
docker logs -f devops-llm-service
```

### **Last 50 Lines of Logs**
```bash
docker compose logs --tail=50 analyzer-agent
```

---

## 🩺 Health Checks

### **Quick Health Check (All Services)**
```bash
curl http://localhost:3000/health      # Approval API
curl http://localhost:8000/health      # LLM Service
curl http://localhost:8001/api/health  # Ops Dashboard
curl http://localhost:9090/-/healthy   # Prometheus
```

### **Check Container Status**
```bash
docker ps --format "table {{.Names}}\t{{.Status}}" | grep devops
```

### **Check Healthy vs Unhealthy**
```bash
docker ps --filter "name=devops-" --filter "health=healthy"
docker ps --filter "name=devops-" --filter "health=unhealthy"
```

---

## 🔧 Troubleshooting

### **Restart Specific Service**
```bash
docker compose restart analyzer-agent
docker compose restart llm-service
docker compose restart monitoring-agent
```

### **Rebuild and Restart**
```bash
docker compose up -d --force-recreate --build analyzer-agent
```

### **Check RabbitMQ Connections**
```bash
# RabbitMQ logs
docker logs devops-rabbitmq --tail=50

# Check queue status
curl -u devops:devops123 http://localhost:15672/api/queues
```

### **Check Database Connection**
```bash
docker exec devops-postgres pg_isready -U devops
```

---

## 📊 Accessing Dashboards

| Dashboard | URL | Credentials |
|-----------|-----|-------------|
| Approval UI | http://localhost:3001 | None |
| Operations Dashboard | http://localhost:3003 | None |
| Grafana | http://localhost:3002 | admin / admin |
| Prometheus | http://localhost:9090 | None |
| RabbitMQ | http://localhost:15672 | devops / devops123 |
| Test App | http://localhost:8080 | None |

---

## 🐛 Common Issues & Fixes

### **Issue: Analyzer Agent Timeout**
**Symptom:** Analyzer logs show "TimeoutError after 120 seconds"

**Fix:**
```bash
# Check LLM service
docker logs devops-llm-service --tail=30

# Verify Anthropic API key
docker exec devops-llm-service env | grep ANTHROPIC_API_KEY

# Restart both services
docker compose restart llm-service analyzer-agent
```

---

### **Issue: No Incidents Being Detected**
**Symptom:** Monitoring agent not publishing incidents

**Check:**
```bash
# See if monitoring is running
docker logs devops-monitoring-agent --tail=30

# Check Prometheus is reachable
curl http://localhost:9090/-/healthy

# Verify test app is generating metrics
curl http://localhost:8080/metrics
```

**Trigger Test Incident:**
```bash
# Stress CPU to trigger anomaly
docker exec devops-test-app /bin/sh -c "while true; do :; done &"

# Check monitoring logs
docker logs -f devops-monitoring-agent
```

---

### **Issue: RabbitMQ Connection Failed**
**Symptom:** "Connect call failed ('172.28.0.3', 5672)"

**Fix:**
```bash
# Restart RabbitMQ
docker compose restart rabbitmq

# Wait 10 seconds, then restart agents
sleep 10
docker compose restart monitoring-agent analyzer-agent auto-response-agent notifier-agent
```

---

### **Issue: Dashboard Shows Unhealthy**
**Symptom:** ops-dashboard-frontend/backend show "(unhealthy)"

**Check if actually working:**
```bash
curl http://localhost:8001/api/health
curl http://localhost:3003/
```

**If both return valid responses, ignore the unhealthy status - it's a cosmetic issue.**

---

## 🔄 Stopping the System

### **Stop All Services**
```bash
docker compose down
```

### **Stop and Remove Volumes (Fresh Start)**
```bash
docker compose down -v
```

---

## 📝 Viewing Incidents

### **Check PostgreSQL for Stored Incidents**
```bash
docker exec -it devops-postgres psql -U devops -d devops_db -c "SELECT * FROM incidents ORDER BY created_at DESC LIMIT 10;"
```

### **Check RabbitMQ Message Count**
```bash
curl -u devops:devops123 http://localhost:15672/api/queues/%2Fdevops
```

---

## 🧪 Testing the Workflow

### **Method 1: Wait for Natural Anomaly**
```bash
# Watch monitoring logs
docker logs -f devops-monitoring-agent

# When you see "🚨 Incident detected", check analyzer
docker logs -f devops-analyzer-agent
```

### **Method 2: Trigger CPU Spike**
```bash
# Start CPU stress
docker exec devops-test-app /bin/sh -c "yes > /dev/null &"

# Watch logs
docker logs -f devops-monitoring-agent

# Kill stress after test
docker restart devops-test-app
```

---

## 🔑 Important File Locations

### **Environment Configuration**
```
C:\Users\Harshitha Gowda\OneDrive\Desktop\hhagents\LLM DevOps Copilot-main\.env
```

### **Docker Compose**
```
C:\Users\Harshitha Gowda\OneDrive\Desktop\hhagents\LLM DevOps Copilot-main\docker-compose.yml
```

### **Agent Source Code**
```
C:\Users\Harshitha Gowda\OneDrive\Desktop\hhagents\LLM DevOps Copilot-main\services\
├── monitoring-agent/
├── analyzer-agent/
├── auto-response-agent/
├── notifier-agent/
└── llm-service/
```

---

## 📞 Quick Diagnostics

### **One-Line System Check**
```bash
echo "Containers:" && docker ps --filter "name=devops-" --format "{{.Names}}: {{.Status}}" | grep -i agent && echo -e "\nHealth:" && curl -s http://localhost:8000/health && curl -s http://localhost:3000/health
```

### **Agent Status Summary**
```bash
for agent in monitoring analyzer auto-response notifier; do echo "$agent:"; docker logs devops-$agent-agent 2>&1 | tail -3; echo ""; done
```

---

## ⚡ Performance Tips

1. **Check resource usage:**
   ```bash
   docker stats --no-stream
   ```

2. **Clear old logs:**
   ```bash
   docker compose logs --tail=0 -f > /dev/null
   ```

3. **Restart only unhealthy containers:**
   ```bash
   docker ps --filter "name=devops-" --filter "health=unhealthy" --format "{{.Names}}" | xargs -r docker restart
   ```

---

## 📚 More Information

- **Full Documentation:** `DEPLOYMENT_SUMMARY.md`
- **Architecture Details:** See README.md
- **Troubleshooting Guide:** This file
- **API Documentation:** http://localhost:8000/docs (LLM Service)

---

**Last Updated:** November 2, 2025
