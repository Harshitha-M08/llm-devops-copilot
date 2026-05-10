# Fixing Option 2: Docker Compose Executor Support

## Context: What's NOT Happening

### Current Problem
The **Auto-Response Agent** has a `K8sExecutor` that can only execute Kubernetes actions (scale pods, restart deployments, rollback). However, we're running the system locally using **Docker Compose**, NOT Kubernetes.

Even if we fix Option 1 and get recommendations working, the auto-response agent **cannot execute them** because:
1. There's no Kubernetes cluster available locally
2. The K8sExecutor tries to connect to K8s API and fails
3. Docker Compose services need different commands (`docker-compose restart`, not `kubectl restart`)

### Evidence from Logs
```
monitoring-agent  | ERROR - Error checking pod health: K8s client not available
```

This error appears repeatedly because the agents are trying to use Kubernetes APIs in a Docker Compose environment.

### Root Cause
The system was architected for **Kubernetes production deployments**, where:
- Services run as K8s deployments/pods
- Actions are executed via Kubernetes API (kubectl)
- Scaling = `kubectl scale deployment api-service --replicas=3`
- Restart = `kubectl rollout restart deployment api-service`

But in **Docker Compose local environment**:
- Services run as Docker containers
- Actions must use Docker Compose CLI or Docker API
- Scaling = `docker-compose up -d --scale api-service=3`
- Restart = `docker-compose restart api-service`

### Impact
Even when recommendations are generated (after Option 1 fix):
- ❌ Auto-Response Agent cannot execute RESTART actions
- ❌ Auto-Response Agent cannot execute SCALE actions
- ❌ Auto-Response Agent cannot execute ROLLBACK actions
- ✅ It can still create approval requests (stored in database)
- ✅ It can still publish events to RabbitMQ

**Result: No automated healing in Docker Compose environments**

---

## Investigation Plan

### Phase 1: Understand Current K8s Executor (30 minutes)

#### Step 1.1: Locate K8s Executor Code
**File to examine:** `services/auto-response-agent/app/k8s_executor.py` or similar

**What to look for:**
- How it connects to Kubernetes
- What methods it exposes (restart, scale, rollback)
- What parameters each action requires
- Error handling for K8s connection failures

**Commands to run:**
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
find . -name "*k8s*.py" -o -name "*executor*.py"
grep -r "kubectl\|kubernetes\|k8s" app/ -n
grep -r "class.*Executor" app/ -n
```

#### Step 1.2: Examine Auto-Response Agent Integration
**File to examine:** `services/auto-response-agent/app/main.py`

**What to look for:**
- How K8sExecutor is instantiated
- How recommendations are passed to executor
- What happens when executor fails
- Whether there's an executor abstraction/interface

**Questions to answer:**
1. Is there an `Executor` base class or interface?
2. Does the code check which executor to use (K8s vs Docker)?
3. How are executor failures handled?
4. Can we run multiple executors simultaneously?

**Commands to run:**
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
grep -r "K8sExecutor\|Executor" app/ -n -A 5 -B 5
grep -r "def execute\|async def execute" app/ -n
```

#### Step 1.3: Check Configuration
**Files to examine:**
- `services/auto-response-agent/.env`
- `services/auto-response-agent/app/config.py`
- `docker-compose.yml` (auto-response-agent service section)

**What to look for:**
- Environment variables for executor selection
- Configuration for K8s cluster connection
- Any flags for local vs production mode

**Commands to run:**
```bash
cd LLM DevOps Copilot-main
grep -E "K8S|KUBERNETES|EXECUTOR" .env services/auto-response-agent/.env 2>/dev/null
grep -A 20 "auto-response-agent:" docker-compose.yml
```

---

### Phase 2: Design Docker Compose Executor (1 hour)

#### Step 2.1: Define Executor Interface

**Goal:** Create a common interface that both K8sExecutor and DockerExecutor implement

**File to create:** `services/auto-response-agent/app/executors/base.py`

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Optional
from enum import Enum

class ActionType(str, Enum):
    RESTART = "restart"
    SCALE = "scale"
    ROLLBACK = "rollback"

class ExecutionResult:
    """Result of an execution action"""
    def __init__(
        self,
        success: bool,
        message: str,
        action_type: ActionType,
        target_service: str,
        details: Optional[Dict[str, Any]] = None
    ):
        self.success = success
        self.message = message
        self.action_type = action_type
        self.target_service = target_service
        self.details = details or {}

class BaseExecutor(ABC):
    """Base class for all executors"""

    @abstractmethod
    async def restart(self, target_service: str, target_type: str, parameters: Dict[str, Any]) -> ExecutionResult:
        """Restart a service"""
        pass

    @abstractmethod
    async def scale(self, target_service: str, target_type: str, parameters: Dict[str, Any]) -> ExecutionResult:
        """Scale a service"""
        pass

    @abstractmethod
    async def rollback(self, target_service: str, target_type: str, parameters: Dict[str, Any]) -> ExecutionResult:
        """Rollback a service to previous version"""
        pass

    @abstractmethod
    async def health_check(self) -> bool:
        """Check if executor is available and healthy"""
        pass

    @abstractmethod
    def get_executor_type(self) -> str:
        """Return executor type (k8s, docker, etc.)"""
        pass
```

#### Step 2.2: Design Docker Compose Executor

**File to create:** `services/auto-response-agent/app/executors/docker_executor.py`

```python
import asyncio
import subprocess
import logging
from typing import Dict, Any
from .base import BaseExecutor, ExecutionResult, ActionType

logger = logging.getLogger(__name__)

class DockerComposeExecutor(BaseExecutor):
    """Executor for Docker Compose environments"""

    def __init__(self, compose_file_path: str = "/app/docker-compose.yml"):
        self.compose_file_path = compose_file_path
        self.compose_command = f"docker-compose -f {compose_file_path}"
        logger.info(f"Initialized DockerComposeExecutor with compose file: {compose_file_path}")

    async def restart(self, target_service: str, target_type: str, parameters: Dict[str, Any]) -> ExecutionResult:
        """
        Restart a Docker Compose service

        Args:
            target_service: Service name (e.g., 'test-app')
            target_type: Not used in Docker Compose (kept for interface compatibility)
            parameters: Optional parameters (e.g., {'timeout': 30})
        """
        try:
            logger.info(f"Restarting service: {target_service}")

            # Validate service exists
            if not await self._service_exists(target_service):
                return ExecutionResult(
                    success=False,
                    message=f"Service '{target_service}' not found in docker-compose.yml",
                    action_type=ActionType.RESTART,
                    target_service=target_service
                )

            # Execute restart
            timeout = parameters.get('timeout', 30)
            cmd = f"{self.compose_command} restart -t {timeout} {target_service}"

            result = await self._run_command(cmd)

            if result['returncode'] == 0:
                logger.info(f"✓ Successfully restarted {target_service}")
                return ExecutionResult(
                    success=True,
                    message=f"Successfully restarted {target_service}",
                    action_type=ActionType.RESTART,
                    target_service=target_service,
                    details={
                        'stdout': result['stdout'],
                        'execution_time': result['execution_time']
                    }
                )
            else:
                logger.error(f"Failed to restart {target_service}: {result['stderr']}")
                return ExecutionResult(
                    success=False,
                    message=f"Failed to restart: {result['stderr']}",
                    action_type=ActionType.RESTART,
                    target_service=target_service,
                    details={'stderr': result['stderr']}
                )

        except Exception as e:
            logger.error(f"Exception during restart: {e}")
            return ExecutionResult(
                success=False,
                message=f"Exception: {str(e)}",
                action_type=ActionType.RESTART,
                target_service=target_service
            )

    async def scale(self, target_service: str, target_type: str, parameters: Dict[str, Any]) -> ExecutionResult:
        """
        Scale a Docker Compose service

        Args:
            target_service: Service name
            target_type: Not used in Docker Compose
            parameters: Must include 'replicas' (e.g., {'replicas': 3})
        """
        try:
            replicas = parameters.get('replicas')
            if replicas is None:
                return ExecutionResult(
                    success=False,
                    message="Missing required parameter: 'replicas'",
                    action_type=ActionType.SCALE,
                    target_service=target_service
                )

            logger.info(f"Scaling service {target_service} to {replicas} replicas")

            # Validate service exists
            if not await self._service_exists(target_service):
                return ExecutionResult(
                    success=False,
                    message=f"Service '{target_service}' not found in docker-compose.yml",
                    action_type=ActionType.SCALE,
                    target_service=target_service
                )

            # Execute scale
            cmd = f"{self.compose_command} up -d --scale {target_service}={replicas} --no-recreate"

            result = await self._run_command(cmd)

            if result['returncode'] == 0:
                logger.info(f"✓ Successfully scaled {target_service} to {replicas} replicas")
                return ExecutionResult(
                    success=True,
                    message=f"Successfully scaled {target_service} to {replicas} replicas",
                    action_type=ActionType.SCALE,
                    target_service=target_service,
                    details={
                        'replicas': replicas,
                        'stdout': result['stdout'],
                        'execution_time': result['execution_time']
                    }
                )
            else:
                logger.error(f"Failed to scale {target_service}: {result['stderr']}")
                return ExecutionResult(
                    success=False,
                    message=f"Failed to scale: {result['stderr']}",
                    action_type=ActionType.SCALE,
                    target_service=target_service,
                    details={'stderr': result['stderr']}
                )

        except Exception as e:
            logger.error(f"Exception during scale: {e}")
            return ExecutionResult(
                success=False,
                message=f"Exception: {str(e)}",
                action_type=ActionType.SCALE,
                target_service=target_service
            )

    async def rollback(self, target_service: str, target_type: str, parameters: Dict[str, Any]) -> ExecutionResult:
        """
        Rollback is not directly supported in Docker Compose
        We'll simulate by pulling previous image tag if specified

        Args:
            target_service: Service name
            target_type: Not used
            parameters: Should include 'image_tag' (e.g., {'image_tag': 'v1.2.3'})
        """
        try:
            logger.warning(f"Rollback for {target_service} - Docker Compose has limited rollback support")

            # For now, just restart the service (basic rollback simulation)
            # In production, you'd update image tag in compose file and redeploy
            return await self.restart(target_service, target_type, parameters)

        except Exception as e:
            logger.error(f"Exception during rollback: {e}")
            return ExecutionResult(
                success=False,
                message=f"Exception: {str(e)}",
                action_type=ActionType.ROLLBACK,
                target_service=target_service
            )

    async def health_check(self) -> bool:
        """Check if docker-compose is available"""
        try:
            cmd = "docker-compose --version"
            result = await self._run_command(cmd)
            return result['returncode'] == 0
        except Exception as e:
            logger.error(f"Health check failed: {e}")
            return False

    def get_executor_type(self) -> str:
        return "docker-compose"

    async def _service_exists(self, service_name: str) -> bool:
        """Check if service exists in docker-compose.yml"""
        try:
            cmd = f"{self.compose_command} config --services"
            result = await self._run_command(cmd)

            if result['returncode'] == 0:
                services = result['stdout'].strip().split('\n')
                return service_name in services

            return False
        except Exception:
            return False

    async def _run_command(self, cmd: str) -> Dict[str, Any]:
        """
        Run a shell command asynchronously

        Returns:
            Dict with returncode, stdout, stderr, execution_time
        """
        import time
        start_time = time.time()

        logger.debug(f"Executing command: {cmd}")

        process = await asyncio.create_subprocess_shell(
            cmd,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )

        stdout, stderr = await process.communicate()
        execution_time = time.time() - start_time

        return {
            'returncode': process.returncode,
            'stdout': stdout.decode('utf-8'),
            'stderr': stderr.decode('utf-8'),
            'execution_time': execution_time
        }
```

#### Step 2.3: Update K8s Executor to Use Interface

**File to modify:** `services/auto-response-agent/app/executors/k8s_executor.py`

Make it inherit from `BaseExecutor` and implement all abstract methods with the same signatures.

#### Step 2.4: Design Executor Factory

**File to create:** `services/auto-response-agent/app/executors/factory.py`

```python
import os
import logging
from typing import Optional
from .base import BaseExecutor
from .k8s_executor import K8sExecutor
from .docker_executor import DockerComposeExecutor

logger = logging.getLogger(__name__)

class ExecutorFactory:
    """Factory for creating appropriate executor based on environment"""

    @staticmethod
    def create_executor() -> BaseExecutor:
        """
        Create executor based on environment configuration

        Environment variables:
            EXECUTOR_TYPE: 'kubernetes' or 'docker-compose' (default: auto-detect)
            DOCKER_COMPOSE_FILE: Path to docker-compose.yml (for docker executor)
            KUBECONFIG: Path to kubeconfig (for k8s executor)
        """
        executor_type = os.getenv('EXECUTOR_TYPE', 'auto').lower()

        if executor_type == 'docker-compose':
            return ExecutorFactory._create_docker_executor()
        elif executor_type == 'kubernetes' or executor_type == 'k8s':
            return ExecutorFactory._create_k8s_executor()
        elif executor_type == 'auto':
            return ExecutorFactory._auto_detect_executor()
        else:
            logger.warning(f"Unknown executor type: {executor_type}, falling back to auto-detect")
            return ExecutorFactory._auto_detect_executor()

    @staticmethod
    def _create_docker_executor() -> DockerComposeExecutor:
        """Create Docker Compose executor"""
        compose_file = os.getenv('DOCKER_COMPOSE_FILE', '/compose/docker-compose.yml')
        logger.info(f"Creating DockerComposeExecutor with file: {compose_file}")
        return DockerComposeExecutor(compose_file_path=compose_file)

    @staticmethod
    def _create_k8s_executor() -> K8sExecutor:
        """Create Kubernetes executor"""
        logger.info("Creating K8sExecutor")
        return K8sExecutor()

    @staticmethod
    def _auto_detect_executor() -> BaseExecutor:
        """
        Auto-detect which executor to use based on environment

        Detection logic:
        1. Check if KUBERNETES_SERVICE_HOST env var exists (running in K8s)
        2. Check if /var/run/docker.sock exists (Docker available)
        3. Check if kubeconfig exists
        4. Default to Docker Compose
        """
        # Check if running inside Kubernetes
        if os.getenv('KUBERNETES_SERVICE_HOST'):
            logger.info("Auto-detected: Running in Kubernetes cluster")
            return ExecutorFactory._create_k8s_executor()

        # Check if kubeconfig exists (can connect to K8s)
        kubeconfig_path = os.getenv('KUBECONFIG', os.path.expanduser('~/.kube/config'))
        if os.path.exists(kubeconfig_path):
            logger.info(f"Auto-detected: Found kubeconfig at {kubeconfig_path}")
            return ExecutorFactory._create_k8s_executor()

        # Check if docker.sock exists (Docker available)
        if os.path.exists('/var/run/docker.sock'):
            logger.info("Auto-detected: Docker available, using Docker Compose executor")
            return ExecutorFactory._create_docker_executor()

        # Default to Docker Compose
        logger.info("Auto-detected: Defaulting to Docker Compose executor")
        return ExecutorFactory._create_docker_executor()
```

---

### Phase 3: Implementation (2.5 hours)

#### Step 3.1: Create Executor Module Structure

```bash
cd LLM DevOps Copilot-main/services/auto-response-agent/app
mkdir -p executors
touch executors/__init__.py
```

**File:** `executors/__init__.py`
```python
from .base import BaseExecutor, ExecutionResult, ActionType
from .factory import ExecutorFactory

__all__ = ['BaseExecutor', 'ExecutionResult', 'ActionType', 'ExecutorFactory']
```

#### Step 3.2: Implement Base Executor

**File:** `services/auto-response-agent/app/executors/base.py`

1. Copy the interface design from Phase 2.1
2. Add comprehensive docstrings
3. Add type hints

**Implementation checklist:**
- [ ] Create base.py file
- [ ] Define ActionType enum
- [ ] Define ExecutionResult class
- [ ] Define BaseExecutor abstract class
- [ ] Add all abstract methods
- [ ] Add docstrings

#### Step 3.3: Implement Docker Executor

**File:** `services/auto-response-agent/app/executors/docker_executor.py`

1. Copy the implementation from Phase 2.2
2. Add error handling for all edge cases
3. Add logging
4. Add command validation

**Implementation checklist:**
- [ ] Create docker_executor.py file
- [ ] Implement DockerComposeExecutor class
- [ ] Implement restart() method
- [ ] Implement scale() method
- [ ] Implement rollback() method (basic version)
- [ ] Implement health_check() method
- [ ] Implement _service_exists() helper
- [ ] Implement _run_command() helper
- [ ] Add comprehensive logging
- [ ] Handle all exceptions

#### Step 3.4: Refactor Existing K8s Executor

**File:** `services/auto-response-agent/app/k8s_executor.py`

**Move to:** `services/auto-response-agent/app/executors/k8s_executor.py`

**Changes needed:**
1. Import BaseExecutor from .base
2. Make K8sExecutor inherit from BaseExecutor
3. Update method signatures to match interface
4. Return ExecutionResult objects
5. Add health_check() and get_executor_type() methods

**Example changes:**
```python
# OLD
class K8sExecutor:
    def restart_deployment(self, deployment_name: str):
        # ... implementation
        return True

# NEW
from .base import BaseExecutor, ExecutionResult, ActionType

class K8sExecutor(BaseExecutor):
    async def restart(self, target_service: str, target_type: str, parameters: Dict[str, Any]) -> ExecutionResult:
        try:
            # ... existing restart logic
            # Instead of returning True, return ExecutionResult
            return ExecutionResult(
                success=True,
                message=f"Successfully restarted {target_service}",
                action_type=ActionType.RESTART,
                target_service=target_service
            )
        except Exception as e:
            return ExecutionResult(
                success=False,
                message=str(e),
                action_type=ActionType.RESTART,
                target_service=target_service
            )
```

**Implementation checklist:**
- [ ] Move k8s_executor.py to executors/ folder
- [ ] Import BaseExecutor
- [ ] Update class declaration to inherit BaseExecutor
- [ ] Rename methods to match interface (restart_deployment → restart)
- [ ] Update return types to ExecutionResult
- [ ] Add health_check() method (test K8s API connection)
- [ ] Add get_executor_type() method (return "kubernetes")
- [ ] Update all error handling

#### Step 3.5: Implement Executor Factory

**File:** `services/auto-response-agent/app/executors/factory.py`

1. Copy the factory implementation from Phase 2.4
2. Test auto-detection logic
3. Add fallback mechanisms

**Implementation checklist:**
- [ ] Create factory.py file
- [ ] Implement ExecutorFactory class
- [ ] Implement create_executor() method
- [ ] Implement _create_docker_executor() method
- [ ] Implement _create_k8s_executor() method
- [ ] Implement _auto_detect_executor() method
- [ ] Add environment variable checks
- [ ] Add logging for detection results

#### Step 3.6: Update Main Auto-Response Agent

**File:** `services/auto-response-agent/app/main.py`

**Changes needed:**

```python
# OLD
from app.k8s_executor import K8sExecutor

executor = K8sExecutor()

async def process_recommendation(rec):
    if rec.action_type == "restart":
        result = executor.restart_deployment(rec.target_service)

# NEW
from app.executors import ExecutorFactory, ActionType

# Initialize executor using factory
executor = ExecutorFactory.create_executor()
logger.info(f"Using executor: {executor.get_executor_type()}")

# Check executor health
if not await executor.health_check():
    logger.error(f"Executor {executor.get_executor_type()} is not healthy!")

async def process_recommendation(rec):
    """Process a single recommendation"""
    if rec.action_type == ActionType.RESTART:
        result = await executor.restart(
            target_service=rec.target_service,
            target_type=rec.target_type,
            parameters=rec.parameters
        )

        if result.success:
            logger.info(f"✓ {result.message}")
            await publish_action_executed(rec, result)
        else:
            logger.error(f"✗ {result.message}")
            await publish_action_failed(rec, result)

    elif rec.action_type == ActionType.SCALE:
        result = await executor.scale(
            target_service=rec.target_service,
            target_type=rec.target_type,
            parameters=rec.parameters
        )
        # ... handle result

    elif rec.action_type == ActionType.ROLLBACK:
        result = await executor.rollback(
            target_service=rec.target_service,
            target_type=rec.target_type,
            parameters=rec.parameters
        )
        # ... handle result
```

**Implementation checklist:**
- [ ] Replace K8sExecutor import with ExecutorFactory
- [ ] Initialize executor using factory at startup
- [ ] Add executor health check on startup
- [ ] Update all recommendation processing to use new interface
- [ ] Update method calls (restart_deployment → restart)
- [ ] Handle ExecutionResult objects
- [ ] Add logging for execution results
- [ ] Publish success/failure events

#### Step 3.7: Update Docker Compose Configuration

**File:** `docker-compose.yml`

**Changes to auto-response-agent service:**

```yaml
auto-response-agent:
  # ... existing config ...
  environment:
    # ... existing env vars ...
    - EXECUTOR_TYPE=docker-compose  # Force Docker executor
    - DOCKER_COMPOSE_FILE=/compose/docker-compose.yml
  volumes:
    # Mount docker socket so agent can run docker-compose commands
    - /var/run/docker.sock:/var/run/docker.sock
    # Mount compose file so agent can read service definitions
    - ./docker-compose.yml:/compose/docker-compose.yml:ro
```

**IMPORTANT: Security consideration**
Mounting `/var/run/docker.sock` gives the container full Docker access. This is necessary for executing docker-compose commands, but should only be done in trusted environments.

**Implementation checklist:**
- [ ] Add EXECUTOR_TYPE environment variable
- [ ] Add DOCKER_COMPOSE_FILE environment variable
- [ ] Mount docker socket
- [ ] Mount docker-compose.yml file
- [ ] Document security implications

#### Step 3.8: Update Requirements

**File:** `services/auto-response-agent/requirements.txt`

Ensure async support:
```
asyncio
```

---

### Phase 4: Testing (2 hours)

#### Step 4.1: Unit Tests for Docker Executor

**File:** `services/auto-response-agent/tests/test_docker_executor.py`

```python
import pytest
import asyncio
from app.executors.docker_executor import DockerComposeExecutor
from app.executors.base import ActionType

@pytest.fixture
def executor():
    return DockerComposeExecutor(compose_file_path="./docker-compose.yml")

@pytest.mark.asyncio
async def test_health_check(executor):
    """Test executor health check"""
    result = await executor.health_check()
    assert result == True

@pytest.mark.asyncio
async def test_service_exists(executor):
    """Test service existence check"""
    result = await executor._service_exists("test-app")
    assert result == True

    result = await executor._service_exists("nonexistent-service")
    assert result == False

@pytest.mark.asyncio
async def test_restart_service(executor):
    """Test restarting a service"""
    result = await executor.restart(
        target_service="test-app",
        target_type="service",
        parameters={}
    )

    assert result.success == True
    assert result.action_type == ActionType.RESTART
    assert "test-app" in result.message

@pytest.mark.asyncio
async def test_restart_nonexistent_service(executor):
    """Test restarting a service that doesn't exist"""
    result = await executor.restart(
        target_service="fake-service",
        target_type="service",
        parameters={}
    )

    assert result.success == False
    assert "not found" in result.message.lower()

@pytest.mark.asyncio
async def test_scale_service(executor):
    """Test scaling a service"""
    result = await executor.scale(
        target_service="test-app",
        target_type="service",
        parameters={"replicas": 2}
    )

    # Note: Scale may not work for services without replicas configured
    # Adjust assertion based on your docker-compose.yml
    assert result.action_type == ActionType.SCALE

@pytest.mark.asyncio
async def test_scale_without_replicas_param(executor):
    """Test scaling without replicas parameter"""
    result = await executor.scale(
        target_service="test-app",
        target_type="service",
        parameters={}
    )

    assert result.success == False
    assert "replicas" in result.message.lower()
```

**Run tests:**
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
pytest tests/test_docker_executor.py -v
```

#### Step 4.2: Unit Tests for Executor Factory

**File:** `services/auto-response-agent/tests/test_executor_factory.py`

```python
import pytest
import os
from app.executors.factory import ExecutorFactory
from app.executors.docker_executor import DockerComposeExecutor
from app.executors.k8s_executor import K8sExecutor

def test_create_docker_executor_explicit():
    """Test creating Docker executor explicitly"""
    os.environ['EXECUTOR_TYPE'] = 'docker-compose'

    executor = ExecutorFactory.create_executor()

    assert isinstance(executor, DockerComposeExecutor)
    assert executor.get_executor_type() == "docker-compose"

def test_create_k8s_executor_explicit():
    """Test creating K8s executor explicitly"""
    os.environ['EXECUTOR_TYPE'] = 'kubernetes'

    executor = ExecutorFactory.create_executor()

    assert isinstance(executor, K8sExecutor)
    assert executor.get_executor_type() == "kubernetes"

def test_auto_detect_docker():
    """Test auto-detection of Docker executor"""
    os.environ['EXECUTOR_TYPE'] = 'auto'
    # Ensure KUBERNETES_SERVICE_HOST is not set
    os.environ.pop('KUBERNETES_SERVICE_HOST', None)

    executor = ExecutorFactory.create_executor()

    # In Docker Compose environment, should default to Docker executor
    assert isinstance(executor, DockerComposeExecutor)
```

**Run tests:**
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
pytest tests/test_executor_factory.py -v
```

#### Step 4.3: Integration Test with Real Docker Compose

**File:** `services/auto-response-agent/tests/test_integration.py`

```python
import pytest
import asyncio
from app.executors import ExecutorFactory
from app.executors.base import ActionType

@pytest.mark.asyncio
async def test_end_to_end_restart():
    """Test full restart flow"""
    # Create executor
    executor = ExecutorFactory.create_executor()

    print(f"\nUsing executor: {executor.get_executor_type()}")

    # Health check
    is_healthy = await executor.health_check()
    assert is_healthy, "Executor is not healthy"
    print("✓ Executor is healthy")

    # Restart test-app
    result = await executor.restart(
        target_service="test-app",
        target_type="service",
        parameters={"timeout": 10}
    )

    print(f"Restart result: {result.message}")
    assert result.success, f"Restart failed: {result.message}"
    assert result.action_type == ActionType.RESTART

    print("✓ Service restarted successfully")

if __name__ == "__main__":
    asyncio.run(test_end_to_end_restart())
```

**Run integration test:**
```bash
cd LLM DevOps Copilot-main/services/auto-response-agent
python -m tests.test_integration
```

#### Step 4.4: End-to-End Test with Full System

**Step 1: Rebuild auto-response-agent**
```bash
cd LLM DevOps Copilot-main
docker-compose build auto-response-agent
docker-compose up -d auto-response-agent
```

**Step 2: Check logs for executor initialization**
```bash
docker-compose logs auto-response-agent | grep -E "executor|Executor"
```

**Expected output:**
```
auto-response-agent: INFO - Using executor: docker-compose
auto-response-agent: INFO - Executor health check: True
```

**Step 3: Trigger incident**
```bash
# Reset previous chaos
curl -X POST http://localhost:8080/reset

# Trigger new memory leak
curl -X POST http://localhost:8080/trigger-memory
```

**Step 4: Monitor the full flow**

**Terminal 1: Watch analyzer**
```bash
docker-compose logs -f analyzer-agent | grep -E "recommendation|Analysis"
```

**Terminal 2: Watch auto-response**
```bash
docker-compose logs -f auto-response-agent | grep -E "recommendation|Executing|result"
```

**Terminal 3: Watch test-app status**
```bash
watch -n 2 "docker-compose ps test-app && curl -s http://localhost:8080/status"
```

**Expected successful flow:**
```
analyzer-agent: ✓ Generated 1 valid recommendation
analyzer-agent: 📊 Analysis published: Memory leak | Recommendations: 1

auto-response-agent: Processing analysis: 1 recommendation
auto-response-agent: Evaluating recommendation: RESTART test-app
auto-response-agent: ✓ Decision: Execute automatically (confidence: 85%)
auto-response-agent: Executing action: RESTART test-app
auto-response-agent: ✓ Successfully restarted test-app
auto-response-agent: 📤 Published action executed event

test-app: Container restarted (visible in docker-compose ps)
```

#### Step 4.5: Verify Auto-Healing Worked

**Check test-app was actually restarted:**
```bash
# Check container restart count increased
docker inspect devops-test-app --format='{{.RestartCount}}'

# Check container uptime is recent
docker-compose ps test-app
# Look for "Up X seconds" instead of "Up X hours"
```

**Check incident was resolved:**
```bash
# Chaos should be cleared after restart
curl http://localhost:8080/status
# Should show: "memory_leak": false
```

---

### Phase 5: Validation & Documentation (1 hour)

#### Step 5.1: Success Criteria Checklist

- [ ] DockerComposeExecutor can restart services
- [ ] DockerComposeExecutor can scale services (if supported)
- [ ] ExecutorFactory correctly auto-detects environment
- [ ] Auto-response-agent initializes Docker executor in Docker Compose
- [ ] Auto-response-agent successfully executes RESTART actions
- [ ] Test-app actually restarts when action is executed
- [ ] Chaos scenarios are cleared after restart
- [ ] No "K8s client not available" errors in auto-response logs
- [ ] ExecutionResult events are published to RabbitMQ
- [ ] End-to-end incident flow completes with auto-healing

#### Step 5.2: Update Documentation

**File:** `services/auto-response-agent/README.md`

Add section on executor configuration:

```markdown
## Executor Configuration

The auto-response agent supports multiple execution backends:

### Docker Compose (Local Development)
```bash
EXECUTOR_TYPE=docker-compose
DOCKER_COMPOSE_FILE=/compose/docker-compose.yml
```

Mount requirements:
- `/var/run/docker.sock` - Docker API access
- `/compose/docker-compose.yml` - Service definitions

### Kubernetes (Production)
```bash
EXECUTOR_TYPE=kubernetes
KUBECONFIG=/path/to/kubeconfig
```

### Auto-Detection
```bash
EXECUTOR_TYPE=auto
```

Auto-detection logic:
1. Checks for `KUBERNETES_SERVICE_HOST` (running in K8s)
2. Checks for `~/.kube/config` (K8s cluster access)
3. Checks for `/var/run/docker.sock` (Docker available)
4. Defaults to Docker Compose
```

#### Step 5.3: Update Implementation Tracker

**File:** `implementation-tracker-v1.md`

Add new section:

```markdown
## Auto-Healing Implementation

### Status: ✅ WORKING (Docker Compose)

### Changes Made

1. **Created Executor Abstraction**
   - Base executor interface: `executors/base.py`
   - Allows multiple executor implementations

2. **Implemented Docker Compose Executor**
   - File: `executors/docker_executor.py`
   - Supports: restart, scale (basic rollback)
   - Uses docker-compose CLI commands

3. **Created Executor Factory**
   - File: `executors/factory.py`
   - Auto-detects environment (K8s vs Docker)
   - Configurable via EXECUTOR_TYPE env var

4. **Updated Auto-Response Agent**
   - Uses ExecutorFactory instead of hardcoded K8sExecutor
   - Handles ExecutionResult objects
   - Publishes execution success/failure events

### Supported Actions

| Action | Docker Compose | Kubernetes |
|--------|---------------|------------|
| RESTART | ✅ Full support | ✅ Full support |
| SCALE | ⚠️ Basic support | ✅ Full support |
| ROLLBACK | ⚠️ Limited | ✅ Full support |

### Testing Results

- ✅ Restart test-app: Working
- ✅ Auto-healing on memory leak: Working
- ✅ Executor auto-detection: Working
- ✅ Health checks: Working
```

---

## Expected Outcome

### Before Fix
```
auto-response-agent: INFO - Processing analysis: 1 recommendation
auto-response-agent: ERROR - K8s client not available
auto-response-agent: ERROR - Failed to execute RESTART action
```

### After Fix
```
auto-response-agent: INFO - Using executor: docker-compose
auto-response-agent: INFO - Executor health check: True
auto-response-agent: INFO - Processing analysis: 1 recommendation
auto-response-agent: INFO - Evaluating recommendation: RESTART test-app
auto-response-agent: INFO - ✓ Decision: Execute automatically (confidence: 85%)
auto-response-agent: INFO - Executing action: RESTART test-app via docker-compose
auto-response-agent: INFO - ✓ Successfully restarted test-app in 2.3 seconds
auto-response-agent: INFO - 📤 Published action_executed event

test-app: Container restarted
test-app: Chaos scenarios cleared
test-app: Service healthy
```

---

## Timeline Estimate

- **Phase 1 (Investigation):** 30 minutes
- **Phase 2 (Design):** 1 hour
- **Phase 3 (Implementation):** 2.5 hours
- **Phase 4 (Testing):** 2 hours
- **Phase 5 (Validation):** 1 hour

**Total: ~7 hours**

---

## Dependencies

- Option 1 must be completed first (action mapping must work)
- Docker socket access in container
- docker-compose CLI available in auto-response-agent container

---

## Security Considerations

⚠️ **Mounting `/var/run/docker.sock` gives full Docker access**

**Risks:**
- Container can start/stop any container
- Container can access host filesystem via volume mounts
- Container can escalate privileges

**Mitigations:**
- Only use in trusted development environments
- For production, use Kubernetes with RBAC instead
- Add action validation and approval workflows
- Limit which services can be restarted/scaled
- Add audit logging for all actions

---

## Limitations

### Docker Compose Executor Limitations

1. **Rollback:** Docker Compose doesn't have native rollback. Current implementation just restarts the service.

2. **Scaling:** Not all services support scaling in Docker Compose. Services must be configured without fixed ports.

3. **No Blue-Green Deployments:** Docker Compose doesn't support advanced deployment strategies.

4. **Single Host:** Docker Compose is single-host, no distributed orchestration.

**Recommendation:** Use Kubernetes for production deployments to get full auto-healing capabilities.

---

## Next Steps After This Fix

1. ✅ Verify Option 1 (action mapping) is working
2. ✅ Implement Option 2 (Docker executor)
3. Test combined flow end-to-end
4. Add approval workflow for high-risk actions
5. Add action validation (prevent dangerous operations)
6. Add rollback capability (track previous states)
7. Consider migrating to local Kubernetes (kind/minikube) for full feature testing
8. Add metrics and monitoring for auto-healing actions
9. Implement incident auto-resolution logic
10. Add learning feedback loop (did the action fix the issue?)
