# Option 2: Docker Executor Implementation - Multi-Phase Plan

## Overview
Add Docker Compose executor support so auto-response-agent can execute healing actions in local Docker Compose environment.

**Total Estimated Time:** 7 hours
**Number of Phases:** 5
**Prerequisite:** ✅ Option 1 completed (October 30, 2025)
**Status:** ⏳ **PENDING** - Ready to start

---

## Phase 1: Code Discovery & Analysis ⏳ PENDING
**Time Estimate:** 30 minutes
**Goal:** Understand current K8s executor and plan abstraction

### Tasks:
1. [ ] Locate K8s executor implementation
2. [ ] Understand how auto-response-agent uses the executor
3. [ ] Identify integration points
4. [ ] Check for existing executor abstraction

### Commands to Run:
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
find . -name "*k8s*.py" -o -name "*executor*.py"
grep -r "K8sExecutor\|Executor" app/ -n -A 5
grep -r "def execute\|async def execute" app/ -n
grep -r "kubernetes\|kubectl" app/ -n
```

### Analysis Questions:
- How is K8sExecutor instantiated?
- What methods does it expose? (restart, scale, rollback)
- What parameters do these methods take?
- Is there already a base class/interface?
- How does main.py call the executor?
- What happens when executor fails?

### Deliverables:
- [ ] Document K8sExecutor class structure
- [ ] Map all executor method signatures
- [ ] Identify integration points in main.py
- [ ] List all files that need modification

### Success Criteria:
- Complete understanding of current executor architecture
- Clear list of files to modify/create
- Integration strategy documented

---

## Phase 2: Design Executor Abstraction ⏳ PENDING
**Time Estimate:** 1 hour
**Goal:** Design the executor system architecture

### Sub-Phase 2.1: Design Base Interface (20 min) ⏳

Tasks:
- [ ] Design BaseExecutor abstract class
- [ ] Define common methods: restart(), scale(), rollback()
- [ ] Design ExecutionResult data class
- [ ] Define ActionType enum

Decisions:
- Use ABC (Abstract Base Class)
- All methods async (async def)
- Consistent return type (ExecutionResult)
- Common error handling pattern

### Sub-Phase 2.2: Design Docker Executor (25 min) ⏳

Tasks:
- [ ] Design DockerComposeExecutor class
- [ ] Plan restart() implementation (docker-compose restart)
- [ ] Plan scale() implementation (docker-compose up --scale)
- [ ] Plan rollback() implementation (basic: restart service)
- [ ] Design helper methods (_run_command, _service_exists)

Commands it will run:
```bash
docker-compose -f /compose/docker-compose.yml restart <service>
docker-compose -f /compose/docker-compose.yml up -d --scale <service>=<replicas>
docker-compose -f /compose/docker-compose.yml config --services
```

### Sub-Phase 2.3: Design Executor Factory (15 min) ⏳

Tasks:
- [ ] Design ExecutorFactory class
- [ ] Plan auto-detection logic
- [ ] Define environment variables for configuration

Auto-detection logic:
1. Check EXECUTOR_TYPE env var (explicit)
2. Check KUBERNETES_SERVICE_HOST (in K8s cluster)
3. Check ~/.kube/config exists (can access K8s)
4. Check /var/run/docker.sock exists (Docker available)
5. Default to Docker Compose

### Deliverables:
- [ ] BaseExecutor interface design (code)
- [ ] DockerComposeExecutor design (code)
- [ ] ExecutorFactory design (code)
- [ ] Architecture diagram (text)

### File Structure:
```
auto-response-agent/app/
  executors/
    __init__.py
    base.py            # BaseExecutor, ExecutionResult
    docker_executor.py # DockerComposeExecutor
    k8s_executor.py    # K8sExecutor (refactored)
    factory.py         # ExecutorFactory
```

### Success Criteria:
- Complete class designs ready to implement
- All method signatures defined
- Clear separation of concerns
- Factory pattern properly designed

---

## Phase 3: Implementation
**Time Estimate:** 2.5 hours
**Goal:** Implement all executor classes and integrate

### Sub-Phase 3.1: Create Module Structure (10 min)

Tasks:
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent/app
mkdir -p executors
touch executors/__init__.py
touch executors/base.py
touch executors/docker_executor.py
touch executors/factory.py
```

- [ ] Create executors directory
- [ ] Create __init__.py with exports
- [ ] Set up module structure

### Sub-Phase 3.2: Implement Base Executor (20 min)
**File:** `app/executors/base.py`

Tasks:
- [ ] Import ABC, abstractmethod
- [ ] Define ActionType enum (RESTART, SCALE, ROLLBACK)
- [ ] Define ExecutionResult class
- [ ] Define BaseExecutor abstract class
- [ ] Add all abstract methods with docstrings
- [ ] Add type hints

Checklist:
- [ ] Code compiles
- [ ] All abstract methods defined
- [ ] Type hints added
- [ ] Docstrings written

### Sub-Phase 3.3: Implement Docker Executor (60 min)
**File:** `app/executors/docker_executor.py`

Tasks:
- [ ] Create DockerComposeExecutor class
- [ ] Implement __init__(compose_file_path)
- [ ] Implement restart() method
  - Validate service exists
  - Run docker-compose restart
  - Handle timeout parameter
  - Return ExecutionResult
- [ ] Implement scale() method
  - Validate replicas parameter
  - Run docker-compose up --scale
  - Return ExecutionResult
- [ ] Implement rollback() method
  - Basic: just call restart
  - Return ExecutionResult
- [ ] Implement health_check() method
  - Test docker-compose --version
  - Return bool
- [ ] Implement get_executor_type()
  - Return "docker-compose"
- [ ] Implement _service_exists() helper
  - Run docker-compose config --services
  - Parse output
  - Check if service in list
- [ ] Implement _run_command() helper
  - Use asyncio.create_subprocess_shell
  - Capture stdout/stderr
  - Return dict with returncode, stdout, stderr, time
- [ ] Add comprehensive logging
- [ ] Add error handling

Checklist:
- [ ] All methods implemented
- [ ] Async/await used correctly
- [ ] Error handling comprehensive
- [ ] Logging statements added
- [ ] Code compiles

### Sub-Phase 3.4: Refactor K8s Executor (30 min)
**File:** Move `app/k8s_executor.py` to `app/executors/k8s_executor.py`

Tasks:
- [ ] Move file to executors folder
- [ ] Import BaseExecutor from .base
- [ ] Make K8sExecutor inherit from BaseExecutor
- [ ] Rename methods to match interface:
  - restart_deployment() → restart()
  - scale_deployment() → scale()
  - rollback_deployment() → rollback()
- [ ] Update method signatures to match BaseExecutor
- [ ] Change return types to ExecutionResult
- [ ] Add health_check() method (test K8s API connection)
- [ ] Add get_executor_type() method (return "kubernetes")
- [ ] Update error handling

Checklist:
- [ ] Inherits from BaseExecutor
- [ ] All methods match interface
- [ ] Returns ExecutionResult objects
- [ ] Code compiles

### Sub-Phase 3.5: Implement Factory (20 min)
**File:** `app/executors/factory.py`

Tasks:
- [ ] Create ExecutorFactory class
- [ ] Implement create_executor() static method
- [ ] Implement _create_docker_executor() helper
- [ ] Implement _create_k8s_executor() helper
- [ ] Implement _auto_detect_executor() helper
- [ ] Add environment variable checks
- [ ] Add logging for detection results

Auto-detection logic:
```python
if os.getenv('EXECUTOR_TYPE') == 'docker-compose':
    return docker
elif os.getenv('EXECUTOR_TYPE') == 'kubernetes':
    return k8s
elif os.getenv('KUBERNETES_SERVICE_HOST'):
    return k8s
elif os.path.exists('~/.kube/config'):
    return k8s
elif os.path.exists('/var/run/docker.sock'):
    return docker
else:
    return docker  # default
```

Checklist:
- [ ] All factory methods implemented
- [ ] Auto-detection works
- [ ] Environment variables respected
- [ ] Logging added

### Sub-Phase 3.6: Update Main Integration (20 min)
**File:** `app/main.py`

Tasks:
- [ ] Backup: `cp app/main.py app/main.py.backup`
- [ ] Replace K8sExecutor import with ExecutorFactory
- [ ] Initialize executor: `executor = ExecutorFactory.create_executor()`
- [ ] Add health check on startup
- [ ] Update all executor calls:
  - executor.restart_deployment() → executor.restart()
  - executor.scale_deployment() → executor.scale()
  - executor.rollback_deployment() → executor.rollback()
- [ ] Handle ExecutionResult objects
- [ ] Publish execution results to RabbitMQ
- [ ] Update logging statements

Checklist:
- [ ] Backup created
- [ ] Factory integration complete
- [ ] All method calls updated
- [ ] ExecutionResult handling added
- [ ] Code compiles

### Sub-Phase 3.7: Update Docker Compose Config (10 min)
**File:** `docker-compose.yml`

Tasks:
- [ ] Add EXECUTOR_TYPE=docker-compose to auto-response-agent env
- [ ] Add DOCKER_COMPOSE_FILE=/compose/docker-compose.yml
- [ ] Mount /var/run/docker.sock volume
- [ ] Mount ./docker-compose.yml:/compose/docker-compose.yml:ro
- [ ] Add security warning comment

```yaml
auto-response-agent:
  environment:
    - EXECUTOR_TYPE=docker-compose
    - DOCKER_COMPOSE_FILE=/compose/docker-compose.yml
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock  # Security: gives full Docker access
    - ./docker-compose.yml:/compose/docker-compose.yml:ro
```

Checklist:
- [ ] Environment variables added
- [ ] Docker socket mounted
- [ ] Compose file mounted
- [ ] Security comment added

### Success Criteria:
- All files created and implemented
- Code compiles without errors
- Backup created
- Docker Compose config updated

---

## Phase 4: Testing
**Time Estimate:** 2 hours
**Goal:** Validate executor works at all levels

### Sub-Phase 4.1: Unit Tests - Docker Executor (40 min)
**File:** `tests/test_docker_executor.py`

Tasks:
- [ ] Create test file
- [ ] Test health_check() returns True
- [ ] Test _service_exists() with valid service
- [ ] Test _service_exists() with invalid service
- [ ] Test restart() with valid service
- [ ] Test restart() with invalid service
- [ ] Test scale() with valid params
- [ ] Test scale() without replicas param
- [ ] Test ExecutionResult objects returned correctly

Run:
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
pytest tests/test_docker_executor.py -v
```

### Sub-Phase 4.2: Unit Tests - Factory (20 min)
**File:** `tests/test_executor_factory.py`

Tasks:
- [ ] Test create_executor() with EXECUTOR_TYPE=docker-compose
- [ ] Test create_executor() with EXECUTOR_TYPE=kubernetes
- [ ] Test auto-detection returns Docker executor
- [ ] Test correct executor type returned

Run:
```bash
pytest tests/test_executor_factory.py -v
```

### Sub-Phase 4.3: Integration Test (20 min)
**File:** `tests/test_integration.py`

Tasks:
- [ ] Test full restart flow with real Docker Compose
- [ ] Test executor health check
- [ ] Test restart actually restarts container
- [ ] Verify ExecutionResult contains expected data

Run:
```bash
python -m tests.test_integration
```

### Sub-Phase 4.4: Docker E2E Test (40 min)

Tasks:
- [ ] Rebuild: `docker-compose build auto-response-agent`
- [ ] Restart: `docker-compose up -d auto-response-agent`
- [ ] Check logs for executor initialization
- [ ] Verify "Using executor: docker-compose" in logs
- [ ] Reset chaos: `curl -X POST http://localhost:8080/reset`
- [ ] Trigger incident: `curl -X POST http://localhost:8080/trigger-memory`
- [ ] Monitor analyzer logs (should generate recommendations)
- [ ] Monitor auto-response logs (should execute actions)
- [ ] Monitor test-app (should restart)
- [ ] Verify chaos cleared after restart

Expected flow:
```
analyzer-agent: ✓ Generated 1 valid recommendation: RESTART test-app
auto-response-agent: Using executor: docker-compose
auto-response-agent: Executing action: RESTART test-app
auto-response-agent: ✓ Successfully restarted test-app in 2.3s
test-app: [container restarted - check with docker-compose ps]
```

Verification:
```bash
# Check test-app restart count increased
docker inspect devops-test-app --format='{{.RestartCount}}'

# Check test-app uptime is recent
docker-compose ps test-app

# Check chaos cleared
curl http://localhost:8080/status | grep memory_leak
# Should show: "memory_leak": false
```

### Success Criteria:
- All unit tests pass
- Integration test passes
- Docker E2E shows auto-healing working
- Test-app actually restarts
- Chaos scenarios cleared after healing

---

## Phase 5: Validation & Documentation
**Time Estimate:** 1 hour
**Goal:** Confirm complete auto-healing works and document

### Sub-Phase 5.1: Comprehensive Validation (30 min)

Test multiple scenarios:

**Scenario 1: Memory Leak**
```bash
curl -X POST http://localhost:8080/trigger-memory
# Wait 30s, watch logs
# Verify: test-app restarts, memory_leak cleared
```

**Scenario 2: CPU Spike**
```bash
curl -X POST http://localhost:8080/trigger-cpu
# Wait 30s, watch logs
# Verify: test-app restarts, cpu_burn cleared
```

**Scenario 3: HTTP Errors**
```bash
curl -X POST http://localhost:8080/trigger-errors
# Wait 30s, watch logs
# Verify: test-app restarts, error_mode cleared
```

Checklist for each scenario:
- [ ] Monitoring agent detects incident
- [ ] Analyzer generates recommendations (count > 0)
- [ ] Auto-response evaluates recommendations
- [ ] Docker executor executes restart
- [ ] Test-app container restarts (verify with docker ps)
- [ ] Chaos scenario cleared
- [ ] No errors in any agent logs

### Sub-Phase 5.2: Success Criteria Verification (15 min)

Check all criteria met:
- [ ] DockerComposeExecutor can restart services
- [ ] DockerComposeExecutor can scale services (basic)
- [ ] ExecutorFactory auto-detects Docker environment
- [ ] Auto-response initializes Docker executor
- [ ] RESTART actions execute successfully
- [ ] Test-app actually restarts (not just logs)
- [ ] Chaos cleared after restart
- [ ] No "K8s client not available" errors
- [ ] ExecutionResult events published to RabbitMQ
- [ ] Complete incident → healing flow works

### Sub-Phase 5.3: Monitor Metrics (15 min)

Check Grafana dashboards (http://localhost:3002):
- [ ] `autoresponse_actions_executed_total` increased
- [ ] `autoresponse_execution_success_total` > 0
- [ ] `autoresponse_execution_failures_total` = 0 (or minimal)
- [ ] Incident resolution time decreased

### Sub-Phase 5.4: Update Documentation (15 min)

**Update:** `implementation-tracker-v1.md`

Add section:
```markdown
## Auto-Healing Implementation - COMPLETE ✅

### Option 1: Action Mapping
- Status: ✅ WORKING
- Analyzer generates 1-3 recommendations per incident
- No more "Could not map action" errors

### Option 2: Docker Executor
- Status: ✅ WORKING
- Auto-response uses DockerComposeExecutor
- Successfully restarts services
- Clears chaos scenarios

### End-to-End Flow: WORKING ✅
test-app chaos → monitoring → analyzer → auto-response → docker restart → chaos cleared
```

**Create:** `services/auto-response-agent/README.md` (if doesn't exist)

Add executor configuration section:
```markdown
## Executor Configuration

### Docker Compose (Local)
```bash
EXECUTOR_TYPE=docker-compose
DOCKER_COMPOSE_FILE=/compose/docker-compose.yml
```

### Kubernetes (Production)
```bash
EXECUTOR_TYPE=kubernetes
```

### Auto-Detection
```bash
EXECUTOR_TYPE=auto
```
```

Checklist:
- [ ] implementation-tracker-v1.md updated
- [ ] Auto-response README updated
- [ ] Architecture documented
- [ ] Configuration options documented

### Rollback Plan:
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
rm -rf app/executors/
cp app/main.py.backup app/main.py
# Restore old k8s_executor.py if backed up
docker-compose build auto-response-agent
docker-compose restart auto-response-agent
```

### Success Criteria:
- All validation scenarios pass
- Metrics show successful executions
- Documentation complete and accurate
- Team can understand and maintain the code

---

## Phase Completion Checklist

Track overall progress:

- [ ] **Phase 1 Complete:** Code discovered and analyzed
- [ ] **Phase 2 Complete:** Architecture designed
- [ ] **Phase 3 Complete:** All code implemented and integrated
- [ ] **Phase 4 Complete:** All tests passing, E2E working
- [ ] **Phase 5 Complete:** Validated and documented

---

## Critical Path

**Sequential requirements:**
1. Phase 1 → Phase 2 (need analysis before design)
2. Phase 2 → Phase 3 (need design before implementation)
3. Phase 3 → Phase 4 (need code before testing)
4. Phase 4 → Phase 5 (need passing tests before validation)

**Hard prerequisite:**
- **Option 1 MUST be complete** before starting Option 2
- If Option 1 isn't done, auto-response will get 0 recommendations and have nothing to execute

---

## Dependencies

### Before Starting Phase 1:
- [ ] Option 1 is complete and validated
- [ ] Analyzer generates recommendations > 0
- [ ] Auto-response-agent is running
- [ ] Docker Compose environment available

### For Phase 3 (Implementation):
- [ ] Docker socket accessible in container
- [ ] docker-compose CLI available
- [ ] Compose file path known

### For Phase 4 (Testing):
- [ ] Test-app running and generating metrics
- [ ] All agents operational
- [ ] Can trigger chaos scenarios

---

## Risk Assessment & Mitigation

### Phase 1-2 Risks:
**Risk:** K8s executor is too tightly coupled
**Mitigation:** Create abstraction layer, don't modify K8s code initially

### Phase 3 Risks:
**Risk:** Docker socket mounting fails
**Mitigation:** Test socket access before full implementation

**Risk:** docker-compose commands fail in container
**Mitigation:** Test _run_command() helper extensively

### Phase 4 Risks:
**Risk:** Restart doesn't actually work
**Mitigation:** Verify with docker inspect, not just logs

**Risk:** Race conditions in async code
**Mitigation:** Proper await usage, test timing

### Phase 5 Risk:
**Risk:** Works in test but fails in real incidents
**Mitigation:** Test multiple chaos scenarios, monitor metrics

---

## Security Considerations

⚠️ **CRITICAL: Docker Socket Access**

Mounting `/var/run/docker.sock` gives **FULL Docker control**:
- Can start/stop ANY container
- Can access host filesystem
- Can escalate privileges
- Can break out of container

**Mitigations:**
1. Only use in trusted dev environments
2. For production, use Kubernetes with RBAC
3. Add action validation (whitelist services)
4. Add approval workflow for risky actions
5. Implement audit logging
6. Consider Docker API with TLS instead of socket

**Document this prominently in README**

---

## Time Budget per Phase

| Phase | Time | Cumulative |
|-------|------|------------|
| Phase 1 | 30 min | 30 min |
| Phase 2 | 1h 0m | 1h 30m |
| Phase 3 | 2h 30m | 4h 0m |
| Phase 4 | 2h 0m | 6h 0m |
| Phase 5 | 1h 0m | 7h 0m |

**Total: 7 hours**

---

## Expected Outcomes

### Before Option 2:
```
auto-response-agent: Processing analysis: 1 recommendation
auto-response-agent: ERROR - K8s client not available
auto-response-agent: ERROR - Failed to execute action
```

### After Phase 5:
```
auto-response-agent: Using executor: docker-compose
auto-response-agent: Executor health check: True
auto-response-agent: Processing analysis: 1 recommendation
auto-response-agent: Evaluating: RESTART test-app
auto-response-agent: ✓ Decision: Execute automatically (confidence: 85%)
auto-response-agent: Executing: RESTART test-app via docker-compose
auto-response-agent: Running: docker-compose restart test-app
auto-response-agent: ✓ Successfully restarted test-app in 2.3s
auto-response-agent: 📤 Published action_executed event

test-app: Container restarted (docker-compose ps shows recent uptime)
test-app: Health check: PASSING
test-app: Chaos cleared: memory_leak=false
```

---

## Final Success Definition

**Option 2 is COMPLETE when:**

1. ✅ Auto-response-agent uses DockerComposeExecutor
2. ✅ Executor can restart Docker Compose services
3. ✅ Triggering chaos → incident → analysis → auto-healing → chaos cleared
4. ✅ Test-app container actually restarts (verified with docker inspect)
5. ✅ No K8s errors in logs
6. ✅ All tests pass
7. ✅ Documentation complete

**Combined with Option 1:**
- ✅ Analyzer generates recommendations
- ✅ Auto-response executes recommendations
- ✅ **Complete autonomous incident response working!** 🎉

---

## Post-Completion

After both Option 1 and Option 2 are complete:

1. Test multiple incident types
2. Measure MTTR (Mean Time To Recovery)
3. Add more action types if needed
4. Tune confidence thresholds
5. Add approval workflows for critical services
6. Consider migrating to local K8s (kind/minikube) for full features
7. Add incident learning feedback loop
8. Implement auto-resolution verification
