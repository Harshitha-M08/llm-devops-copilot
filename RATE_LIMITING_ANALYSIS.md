# 🔍 Root Cause Analysis: Excessive API Calls & Rate Limiting

## 📊 Executive Summary

Your system is hitting rate limits (429) and access errors (403) due to **6 critical issues**:

1. **Ultra-Fast Monitoring Loop**: Checking metrics every **10 seconds**
2. **Short Deduplication Window**: Only **1 minute** (allows same incident to repeat)
3. **No Rate Limiting**: Monitoring agent can fire unlimited incidents
4. **No LLM Response Caching**: Every incident triggers a new AI call
5. **Aggressive Retry Logic**: Failed LLM calls retry 3 times without backoff
6. **Low Thresholds**: CPU at 80%, Memory at 85% triggers too easily

---

## 🚨 Issue #1: Ultra-Fast Monitoring Loop (10s interval)

### Location
- `services/monitoring-agent/app/config.py` line 23
- `docker-compose.yml` line 164
- `.env` line 123

### Current Behavior
```python
prometheus_scrape_interval: int = Field(default=10, description="How often to check metrics (seconds)")
```

**Every 10 seconds**, the monitoring agent:
- Queries Prometheus for 4+ metrics
- Checks each metric against thresholds
- Potentially triggers multiple incidents

### Impact
- **360 metric checks per hour** (6 per minute × 60 minutes)
- If CPU exceeds 80%, you could trigger **6 incidents per minute**
- Even with 1-minute deduplication, you get **60+ incidents per hour**

### Recommendation
✅ **Change to 30-60 seconds** for production systems

---

## 🚨 Issue #2: 1-Minute Deduplication Window (Too Short)

### Location
`services/monitoring-agent/app/db_client.py` lines 169-197

### Current Code
```python
# DEMO MODE: Reduced to 1 minute window for faster incident generation
SELECT incident_id
FROM incidents
WHERE metric_name = $1
AND (labels->>'service' = $2 OR labels->>'pod' = $2)
AND status NOT IN ('resolved', 'closed', 'false_positive')
AND updated_at > NOW() - INTERVAL '1 minute'  # ⚠️ TOO SHORT
```

### Problem
- After 60 seconds, the same metric breach is treated as a **new incident**
- High CPU persisting for 10 minutes = **9-10 duplicate incidents**
- Each duplicate triggers a new LLM analysis call

### Impact Timeline
```
00:00 - CPU 95% → Incident A created → LLM called
01:00 - CPU 95% → Dedup window expired → Incident B created → LLM called
02:00 - CPU 95% → Dedup window expired → Incident C created → LLM called
...
```

### Recommendation
✅ **Increase to 5-10 minutes** to prevent duplicate processing

---

## 🚨 Issue #3: No Rate Limiting on Incident Creation

### Location
`services/monitoring-agent/app/main.py` lines 250-312

### Current Behavior
```python
async def publish_incident(...):
    # Check for existing active incident
    active_incident_id = await self.db_client.check_active_incident(metric, service_name)
    
    if active_incident_id:
        logger.debug(f"⏭️ Skipping duplicate incident")
        return
    
    # ⚠️ NO RATE LIMITING - publishes immediately
    incident_id = str(uuid.uuid4())
    await self.event_publisher.publish(...)
```

### Problem
- No throttling mechanism
- If 5 different metrics breach simultaneously → 5 incidents → 5 LLM calls
- No cooldown period between incidents

### Recommendation
✅ **Add rate limiter**: Max 1 incident per 10 seconds per metric type

---

## 🚨 Issue #4: No LLM Response Caching

### Location
`services/analyzer-agent/app/main.py` lines 191-230

### Current Behavior
```python
async def analyze_with_llm(self, event, logs, similar_incidents):
    # ⚠️ ALWAYS calls LLM - no caching
    prompt = self._build_analysis_prompt(event, logs, similar_incidents)
    
    analysis = await self.llm_analyzer.analyze(
        prompt=prompt,
        timeout=config.analysis_timeout
    )
    return analysis
```

### Problem
- Identical incidents (CPU 95% on same service) analyzed separately
- No cache lookup before calling expensive LLM API
- Wasted API calls for duplicate scenarios

### Example
```
10:00 - CPU 95% → LLM Analysis #1 (Gemini API call)
10:02 - CPU 95% → LLM Analysis #2 (Gemini API call) # ⚠️ Same issue!
10:04 - CPU 95% → LLM Analysis #3 (Gemini API call) # ⚠️ Same issue!
```

### Recommendation
✅ **Implement Redis caching**:
- Cache key: `hash(metric + value_range + service)`
- TTL: 5-10 minutes
- Return cached analysis for identical scenarios

---

## 🚨 Issue #5: Aggressive Retry Without Backoff

### Location
`services/llm-service/app/llm_client.py` lines 153-156

### Current Code
```python
@retry(
    stop=stop_after_attempt(3),  # ⚠️ 3 attempts
    wait=wait_exponential(multiplier=1, min=2, max=10)  # ⚠️ Only 2-10s wait
)
def chat_completion(self, messages, provider, **kwargs):
    # Call LLM API
```

### Problem
When Gemini hits rate limit:
```
Attempt 1: Immediate call → 429 Rate Limit
Attempt 2: Wait 2s → Call → 429 Rate Limit
Attempt 3: Wait 4s → Call → 429 Rate Limit
→ Falls back to OpenAI
Attempt 1: Immediate call → 403 Forbidden
Attempt 2: Wait 2s → Call → 403 Forbidden
Attempt 3: Wait 4s → Call → 403 Forbidden
→ Falls back to Anthropic
Attempt 1: Immediate call → 429 Rate Limit
...
```

**Total API calls for ONE incident: 9+ calls in 20 seconds**

### Recommendation
✅ **Reduce retries to 1-2 attempts**
✅ **Increase backoff**: `wait_exponential(multiplier=2, min=5, max=60)`

---

## 🚨 Issue #6: Overly Sensitive Thresholds

### Location
`.env` lines 123-128

### Current Values
```env
MONITORING_CPU_THRESHOLD=80.0          # ⚠️ Triggers easily
MONITORING_MEMORY_THRESHOLD=85.0       # ⚠️ Triggers easily
MONITORING_ERROR_RATE_THRESHOLD=5.0    # ⚠️ 5% errors
MONITORING_RESPONSE_TIME_THRESHOLD=2000.0
```

### Problem
- Modern applications often run at 70-80% CPU normally
- 80% threshold triggers **too frequently**
- Combined with 10s check interval = **spam**

### Recommendation
✅ **Increase thresholds**:
- CPU: 90%
- Memory: 90%
- Error Rate: 10%

---

## 📈 Projected Impact After Fixes

### Before Fixes (Current State)
```
Monitoring Interval:    10 seconds
Deduplication Window:   1 minute
Rate Limiting:          None
LLM Caching:            None
Retry Attempts:         3 per provider × 3 providers = 9 total
CPU Threshold:          80%

Scenario: CPU stays at 95% for 10 minutes
→ 10 separate incidents created
→ 10 LLM analysis calls
→ With retries: 10 × 9 = 90 API calls
→ Rate limit hit within 2 minutes
```

### After Fixes
```
Monitoring Interval:    30 seconds
Deduplication Window:   5 minutes
Rate Limiting:          1 incident per 10s per metric
LLM Caching:            5 minute TTL
Retry Attempts:         2 total (with longer backoff)
CPU Threshold:          90%

Scenario: CPU stays at 95% for 10 minutes
→ 1 incident created (rate limited)
→ 1 LLM call (or cached response)
→ With retries (if needed): 1 × 2 = 2 API calls maximum
→ **45x reduction in API calls**
```

---

## 🛠️ Implementation Checklist

- [ ] **Monitoring Agent**: Add rate limiter (1 per 10s per metric)
- [ ] **Database**: Change deduplication window from 1 minute to 5 minutes
- [ ] **Analyzer Agent**: Add Redis-based LLM response caching
- [ ] **LLM Service**: Reduce retry attempts and increase backoff
- [ ] **Configuration**: Update thresholds (CPU 90%, Memory 90%)
- [ ] **Configuration**: Change monitoring interval from 10s to 30s
- [ ] **Testing**: Verify fixes with controlled incident simulation

---

## 🎯 Expected Results

✅ **95% reduction in API calls**
✅ **No more rate limiting** under normal load
✅ **Faster incident processing** (no retry delays)
✅ **Lower costs** (fewer LLM API calls)
✅ **Better user experience** (relevant alerts only)

---

## 🔒 Workflow Protection

All fixes preserve the existing workflow:
- ✅ Monitoring → Analyzer → Auto-Response → Approval
- ✅ RabbitMQ event routing unchanged
- ✅ Dashboard real-time updates working
- ✅ Glassmorphic UI unaffected

The changes only **optimize the rate** of incident creation and LLM calls, not the **functionality** of the pipeline.
