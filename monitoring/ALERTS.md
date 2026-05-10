# Alert Definitions and Runbooks

This document provides detailed information about all configured alerts, their thresholds, and step-by-step runbooks for resolution.

## Table of Contents

- [Service Alerts](#service-alerts)
- [Resource Alerts](#resource-alerts)
- [Database Alerts](#database-alerts)
- [Kubernetes Alerts](#kubernetes-alerts)

## Service Alerts

### HighErrorRate

**Severity**: Critical
**Threshold**: Error rate > 5% for 5 minutes
**Description**: Service is experiencing elevated error rates (5xx responses)

#### Symptoms
- Increased 5xx HTTP responses
- User-facing errors
- Service degradation

#### Diagnosis
```bash
# Check error rate by service
kubectl logs -n <namespace> -l app=<service> --tail=100 | grep ERROR

# Check Prometheus metrics
# Query: rate(http_requests_total{status=~"5.."}[5m])

# View recent errors in Kibana
# Filter: level:error AND kubernetes.container.name:<service>
```

#### Resolution Steps
1. Check service logs for error patterns
2. Verify database connectivity
3. Check external API dependencies
4. Review recent deployments (potential rollback needed)
5. Check resource utilization (CPU/Memory)
6. Scale service if needed: `kubectl scale deployment <service> --replicas=<n>`

---

### VeryHighErrorRate

**Severity**: Page (Immediate attention)
**Threshold**: Error rate > 20% for 2 minutes
**Description**: CRITICAL - Service is experiencing severe errors requiring immediate action

#### Symptoms
- Majority of requests failing
- Service potentially unusable
- High user impact

#### Diagnosis
```bash
# Immediate checks
kubectl get pods -n <namespace> -l app=<service>
kubectl logs -n <namespace> -l app=<service> --tail=50

# Check for crashloops
kubectl describe pod -n <namespace> <pod-name>
```

#### Resolution Steps
1. **IMMEDIATE**: Consider rollback if recent deployment
   ```bash
   kubectl rollout undo deployment/<service> -n <namespace>
   ```
2. Check for cascading failures from dependencies
3. Verify database/cache availability
4. Check for network issues
5. Scale up service immediately if resource-related
6. Engage on-call team if issue persists > 5 minutes

---

### LLMServiceHighLatency

**Severity**: Warning
**Threshold**: P95 latency > 5 seconds for 10 minutes
**Description**: LLM service responses are slower than expected

#### Symptoms
- Slow API responses
- User complaints about performance
- Request timeouts

#### Diagnosis
```bash
# Check current latency
# Prometheus query: histogram_quantile(0.95, rate(llm_request_duration_seconds_bucket[5m]))

# Check LLM provider status
kubectl logs -n default -l app=llm-service --tail=100 | grep -i "timeout\|slow"

# Check resource usage
kubectl top pods -n default -l app=llm-service
```

#### Resolution Steps
1. Check external LLM provider status (OpenAI, Anthropic, etc.)
2. Review token usage - high tokens = slower responses
3. Check for rate limiting from LLM provider
4. Scale LLM service pods if needed
5. Review and optimize prompts if consistently slow
6. Consider implementing request queuing or rate limiting
7. Check network connectivity to LLM provider

---

### LLMServiceHighTokenUsage

**Severity**: Warning
**Threshold**: Token rate > 100,000 tokens/sec for 15 minutes
**Description**: Unusually high token consumption, potential cost impact

#### Symptoms
- Elevated costs from LLM provider
- Potential API quota exhaustion
- Possible abuse or inefficient usage

#### Diagnosis
```bash
# Check token usage patterns
# Prometheus query: rate(llm_tokens_used_total[5m])

# Check top users/requests
kubectl logs -n default -l app=llm-service | grep token_usage | sort -k5 -nr | head -20

# Review in Grafana LLM dashboard
```

#### Resolution Steps
1. Identify source of high token usage
2. Check for anomalous requests or potential abuse
3. Review prompt templates for efficiency
4. Implement or adjust rate limiting per user
5. Consider caching for repeated queries
6. Review and optimize system prompts
7. Notify finance team about potential cost impact

---

### LLMServiceDown

**Severity**: Critical
**Threshold**: Service unreachable for 2 minutes
**Description**: LLM service is not responding

#### Symptoms
- Service endpoints returning 503 or timing out
- All LLM requests failing
- Pods not ready

#### Diagnosis
```bash
# Check pod status
kubectl get pods -n default -l app=llm-service

# Check recent events
kubectl get events -n default --sort-by='.lastTimestamp' | head -20

# Check logs
kubectl logs -n default -l app=llm-service --tail=100
```

#### Resolution Steps
1. Check if pods are running: `kubectl get pods -n default -l app=llm-service`
2. If pods are CrashLooping, check logs for startup errors
3. Verify service configuration (environment variables, secrets)
4. Check resource availability on nodes
5. Restart deployment if needed: `kubectl rollout restart deployment/llm-service`
6. Verify external dependencies (database, cache, LLM provider API)
7. Check for recent configuration changes

---

## Worker Service Alerts

### WorkerQueueBacklog

**Severity**: Warning
**Threshold**: Queue depth > 1000 messages for 10 minutes
**Description**: Worker task queue has significant backlog

#### Symptoms
- Delayed task processing
- Growing queue size
- Slow job completion

#### Diagnosis
```bash
# Check queue metrics
# Prometheus query: rabbitmq_queue_messages{queue="worker-tasks"}

# Check worker pod count
kubectl get pods -n default -l app=worker-service

# Check worker processing rate
# Prometheus query: rate(worker_tasks_completed_total[5m])
```

#### Resolution Steps
1. Scale up worker pods:
   ```bash
   kubectl scale deployment worker-service --replicas=<n>
   ```
2. Check for slow/stuck tasks
3. Review worker logs for errors
4. Verify database/external API performance
5. Consider implementing task prioritization
6. Check if specific task types are causing issues
7. Monitor queue drain rate after scaling

---

### WorkerQueueCriticalBacklog

**Severity**: Critical
**Threshold**: Queue depth > 5000 messages for 5 minutes
**Description**: Worker queue critically backed up - immediate scaling required

#### Symptoms
- Severe task processing delays
- SLA breaches
- System unable to keep up with load

#### Diagnosis
```bash
# Immediate queue check
kubectl exec -n default <rabbitmq-pod> -- rabbitmqctl list_queues name messages

# Check worker status
kubectl get pods -n default -l app=worker-service
kubectl top pods -n default -l app=worker-service
```

#### Resolution Steps
1. **IMMEDIATE**: Scale workers aggressively
   ```bash
   kubectl scale deployment worker-service --replicas=<2x-5x current>
   ```
2. Check for deadlocked/stuck workers
3. Verify RabbitMQ is healthy
4. Consider pausing low-priority job submission
5. Review recent changes to worker code
6. Check database connection pool settings
7. Monitor closely and adjust scaling as needed

---

### WorkerHighFailureRate

**Severity**: Warning
**Threshold**: Task failure rate > 10% for 5 minutes
**Description**: Workers are failing to process tasks successfully

#### Symptoms
- Increased task failures
- Tasks moving to dead letter queue
- Error logs from workers

#### Diagnosis
```bash
# Check failure rate
# Prometheus query: rate(worker_tasks_failed_total[5m]) / rate(worker_tasks_total[5m])

# Review failed tasks
kubectl logs -n default -l app=worker-service | grep -i "failed\|error" | tail -50

# Check Kibana for error patterns
# Filter: tags:worker-service AND level:error
```

#### Resolution Steps
1. Identify common failure patterns in logs
2. Check external dependencies (APIs, databases)
3. Review recent code changes
4. Verify task payload validity
5. Check for resource exhaustion
6. Review timeout configurations
7. Consider implementing retry logic with backoff
8. Move problematic tasks to manual investigation queue

---

## Approval Service Alerts

### ApprovalPendingBacklog

**Severity**: Warning
**Threshold**: > 100 pending approvals for 30 minutes
**Description**: Large number of approvals awaiting decision

#### Symptoms
- Delayed approval processing
- User complaints about waiting times
- SLA risks

#### Diagnosis
```bash
# Check pending count
# Prometheus query: approval_pending_count

# Review approval dashboard in Grafana

# Check for stuck approvals
kubectl logs -n default -l app=approval-service | grep pending
```

#### Resolution Steps
1. Notify approval team of backlog
2. Review approval priorities
3. Check for notification delivery issues
4. Consider increasing timeout for auto-approval
5. Review approval routing rules
6. Identify and expedite urgent approvals
7. Consider implementing approval delegation

---

### ApprovalTimeoutRate

**Severity**: Warning
**Threshold**: Timeout rate > 5% for 30 minutes
**Description**: High rate of approval timeouts

#### Symptoms
- Approvals timing out before decision
- Requests failing due to timeout
- User frustration

#### Diagnosis
```bash
# Check timeout rate
# Prometheus query: rate(approval_timeout_total[1h]) / rate(approval_requests_total[1h])

# Review timeout patterns
kubectl logs -n default -l app=approval-service | grep timeout
```

#### Resolution Steps
1. Review timeout configuration
2. Check notification delivery (email, Slack)
3. Verify approver availability
4. Consider increasing timeout duration
5. Implement reminder notifications
6. Review approval escalation rules
7. Check for timezone/working hours issues

---

## Resource Alerts

### HighCPUUsage

**Severity**: Warning
**Threshold**: CPU > 80% for 10 minutes
**Description**: Pod is using high CPU resources

#### Diagnosis
```bash
# Check CPU usage
kubectl top pods -n <namespace> <pod-name>

# Check CPU throttling
kubectl describe pod -n <namespace> <pod-name> | grep -i throttl

# Profile application (if tools available)
kubectl exec -n <namespace> <pod-name> -- curl http://localhost:8080/debug/pprof/profile
```

#### Resolution Steps
1. Identify CPU-intensive operations in logs
2. Scale deployment horizontally:
   ```bash
   kubectl scale deployment <service> --replicas=<n>
   ```
3. Increase CPU limits if consistently needed:
   ```yaml
   resources:
     limits:
       cpu: 2000m  # increase as needed
   ```
4. Review code for inefficiencies
5. Implement caching where appropriate
6. Consider vertical pod autoscaling

---

### CriticalCPUUsage

**Severity**: Critical
**Threshold**: CPU > 95% for 5 minutes
**Description**: Pod CPU critically high - throttling likely

#### Resolution Steps
1. **IMMEDIATE**: Scale service:
   ```bash
   kubectl scale deployment <service> --replicas=<2x current>
   ```
2. Increase CPU limits immediately
3. Check for infinite loops or runaway processes
4. Consider restarting affected pods
5. Review recent deployments for issues

---

### HighMemoryUsage

**Severity**: Warning
**Threshold**: Memory > 80% for 10 minutes
**Description**: Pod is using high memory

#### Diagnosis
```bash
# Check memory usage
kubectl top pods -n <namespace> <pod-name>

# Check for memory leaks
kubectl logs -n <namespace> <pod-name> | grep -i "memory\|oom"

# Get heap dump (if supported)
kubectl exec -n <namespace> <pod-name> -- curl http://localhost:8080/debug/pprof/heap > heap.prof
```

#### Resolution Steps
1. Check for memory leaks in application
2. Review caching strategies
3. Scale horizontally if memory usage is legitimate
4. Increase memory limits if needed
5. Implement memory limits on caches
6. Review large object creation patterns

---

### CriticalMemoryUsage

**Severity**: Critical
**Threshold**: Memory > 95% for 5 minutes
**Description**: Pod at risk of OOMKill

#### Resolution Steps
1. **IMMEDIATE**: Scale service to distribute load
2. Increase memory limits immediately
3. Check for memory leaks
4. Restart pod if needed (will be done automatically on OOM)
5. Implement memory profiling
6. Review recent changes

---

## Database Alerts

### PostgreSQLDown

**Severity**: Critical
**Threshold**: Database unreachable for 2 minutes
**Description**: PostgreSQL database is not responding

#### Diagnosis
```bash
# Check pod status
kubectl get pods -n default -l app=postgresql

# Check logs
kubectl logs -n default -l app=postgresql --tail=100

# Try connecting
kubectl exec -n default <pg-pod> -- psql -U postgres -c "SELECT 1"
```

#### Resolution Steps
1. Check if pod is running
2. Review logs for crash causes
3. Check disk space on PVC
4. Verify pod resources (CPU/memory)
5. Check for failed connections in logs
6. Restart pod if necessary:
   ```bash
   kubectl delete pod -n default <pg-pod>
   ```
7. Verify backup availability
8. Check replication status if using HA setup

---

### PostgreSQLHighConnections

**Severity**: Warning
**Threshold**: > 80% of max connections for 5 minutes
**Description**: Database connection pool nearly exhausted

#### Diagnosis
```bash
# Check current connections
kubectl exec -n default <pg-pod> -- psql -U postgres -c \
  "SELECT count(*) FROM pg_stat_activity;"

# Check connection limits
kubectl exec -n default <pg-pod> -- psql -U postgres -c \
  "SHOW max_connections;"

# Find connection sources
kubectl exec -n default <pg-pod> -- psql -U postgres -c \
  "SELECT client_addr, count(*) FROM pg_stat_activity GROUP BY client_addr;"
```

#### Resolution Steps
1. Identify services with connection leaks
2. Check application connection pooling configuration
3. Increase max_connections if needed
4. Review and close idle connections
5. Implement connection timeouts
6. Scale database (vertical or read replicas)
7. Audit application code for connection handling

---

### RedisDown

**Severity**: Critical
**Threshold**: Redis unreachable for 2 minutes
**Description**: Redis cache is not responding

#### Diagnosis
```bash
# Check pod status
kubectl get pods -n default -l app=redis

# Check logs
kubectl logs -n default -l app=redis --tail=100

# Try ping
kubectl exec -n default <redis-pod> -- redis-cli ping
```

#### Resolution Steps
1. Check if pod is running
2. Review logs for errors
3. Check memory usage (Redis can crash on OOM)
4. Verify persistence if enabled
5. Restart pod if necessary
6. Check client connection count
7. Verify network connectivity

---

### RedisHighMemoryUsage

**Severity**: Warning
**Threshold**: Memory > 90% for 5 minutes
**Description**: Redis approaching memory limit

#### Diagnosis
```bash
# Check memory usage
kubectl exec -n default <redis-pod> -- redis-cli INFO memory

# Check key count
kubectl exec -n default <redis-pod> -- redis-cli DBSIZE

# Check eviction stats
kubectl exec -n default <redis-pod> -- redis-cli INFO stats | grep evicted
```

#### Resolution Steps
1. Review eviction policy (allkeys-lru recommended)
2. Increase Redis memory limits
3. Review TTL settings on keys
4. Identify large keys:
   ```bash
   kubectl exec <redis-pod> -- redis-cli --bigkeys
   ```
5. Implement more aggressive TTLs
6. Consider Redis cluster for horizontal scaling
7. Review caching strategy

---

## Kubernetes Alerts

### KubernetesNodeNotReady

**Severity**: Critical
**Threshold**: Node not ready for 5 minutes
**Description**: Kubernetes node is not in Ready state

#### Diagnosis
```bash
# Check node status
kubectl get nodes

# Check node conditions
kubectl describe node <node-name>

# Check kubelet logs (on node)
journalctl -u kubelet -n 100
```

#### Resolution Steps
1. Check node conditions (disk pressure, memory pressure, etc.)
2. SSH to node and check resources
3. Restart kubelet if needed:
   ```bash
   systemctl restart kubelet
   ```
4. Check for network issues
5. Verify container runtime is running
6. Drain and restart node if needed
7. Replace node if hardware issues

---

### KubernetesDeploymentReplicasMismatch

**Severity**: Warning
**Threshold**: Mismatch for 10 minutes
**Description**: Deployment unable to maintain desired replica count

#### Diagnosis
```bash
# Check deployment status
kubectl get deployment <deployment-name> -n <namespace>

# Check events
kubectl describe deployment <deployment-name> -n <namespace>

# Check pods
kubectl get pods -n <namespace> -l app=<app>
```

#### Resolution Steps
1. Check for insufficient cluster resources
2. Review pod events for scheduling failures
3. Check for node affinity/anti-affinity rules
4. Verify resource requests/limits
5. Check for PVC binding issues
6. Review image pull errors
7. Check for node taints and tolerations

---

### KubernetesPersistentVolumeClaimPending

**Severity**: Warning
**Threshold**: PVC pending for 10 minutes
**Description**: PersistentVolumeClaim cannot be bound

#### Diagnosis
```bash
# Check PVC status
kubectl get pvc -n <namespace>

# Check PVC events
kubectl describe pvc <pvc-name> -n <namespace>

# Check available PVs
kubectl get pv
```

#### Resolution Steps
1. Check if StorageClass exists
2. Verify provisioner is running
3. Check for available PVs matching claim
4. Review storage class configuration
5. Check provisioner logs
6. Verify cloud provider credentials (if using dynamic provisioning)
7. Create PV manually if using static provisioning

---

## Alert Routing

Alerts are routed based on severity and service:

- **Critical/Page**: PagerDuty + Slack + Email
- **Warning**: Slack + Email
- **Service-specific**: Routed to appropriate team channel

## Contact Information

- **LLM Team**: #llm-service-alerts, llm-team@company.com
- **Worker Team**: #worker-service-alerts, worker-team@company.com
- **Infrastructure Team**: #infrastructure-alerts, infrastructure-team@company.com
- **Ops Team**: #ops-alerts, ops-team@company.com

## Escalation Policy

1. **0-15 min**: On-call engineer responds
2. **15-30 min**: Team lead notified
3. **30-60 min**: Engineering manager notified
4. **60+ min**: Incident commander assigned

## Post-Incident

After resolving an alert:
1. Document resolution steps
2. Update runbook if needed
3. Create post-mortem for critical incidents
4. Implement preventive measures
5. Update alert thresholds if necessary
