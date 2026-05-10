# Operations Manual - AI-Driven Hybrid Kubernetes System

## Table of Contents
1. [Monitoring and Alerting](#monitoring-and-alerting)
2. [Log Analysis](#log-analysis)
3. [Backup and Recovery](#backup-and-recovery)
4. [Scaling Procedures](#scaling-procedures)
5. [Performance Tuning](#performance-tuning)
6. [Incident Response](#incident-response)
7. [Maintenance Procedures](#maintenance-procedures)
8. [Runbooks](#runbooks)

---

## Monitoring and Alerting

### Monitoring Stack

The system uses a comprehensive monitoring stack:

- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **Alertmanager**: Alert routing and notification
- **Node Exporter**: System metrics
- **kube-state-metrics**: Kubernetes object metrics

### Accessing Monitoring Tools

#### Prometheus

```bash
# Port-forward to Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Open browser
open http://localhost:9090
```

**Key Prometheus Queries**:

```promql
# CPU usage by pod
sum(rate(container_cpu_usage_seconds_total{namespace="ai-system"}[5m])) by (pod)

# Memory usage by pod
sum(container_memory_working_set_bytes{namespace="ai-system"}) by (pod) / 1024 / 1024

# Request rate by service
sum(rate(http_requests_total{namespace="ai-system"}[5m])) by (service)

# Error rate
sum(rate(http_requests_total{namespace="ai-system",status=~"5.."}[5m])) by (service)

# Request latency (p95)
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# LLM API request count
sum(llm_requests_total) by (provider, endpoint)

# Queue depth
rabbitmq_queue_messages{queue="llm_tasks"}

# Database connections
pg_stat_database_numbackends{datname="ai_system"}
```

#### Grafana

```bash
# Port-forward to Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Open browser
open http://localhost:3000
# Login: admin / admin123
```

**Pre-built Dashboards**:

1. **Kubernetes Cluster Overview**
   - Node CPU/Memory usage
   - Pod count and status
   - Network I/O
   - Disk usage

2. **Application Metrics**
   - Request rate and latency
   - Error rates
   - Active connections
   - Cache hit ratio

3. **LLM Service Dashboard**
   - API call volume by provider
   - Token usage
   - Response times
   - Error rates by endpoint

4. **Database Performance**
   - Query performance
   - Connection pool usage
   - Transaction rate
   - Cache statistics

5. **Queue Metrics**
   - Message throughput
   - Queue depth
   - Consumer lag
   - Processing time

### Key Metrics to Monitor

#### System Health Metrics

| Metric                  | Normal Range    | Warning Threshold | Critical Threshold |
|-------------------------|-----------------|-------------------|--------------------|
| CPU Usage (avg)         | < 60%           | > 70%             | > 85%              |
| Memory Usage (avg)      | < 70%           | > 80%             | > 90%              |
| Disk Usage              | < 70%           | > 80%             | > 90%              |
| Pod Restart Count       | 0               | > 3/hour          | > 5/hour           |
| API Response Time (p95) | < 500ms         | > 1s              | > 3s               |
| Error Rate              | < 0.1%          | > 1%              | > 5%               |
| Queue Depth             | < 100           | > 500             | > 1000             |

#### Application Metrics

```yaml
# LLM Service
- llm_requests_total: Total API requests
- llm_request_duration_seconds: Request latency
- llm_errors_total: Error count by type
- llm_token_usage_total: Token consumption

# Worker Service
- worker_tasks_processed_total: Tasks completed
- worker_tasks_failed_total: Failed tasks
- worker_task_duration_seconds: Processing time

# Approval Backend
- approval_requests_total: Total approval requests
- approval_response_time_seconds: Response time
- approval_websocket_connections: Active WebSocket connections

# Database
- pg_stat_database_numbackends: Active connections
- pg_stat_database_tup_fetched: Rows read
- pg_stat_database_tup_inserted: Rows inserted
- pg_locks_count: Lock count
```

### Alert Rules

**Critical Alerts** (Immediate response required):

```yaml
# High error rate
- alert: HighErrorRate
  expr: |
    sum(rate(http_requests_total{status=~"5..",namespace="ai-system"}[5m])) by (service)
    / sum(rate(http_requests_total{namespace="ai-system"}[5m])) by (service) > 0.05
  for: 5m
  annotations:
    summary: "High error rate on {{ $labels.service }}"
    description: "Error rate is {{ $value | humanizePercentage }}"

# Pod crash looping
- alert: PodCrashLooping
  expr: rate(kube_pod_container_status_restarts_total{namespace="ai-system"}[15m]) > 0.1
  for: 5m
  annotations:
    summary: "Pod {{ $labels.pod }} is crash looping"

# Database down
- alert: DatabaseDown
  expr: up{job="postgres"} == 0
  for: 1m
  annotations:
    summary: "PostgreSQL database is down"

# High memory usage
- alert: HighMemoryUsage
  expr: |
    container_memory_working_set_bytes{namespace="ai-system"}
    / container_spec_memory_limit_bytes{namespace="ai-system"} > 0.9
  for: 5m
  annotations:
    summary: "High memory usage on {{ $labels.pod }}"
    description: "Memory usage is {{ $value | humanizePercentage }}"

# Queue backup
- alert: QueueBackup
  expr: rabbitmq_queue_messages{queue="llm_tasks"} > 1000
  for: 10m
  annotations:
    summary: "RabbitMQ queue backup detected"
    description: "Queue depth is {{ $value }}"
```

**Warning Alerts** (Monitor closely):

```yaml
# High latency
- alert: HighLatency
  expr: |
    histogram_quantile(0.95,
      sum(rate(http_request_duration_seconds_bucket{namespace="ai-system"}[5m])) by (le, service)
    ) > 3
  for: 10m
  annotations:
    summary: "High latency on {{ $labels.service }}"

# Approaching resource limits
- alert: HighCPUUsage
  expr: |
    sum(rate(container_cpu_usage_seconds_total{namespace="ai-system"}[5m])) by (pod)
    / sum(container_spec_cpu_quota{namespace="ai-system"}) by (pod) > 0.8
  for: 15m
  annotations:
    summary: "High CPU usage on {{ $labels.pod }}"

# Certificate expiring
- alert: CertificateExpiringSoon
  expr: (certmanager_certificate_expiration_timestamp_seconds - time()) / 86400 < 7
  annotations:
    summary: "Certificate expiring in less than 7 days"
```

### Alertmanager Configuration

```yaml
# alertmanager-config.yaml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
  - match:
      severity: critical
    receiver: 'critical-alerts'
    continue: true
  - match:
      severity: warning
    receiver: 'warning-alerts'

receivers:
- name: 'default'
  email_configs:
  - to: 'team@example.com'
    send_resolved: true

- name: 'critical-alerts'
  slack_configs:
  - channel: '#alerts-critical'
    title: 'Critical Alert: {{ .GroupLabels.alertname }}'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
  pagerduty_configs:
  - service_key: 'YOUR_PAGERDUTY_KEY'

- name: 'warning-alerts'
  slack_configs:
  - channel: '#alerts-warning'
    title: 'Warning: {{ .GroupLabels.alertname }}'
```

### Setting Up Notifications

**Slack Notifications**:
```bash
# Create Slack incoming webhook
# https://api.slack.com/messaging/webhooks

# Update Alertmanager config
kubectl create secret generic alertmanager-slack-webhook \
  --from-literal=url='https://hooks.slack.com/services/YOUR/WEBHOOK/URL' \
  -n monitoring
```

**Email Notifications**:
```yaml
# Update alertmanager-config
receivers:
- name: 'email-team'
  email_configs:
  - to: 'team@example.com'
    from: 'alerts@example.com'
    smarthost: 'smtp.gmail.com:587'
    auth_username: 'alerts@example.com'
    auth_password: 'app-specific-password'
    headers:
      Subject: 'Alert: {{ .GroupLabels.alertname }}'
```

**PagerDuty Integration**:
```yaml
receivers:
- name: 'pagerduty'
  pagerduty_configs:
  - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
    description: '{{ .GroupLabels.alertname }}: {{ .Annotations.summary }}'
```

---

## Log Analysis

### Centralized Logging with ELK Stack

#### Elasticsearch

```bash
# Port-forward to Elasticsearch
kubectl port-forward -n logging svc/elasticsearch 9200:9200

# Check cluster health
curl http://localhost:9200/_cluster/health?pretty

# List indices
curl http://localhost:9200/_cat/indices?v

# Search logs
curl -X GET "http://localhost:9200/logstash-*/_search?pretty" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "kubernetes.namespace": "ai-system"
    }
  },
  "size": 10,
  "sort": [
    { "@timestamp": "desc" }
  ]
}
'
```

#### Kibana

```bash
# Port-forward to Kibana
kubectl port-forward -n logging svc/kibana 5601:5601

# Open browser
open http://localhost:5601
```

**Useful Kibana Queries**:

```
# All errors in last hour
level:ERROR AND kubernetes.namespace:ai-system

# LLM service errors
kubernetes.container_name:"llm-service" AND level:ERROR

# Slow requests (> 1 second)
response_time:>1000 AND kubernetes.namespace:ai-system

# Failed authentication attempts
message:"authentication failed" OR message:"invalid credentials"

# Database connection errors
message:"database" AND (message:"connection" OR message:"timeout")

# Specific user actions
user_id:123 AND action:("approve" OR "deny")
```

### Viewing Application Logs

**View logs for a specific service**:
```bash
# LLM Service logs
kubectl logs -f deployment/llm-service -n ai-system

# Follow logs from all pods
kubectl logs -f -l app=llm-service -n ai-system

# View logs from previous container instance
kubectl logs deployment/llm-service -n ai-system --previous

# Tail last 100 lines
kubectl logs --tail=100 deployment/llm-service -n ai-system

# View logs with timestamps
kubectl logs --timestamps=true deployment/llm-service -n ai-system

# Filter logs by pattern
kubectl logs deployment/llm-service -n ai-system | grep ERROR
```

**Export logs to file**:
```bash
# Export logs for analysis
kubectl logs deployment/llm-service -n ai-system > llm-service.log

# Export from specific time range (requires Elasticsearch)
curl -X GET "localhost:9200/logstash-*/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        {"range": {"@timestamp": {"gte": "now-1h", "lte": "now"}}},
        {"match": {"kubernetes.container_name": "llm-service"}}
      ]
    }
  }
}
' > exported-logs.json
```

### Log Aggregation with Stern

```bash
# Install stern
brew install stern  # macOS
# or
wget https://github.com/stern/stern/releases/download/v1.26.0/stern_1.26.0_linux_amd64.tar.gz

# Tail logs from multiple pods
stern llm-service -n ai-system

# Tail with timestamps
stern llm-service -n ai-system --timestamps

# Filter by container
stern llm-service -n ai-system -c llm-service

# Color code by pod
stern llm-service -n ai-system --color always
```

### Common Log Patterns

**Error Investigation**:
```bash
# Find all errors in last hour
kubectl logs --since=1h -l app=llm-service -n ai-system | grep -i error

# Count error types
kubectl logs -l app=llm-service -n ai-system | grep ERROR | cut -d' ' -f5- | sort | uniq -c | sort -rn

# Find stack traces
kubectl logs -l app=llm-service -n ai-system | grep -A 10 "Traceback"
```

---

## Backup and Recovery

### Database Backups

#### PostgreSQL Backup

**Automated Backup with CronJob**:
```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: ai-system
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15-alpine
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: ai-system-secrets
                  key: POSTGRES_PASSWORD
            command:
            - /bin/sh
            - -c
            - |
              BACKUP_FILE="/backup/postgres-backup-$(date +%Y%m%d-%H%M%S).sql.gz"
              pg_dump -h postgres -U ai_user -d ai_system | gzip > $BACKUP_FILE
              echo "Backup completed: $BACKUP_FILE"
              # Upload to S3 (optional)
              # aws s3 cp $BACKUP_FILE s3://backup-bucket/postgres/
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

**Manual Backup**:
```bash
# Create backup directory
kubectl exec -n ai-system postgres-0 -- mkdir -p /backup

# Create backup
kubectl exec -n ai-system postgres-0 -- pg_dump -U ai_user -d ai_system -F c -f /backup/backup-$(date +%Y%m%d).dump

# Copy backup to local machine
kubectl cp ai-system/postgres-0:/backup/backup-20250115.dump ./backup-20250115.dump

# Or use plain SQL format
kubectl exec -n ai-system postgres-0 -- pg_dump -U ai_user -d ai_system > backup.sql
```

**Backup to S3**:
```bash
# Install AWS CLI in container or use dedicated backup pod
kubectl exec -n ai-system postgres-0 -- /bin/bash -c "
  pg_dump -U ai_user -d ai_system | gzip | \
  aws s3 cp - s3://my-backup-bucket/postgres/backup-$(date +%Y%m%d-%H%M%S).sql.gz
"
```

#### PostgreSQL Restore

```bash
# Copy backup to pod
kubectl cp ./backup-20250115.dump ai-system/postgres-0:/tmp/backup.dump

# Restore from custom format
kubectl exec -n ai-system postgres-0 -- pg_restore -U ai_user -d ai_system -c /tmp/backup.dump

# Or restore from SQL file
kubectl exec -i -n ai-system postgres-0 -- psql -U ai_user -d ai_system < backup.sql

# Drop and recreate database (if needed)
kubectl exec -n ai-system postgres-0 -- psql -U ai_user -d postgres -c "DROP DATABASE ai_system;"
kubectl exec -n ai-system postgres-0 -- psql -U ai_user -d postgres -c "CREATE DATABASE ai_system;"
kubectl exec -i -n ai-system postgres-0 -- psql -U ai_user -d ai_system < backup.sql
```

### Redis Backup

**Manual Backup**:
```bash
# Trigger RDB snapshot
kubectl exec -n ai-system redis-0 -- redis-cli -a dev_password SAVE

# Copy RDB file
kubectl cp ai-system/redis-0:/data/dump.rdb ./redis-backup.rdb
```

**Restore Redis**:
```bash
# Stop Redis
kubectl scale deployment redis --replicas=0 -n ai-system

# Copy backup to pod (requires restart)
kubectl cp ./redis-backup.rdb ai-system/redis-0:/data/dump.rdb

# Start Redis
kubectl scale deployment redis --replicas=1 -n ai-system
```

### Application Configuration Backup

```bash
# Backup ConfigMaps
kubectl get configmap -n ai-system -o yaml > configmaps-backup.yaml

# Backup Secrets (base64 encoded)
kubectl get secrets -n ai-system -o yaml > secrets-backup.yaml

# Backup all resources in namespace
kubectl get all -n ai-system -o yaml > ai-system-backup.yaml

# Backup with velero (recommended for production)
velero backup create ai-system-backup --include-namespaces ai-system
```

### Disaster Recovery Plan

**RTO (Recovery Time Objective)**: 1 hour
**RPO (Recovery Point Objective)**: 5 minutes

**Recovery Steps**:

1. **Assess the situation**
   ```bash
   # Check cluster status
   kubectl get nodes
   kubectl get pods --all-namespaces

   # Check recent events
   kubectl get events --all-namespaces --sort-by='.lastTimestamp'
   ```

2. **Restore database**
   ```bash
   # Get latest backup
   LATEST_BACKUP=$(aws s3 ls s3://backup-bucket/postgres/ | sort | tail -n 1 | awk '{print $4}')

   # Download and restore
   aws s3 cp s3://backup-bucket/postgres/$LATEST_BACKUP backup.sql.gz
   gunzip backup.sql.gz
   kubectl exec -i -n ai-system postgres-0 -- psql -U ai_user -d ai_system < backup.sql
   ```

3. **Redeploy services**
   ```bash
   # Apply all manifests
   kubectl apply -k devops/k8s/overlays/prod/

   # Wait for rollout
   kubectl rollout status deployment/llm-service -n ai-system
   ```

4. **Verify system health**
   ```bash
   # Check all pods are running
   kubectl get pods -n ai-system

   # Test endpoints
   curl https://api.example.com/health
   ```

5. **Notify stakeholders**
   - Update status page
   - Send communication to users
   - Document incident in postmortem

---

## Scaling Procedures

### Manual Scaling

**Scale deployment**:
```bash
# Scale up
kubectl scale deployment llm-service --replicas=5 -n ai-system

# Scale down
kubectl scale deployment llm-service --replicas=2 -n ai-system

# Scale multiple deployments
kubectl scale deployment llm-service worker-service --replicas=5 -n ai-system

# Verify scaling
kubectl get deployment -n ai-system
kubectl get hpa -n ai-system
```

**Scale StatefulSet**:
```bash
# Scale PostgreSQL read replicas
kubectl scale statefulset postgres --replicas=3 -n ai-system

# Wait for scale completion
kubectl rollout status statefulset postgres -n ai-system
```

### Auto-Scaling Configuration

**Horizontal Pod Autoscaler (HPA)**:
```yaml
# Update HPA
kubectl edit hpa llm-service-hpa -n ai-system

# Or apply updated configuration
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-service-hpa
  namespace: ai-system
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-service
  minReplicas: 3
  maxReplicas: 15  # Increased from 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60  # Decreased from 70 for earlier scaling
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

**KEDA for Event-Driven Scaling**:
```yaml
# worker-service-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-service-scaler
  namespace: ai-system
spec:
  scaleTargetRef:
    name: worker-service
  minReplicaCount: 3
  maxReplicaCount: 20
  cooldownPeriod: 300
  triggers:
  - type: rabbitmq
    metadata:
      host: amqp://admin:dev_password@rabbitmq.ai-system.svc.cluster.local:5672/
      queueName: llm_tasks
      queueLength: "10"  # Scale up if queue has more than 10 messages per replica
```

**Cluster Autoscaling**:

For AWS EKS:
```bash
# Enable cluster autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Configure autoscaler
kubectl annotate serviceaccount cluster-autoscaler \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::ACCOUNT_ID:role/ClusterAutoscalerRole
```

For GKE:
```bash
# Enable autoscaling on node pool
gcloud container clusters update ai-system-cluster \
  --enable-autoscaling \
  --min-nodes 3 \
  --max-nodes 20 \
  --zone us-central1-a
```

### Pre-Event Scaling

Before major events or expected traffic spikes:

```bash
# Scale up all services
kubectl scale deployment llm-service --replicas=10 -n ai-system
kubectl scale deployment worker-service --replicas=15 -n ai-system
kubectl scale deployment approval-backend --replicas=8 -n ai-system

# Disable scale-down temporarily
kubectl patch hpa llm-service-hpa -n ai-system -p '{"spec":{"behavior":{"scaleDown":{"stabilizationWindowSeconds":3600}}}}'

# Pre-warm cache
./scripts/cache-warmup.sh

# Verify scaling
kubectl get pods -n ai-system | grep Running | wc -l
```

After event:
```bash
# Re-enable normal autoscaling
kubectl patch hpa llm-service-hpa -n ai-system -p '{"spec":{"behavior":{"scaleDown":{"stabilizationWindowSeconds":300}}}}'

# Or scale back manually
kubectl scale deployment llm-service --replicas=3 -n ai-system
```

---

## Performance Tuning

### Database Optimization

**PostgreSQL Tuning**:
```sql
-- Check slow queries
SELECT
  query,
  calls,
  total_time,
  mean_time,
  max_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Analyze table statistics
ANALYZE approvals;

-- Vacuum to reclaim storage
VACUUM ANALYZE;

-- Create indexes for frequently queried columns
CREATE INDEX CONCURRENTLY idx_approvals_created_at ON approvals(created_at);
CREATE INDEX CONCURRENTLY idx_approvals_status_created ON approvals(status, created_at);

-- Optimize PostgreSQL configuration
ALTER SYSTEM SET shared_buffers = '4GB';
ALTER SYSTEM SET effective_cache_size = '12GB';
ALTER SYSTEM SET maintenance_work_mem = '1GB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET wal_buffers = '16MB';
ALTER SYSTEM SET default_statistics_target = 100;
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM SET min_wal_size = '1GB';
ALTER SYSTEM SET max_wal_size = '4GB';
```

**Connection Pooling with PgBouncer**:
```yaml
# pgbouncer-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pgbouncer
  namespace: ai-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pgbouncer
  template:
    metadata:
      labels:
        app: pgbouncer
    spec:
      containers:
      - name: pgbouncer
        image: pgbouncer/pgbouncer:latest
        env:
        - name: DATABASES_HOST
          value: postgres
        - name: DATABASES_PORT
          value: "5432"
        - name: DATABASES_USER
          value: ai_user
        - name: DATABASES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: ai-system-secrets
              key: POSTGRES_PASSWORD
        - name: DATABASES_DBNAME
          value: ai_system
        - name: PGBOUNCER_POOL_MODE
          value: transaction
        - name: PGBOUNCER_MAX_CLIENT_CONN
          value: "1000"
        - name: PGBOUNCER_DEFAULT_POOL_SIZE
          value: "25"
        ports:
        - containerPort: 5432
```

### Redis Optimization

```bash
# Check Redis memory usage
kubectl exec -n ai-system redis-0 -- redis-cli -a dev_password INFO memory

# Set maxmemory policy
kubectl exec -n ai-system redis-0 -- redis-cli -a dev_password CONFIG SET maxmemory 2gb
kubectl exec -n ai-system redis-0 -- redis-cli -a dev_password CONFIG SET maxmemory-policy allkeys-lru

# Enable persistence
kubectl exec -n ai-system redis-0 -- redis-cli -a dev_password CONFIG SET save "900 1 300 10 60 10000"

# Monitor slow commands
kubectl exec -n ai-system redis-0 -- redis-cli -a dev_password SLOWLOG GET 10
```

### Application Performance

**Python (FastAPI) Optimization**:
```python
# Use connection pooling
from sqlalchemy.pool import QueuePool

engine = create_engine(
    DATABASE_URL,
    poolclass=QueuePool,
    pool_size=20,
    max_overflow=40,
    pool_pre_ping=True,
    pool_recycle=3600
)

# Enable response caching
from fastapi_cache import FastAPICache
from fastapi_cache.backends.redis import RedisBackend

@app.on_event("startup")
async def startup():
    redis = aioredis.from_url("redis://localhost")
    FastAPICache.init(RedisBackend(redis), prefix="fastapi-cache")

# Use background tasks for non-critical operations
from fastapi import BackgroundTasks

@app.post("/api/v1/process")
async def process(data: dict, background_tasks: BackgroundTasks):
    background_tasks.add_task(send_notification, data)
    return {"status": "processing"}
```

**Node.js Optimization**:
```javascript
// Use clustering
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster) {
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
} else {
  // Worker process
  require('./server.js');
}

// Connection pooling for PostgreSQL
const { Pool } = require('pg');
const pool = new Pool({
  max: 20,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Redis caching middleware
const cache = (duration) => {
  return async (req, res, next) => {
    const key = `cache:${req.originalUrl}`;
    const cached = await redis.get(key);
    if (cached) {
      return res.json(JSON.parse(cached));
    }
    res.sendResponse = res.json;
    res.json = (body) => {
      redis.setex(key, duration, JSON.stringify(body));
      res.sendResponse(body);
    };
    next();
  };
};
```

### Resource Limits Tuning

```yaml
# Optimized resource requests and limits
resources:
  requests:
    memory: "2Gi"    # Guaranteed
    cpu: "1000m"     # Guaranteed
  limits:
    memory: "4Gi"    # Max before OOMKill
    cpu: "2000m"     # Throttled if exceeded

# JVM tuning for Java applications
env:
- name: JAVA_OPTS
  value: "-Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"
```

---

## Incident Response

### Incident Severity Levels

| Level | Description | Response Time | Example |
|-------|-------------|---------------|---------|
| P0 | Critical - Complete outage | < 15 minutes | Database down, all services unavailable |
| P1 | High - Major functionality impacted | < 30 minutes | LLM API not responding, approvals blocked |
| P2 | Medium - Partial functionality impacted | < 2 hours | Slow response times, high error rate |
| P3 | Low - Minor issues | < 24 hours | UI glitch, non-critical feature broken |

### Incident Response Playbook

**1. Detection and Alert**
```bash
# Check what triggered the alert
kubectl get events --sort-by='.lastTimestamp' -n ai-system | head -20

# Check pod status
kubectl get pods -n ai-system

# Check recent deployments
kubectl rollout history deployment/llm-service -n ai-system
```

**2. Initial Assessment**
```bash
# Check service health
curl https://api.example.com/health

# Check metrics
kubectl port-forward -n monitoring svc/prometheus 9090:9090 &
# Query error rate, latency, resource usage

# Check logs for errors
kubectl logs --tail=100 -l app=llm-service -n ai-system | grep -i error
```

**3. Communication**
- Post in incident channel: `#incidents`
- Update status page
- Notify stakeholders if P0/P1

**4. Mitigation**
- Rollback if recent deployment
- Scale up resources if needed
- Restart failing pods
- Switch to backup/DR if necessary

**5. Resolution**
- Verify fix
- Monitor for stability
- Close incident
- Schedule postmortem

### Common Incident Scenarios

#### Scenario 1: High Error Rate

**Symptoms**:
- Alerts firing for high error rate
- Users reporting failures
- 5xx errors in logs

**Investigation**:
```bash
# Check error logs
kubectl logs -l app=llm-service -n ai-system | grep "500\|error" | tail -50

# Check recent changes
kubectl rollout history deployment/llm-service -n ai-system

# Check external dependencies
curl -I https://api.openai.com/v1/chat/completions
```

**Mitigation**:
```bash
# Rollback if recent deployment
kubectl rollout undo deployment/llm-service -n ai-system

# Or restart pods
kubectl rollout restart deployment/llm-service -n ai-system

# Increase resource limits if OOM
kubectl set resources deployment/llm-service --limits=memory=6Gi -n ai-system
```

#### Scenario 2: Database Connection Issues

**Symptoms**:
- "connection refused" errors
- "too many connections" errors
- Slow database queries

**Investigation**:
```bash
# Check PostgreSQL status
kubectl get pods -n ai-system | grep postgres

# Check connections
kubectl exec -n ai-system postgres-0 -- psql -U ai_user -d ai_system -c \
  "SELECT count(*) FROM pg_stat_activity;"

# Check slow queries
kubectl exec -n ai-system postgres-0 -- psql -U ai_user -d ai_system -c \
  "SELECT pid, usename, application_name, state, query
   FROM pg_stat_activity
   WHERE state != 'idle'
   ORDER BY query_start;"
```

**Mitigation**:
```bash
# Kill long-running queries
kubectl exec -n ai-system postgres-0 -- psql -U ai_user -d ai_system -c \
  "SELECT pg_terminate_backend(pid)
   FROM pg_stat_activity
   WHERE state = 'active' AND query_start < now() - interval '5 minutes';"

# Increase connection limit
kubectl exec -n ai-system postgres-0 -- psql -U ai_user -d postgres -c \
  "ALTER SYSTEM SET max_connections = 200;"

# Restart PostgreSQL
kubectl delete pod postgres-0 -n ai-system
```

#### Scenario 3: Queue Backup

**Symptoms**:
- High queue depth alert
- Tasks not being processed
- Worker service issues

**Investigation**:
```bash
# Check queue status
kubectl exec -n ai-system rabbitmq-0 -- rabbitmqctl list_queues name messages consumers

# Check worker logs
kubectl logs -l app=worker-service -n ai-system --tail=100

# Check worker pod status
kubectl get pods -n ai-system | grep worker
```

**Mitigation**:
```bash
# Scale up workers
kubectl scale deployment worker-service --replicas=10 -n ai-system

# Purge dead letter queue if needed
kubectl exec -n ai-system rabbitmq-0 -- rabbitmqctl purge_queue llm_tasks_dlq

# Restart workers
kubectl rollout restart deployment/worker-service -n ai-system
```

### Postmortem Template

After resolving an incident, document it:

```markdown
# Incident Postmortem: [Title]

**Date**: 2025-01-15
**Duration**: 2 hours 15 minutes
**Severity**: P1
**Impact**: 15% of users affected, LLM API unavailable

## Timeline
- 09:00 - Alert fired: High error rate on LLM service
- 09:05 - On-call engineer paged
- 09:10 - Investigation started
- 09:30 - Root cause identified: API key rate limit exceeded
- 09:35 - Mitigation: Switched to backup API key
- 10:00 - Service restored
- 11:15 - Full resolution confirmed

## Root Cause
OpenAI API rate limit exceeded due to unexpected traffic spike from new customer.

## Impact
- 15% of users experienced failed LLM requests
- Average response time increased to 5 seconds
- 2,500 failed requests

## Resolution
- Switched to backup OpenAI API key
- Implemented better rate limiting
- Added monitoring for API quota usage

## Action Items
- [ ] Implement API key rotation (Owner: @john, Due: 2025-01-20)
- [ ] Add quota monitoring alerts (Owner: @jane, Due: 2025-01-18)
- [ ] Document rate limit procedures (Owner: @bob, Due: 2025-01-22)
- [ ] Increase API quota with OpenAI (Owner: @alice, Due: 2025-01-17)

## Lessons Learned
- Need better visibility into external API usage
- Should have multiple API keys configured with automatic failover
- Traffic spike detection needs improvement
```

---

## Maintenance Procedures

### Routine Maintenance Tasks

**Daily**:
- Check system health dashboards
- Review critical alerts
- Monitor resource usage trends
- Check backup completion

**Weekly**:
- Review performance metrics
- Analyze error logs
- Update documentation
- Check certificate expiration
- Review capacity planning

**Monthly**:
- Security updates
- Database maintenance (VACUUM, ANALYZE)
- Review and rotate logs
- Update dependencies
- Disaster recovery drill

### Kubernetes Maintenance

**Node Maintenance**:
```bash
# Drain node (move pods to other nodes)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Perform maintenance (OS updates, etc.)
# ...

# Uncordon node (allow scheduling)
kubectl uncordon <node-name>

# Verify node is ready
kubectl get nodes
```

**Upgrade Kubernetes**:
```bash
# Check current version
kubectl version

# Upgrade control plane (managed cluster)
# AWS EKS
eksctl upgrade cluster --name ai-system-cluster --version 1.28

# GKE
gcloud container clusters upgrade ai-system-cluster --master --cluster-version 1.28

# Upgrade node groups
eksctl upgrade nodegroup --cluster ai-system-cluster --name standard-workers

# Verify upgrade
kubectl get nodes
```

### Certificate Renewal

```bash
# Check certificate expiration
kubectl get certificate -n ai-system

# Force renewal
kubectl delete certificate ai-system-tls -n ai-system
# cert-manager will automatically recreate

# Or manually trigger
kubectl annotate certificate ai-system-tls -n ai-system cert-manager.io/issue-temporary-certificate="true" --overwrite
```

### Database Maintenance

```bash
# Connect to PostgreSQL
kubectl exec -it -n ai-system postgres-0 -- psql -U ai_user -d ai_system

# Run VACUUM to reclaim storage
VACUUM VERBOSE;

# Analyze tables to update statistics
ANALYZE;

# Reindex to fix index bloat
REINDEX DATABASE ai_system;

# Check table sizes
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

### Cleanup Procedures

```bash
# Clean up old Docker images
docker system prune -a -f

# Clean up completed jobs
kubectl delete job --field-selector status.successful=1 -n ai-system

# Clean up old ReplicaSets
kubectl delete replicaset --all -n ai-system

# Clean up evicted pods
kubectl get pods -n ai-system | grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n ai-system

# Clean up old logs in Elasticsearch
curl -X DELETE "http://localhost:9200/logstash-2025.01.*"
```

---

## Runbooks

### Quick Reference Commands

```bash
# Get pod name
export POD_NAME=$(kubectl get pods -n ai-system -l app=llm-service -o jsonpath='{.items[0].metadata.name}')

# Exec into pod
kubectl exec -it $POD_NAME -n ai-system -- /bin/bash

# Copy file from pod
kubectl cp ai-system/$POD_NAME:/app/logs/error.log ./error.log

# Port forward
kubectl port-forward -n ai-system svc/llm-service 8000:8000

# Watch resources
watch kubectl get pods -n ai-system

# Tail logs
kubectl logs -f $POD_NAME -n ai-system

# Get events
kubectl get events -n ai-system --sort-by='.lastTimestamp' | tail -20

# Check resource usage
kubectl top pods -n ai-system
kubectl top nodes
```

### Emergency Contacts

- **On-Call Engineer**: Pager rotation
- **DevOps Lead**: @devops-lead
- **Platform Team**: #platform-team
- **Security Team**: #security-team
- **Escalation**: @engineering-manager

---

## Conclusion

This operations manual provides comprehensive guidance for operating the AI-Driven Hybrid Kubernetes System. Regular review and updates of these procedures ensure smooth operations and quick incident resolution.

For additional support, consult the team documentation or reach out in team channels.
