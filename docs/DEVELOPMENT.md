# Development Guide - AI-Driven Hybrid Kubernetes System

## Table of Contents
1. [Getting Started](#getting-started)
2. [Local Development Setup](#local-development-setup)
3. [Running Services Locally](#running-services-locally)
4. [Testing Guidelines](#testing-guidelines)
5. [Code Standards](#code-standards)
6. [Git Workflow](#git-workflow)
7. [Contributing Guidelines](#contributing-guidelines)
8. [Debugging Tips](#debugging-tips)
9. [IDE Setup](#ide-setup)

---

## Getting Started

### Prerequisites

Before starting development, ensure you have the following installed:

#### Required Tools
- **Git**: Version control (2.30+)
- **Python**: 3.11 or higher
- **Node.js**: 18 LTS or higher
- **Docker**: 20.10+ and Docker Compose
- **PostgreSQL**: 15+ (or use Docker)
- **Redis**: 7+ (or use Docker)
- **RabbitMQ**: 3.12+ (or use Docker)

#### Optional Tools
- **pyenv**: Python version management
- **nvm**: Node.js version management
- **kubectl**: For local Kubernetes testing
- **minikube** or **kind**: Local Kubernetes cluster
- **Postman** or **Insomnia**: API testing

### Clone Repository

```bash
# Clone the repository
git clone https://github.com/your-org/ai-system.git
cd ai-system

# Navigate to devops directory
cd devops
```

### Environment Setup

```bash
# Create virtual environment for Python
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install Python development tools
pip install --upgrade pip
pip install pytest pytest-cov black flake8 mypy

# Install Node.js development tools
npm install -g nodemon eslint prettier
```

---

## Local Development Setup

### 1. Setup Infrastructure Services

Use Docker Compose to run supporting services locally:

```bash
# Create docker-compose.yml for local development
cat > docker-compose-dev.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ai_user
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: ai_system
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ai_user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass dev_password
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: dev_password
    ports:
      - "5672:5672"
      - "15672:15672"
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  qdrant:
    image: qdrant/qdrant:v1.7.0
    ports:
      - "6333:6333"
      - "6334:6334"
    volumes:
      - qdrant_data:/qdrant/storage
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:6333/"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  qdrant_data:
EOF

# Start infrastructure services
docker-compose -f docker-compose-dev.yml up -d

# Verify services are running
docker-compose -f docker-compose-dev.yml ps

# View logs
docker-compose -f docker-compose-dev.yml logs -f
```

### 2. Initialize Database

```bash
# Connect to PostgreSQL
psql -h localhost -U ai_user -d ai_system

# Run migrations (if you have migration files)
# Or run the schema from DEPLOYMENT.md
```

Create `database/schema.sql`:
```sql
-- Users table
CREATE TABLE IF NOT EXISTS users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  role VARCHAR(20) NOT NULL DEFAULT 'viewer',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Approvals table
CREATE TABLE IF NOT EXISTS approvals (
  id SERIAL PRIMARY KEY,
  request_type VARCHAR(50) NOT NULL,
  request_data JSONB NOT NULL,
  status VARCHAR(20) NOT NULL DEFAULT 'pending',
  requested_by INTEGER REFERENCES users(id),
  approved_by INTEGER REFERENCES users(id),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  approved_at TIMESTAMP
);

-- Audit logs table
CREATE TABLE IF NOT EXISTS audit_logs (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  action VARCHAR(100) NOT NULL,
  resource_type VARCHAR(50),
  resource_id INTEGER,
  details JSONB,
  ip_address VARCHAR(45),
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes
CREATE INDEX IF NOT EXISTS idx_approvals_status ON approvals(status);
CREATE INDEX IF NOT EXISTS idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX IF NOT EXISTS idx_audit_logs_created_at ON audit_logs(created_at);

-- Insert test user (password: admin123)
INSERT INTO users (username, email, password_hash, role)
VALUES ('admin', 'admin@example.com', '$2b$10$92IXUNpkjO0rOQ5byMi.Ye4oKoEa3Ro9llC/.og/at2.uheWG/igi', 'admin')
ON CONFLICT (username) DO NOTHING;
```

```bash
# Apply schema
psql -h localhost -U ai_user -d ai_system -f database/schema.sql
```

### 3. Configure Environment Variables

Create `.env` file in each service directory:

**services/llm-service/.env**:
```bash
# Server
PORT=8000
ENV=development
DEBUG=true

# Database
DATABASE_URL=postgresql://ai_user:dev_password@localhost:5432/ai_system

# Redis
REDIS_URL=redis://:dev_password@localhost:6379/0

# RabbitMQ
RABBITMQ_URL=amqp://admin:dev_password@localhost:5672/

# LLM APIs
OPENAI_API_KEY=sk-your-openai-key
ANTHROPIC_API_KEY=sk-ant-your-anthropic-key

# Qdrant
QDRANT_URL=http://localhost:6333
QDRANT_API_KEY=

# Logging
LOG_LEVEL=DEBUG
```

**services/approval-dashboard/backend/.env**:
```bash
# Server
PORT=3000
NODE_ENV=development

# Database
DATABASE_URL=postgresql://ai_user:dev_password@localhost:5432/ai_system

# Redis
REDIS_URL=redis://:dev_password@localhost:6379/0

# JWT
JWT_SECRET=dev-jwt-secret-change-in-production
JWT_REFRESH_SECRET=dev-refresh-secret-change-in-production
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

# SMTP (optional for dev)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password

# CORS
CORS_ORIGIN=http://localhost:3001
```

**services/approval-dashboard/frontend/.env**:
```bash
REACT_APP_API_URL=http://localhost:3000
REACT_APP_WS_URL=ws://localhost:3000
REACT_APP_ENV=development
```

---

## Running Services Locally

### 1. LLM Service (Python/FastAPI)

```bash
cd services/llm-service

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Install dev dependencies
pip install -r requirements-dev.txt

# Run with auto-reload
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Or use the run script
python -m app.main
```

**Alternative with Docker**:
```bash
# Build image
docker build -t llm-service:dev .

# Run container
docker run -p 8000:8000 \
  --env-file .env \
  --network host \
  llm-service:dev
```

**Verify LLM Service**:
```bash
curl http://localhost:8000/health
curl http://localhost:8000/docs  # Open Swagger UI in browser
```

### 2. Worker Service (Python)

```bash
cd services/worker-service

# Create virtual environment
python -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Run worker
python app/main.py

# Or run with logging
python app/main.py --log-level DEBUG
```

**Verify Worker Service**:
```bash
# Check RabbitMQ queues
curl -u admin:dev_password http://localhost:15672/api/queues

# Publish test message
python scripts/test_message.py
```

### 3. Approval Backend (Node.js/Express)

```bash
cd services/approval-dashboard/backend

# Install dependencies
npm install

# Run in development mode (with nodemon)
npm run dev

# Or run normally
npm start
```

**Verify Approval Backend**:
```bash
curl http://localhost:3000/health

# Test login
curl -X POST http://localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

### 4. Approval Frontend (React)

```bash
cd services/approval-dashboard/frontend

# Install dependencies
npm install

# Run development server
npm start

# Build for production
npm run build
```

**Access Frontend**:
Open browser: http://localhost:3001

### 5. Run All Services with Docker Compose

```bash
# Create complete docker-compose.yml
cat > docker-compose-full.yml << 'EOF'
version: '3.8'

services:
  # Infrastructure
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_USER: ai_user
      POSTGRES_PASSWORD: dev_password
      POSTGRES_DB: ai_system
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/schema.sql:/docker-entrypoint-initdb.d/schema.sql

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass dev_password
    ports:
      - "6379:6379"

  rabbitmq:
    image: rabbitmq:3.12-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: dev_password
    ports:
      - "5672:5672"
      - "15672:15672"

  qdrant:
    image: qdrant/qdrant:v1.7.0
    ports:
      - "6333:6333"

  # Application Services
  llm-service:
    build: ./services/llm-service
    ports:
      - "8000:8000"
    env_file:
      - ./services/llm-service/.env
    depends_on:
      - postgres
      - redis
      - rabbitmq
      - qdrant
    volumes:
      - ./services/llm-service:/app
    command: uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

  worker-service:
    build: ./services/worker-service
    env_file:
      - ./services/worker-service/.env
    depends_on:
      - postgres
      - redis
      - rabbitmq
      - llm-service
    volumes:
      - ./services/worker-service:/app

  approval-backend:
    build: ./services/approval-dashboard/backend
    ports:
      - "3000:3000"
    env_file:
      - ./services/approval-dashboard/backend/.env
    depends_on:
      - postgres
      - redis
    volumes:
      - ./services/approval-dashboard/backend:/app
      - /app/node_modules
    command: npm run dev

  approval-frontend:
    build: ./services/approval-dashboard/frontend
    ports:
      - "3001:3000"
    env_file:
      - ./services/approval-dashboard/frontend/.env
    depends_on:
      - approval-backend
    volumes:
      - ./services/approval-dashboard/frontend:/app
      - /app/node_modules
    command: npm start

volumes:
  postgres_data:
EOF

# Start all services
docker-compose -f docker-compose-full.yml up -d

# View logs
docker-compose -f docker-compose-full.yml logs -f

# Stop services
docker-compose -f docker-compose-full.yml down
```

---

## Testing Guidelines

### Unit Testing

#### Python (pytest)

**Test Structure**:
```
services/llm-service/
├── app/
│   ├── main.py
│   ├── llm_client.py
│   └── rag_pipeline.py
└── tests/
    ├── conftest.py
    ├── test_llm_client.py
    └── test_rag_pipeline.py
```

**Example Test** (`tests/test_llm_client.py`):
```python
import pytest
from unittest.mock import Mock, patch
from app.llm_client import LLMClient, LLMConfig, LLMProvider

@pytest.fixture
def llm_config():
    return LLMConfig(
        openai_api_key="test-key",
        preferred_provider=LLMProvider.OPENAI
    )

@pytest.fixture
def llm_client(llm_config):
    return LLMClient(llm_config)

def test_chat_completion_success(llm_client):
    messages = [{"role": "user", "content": "Hello"}]

    with patch('openai.ChatCompletion.create') as mock_create:
        mock_create.return_value = Mock(
            choices=[Mock(message=Mock(content="Hi there!"))]
        )

        response = llm_client.chat_completion(messages)

        assert response == "Hi there!"
        mock_create.assert_called_once()

def test_chat_completion_with_invalid_provider(llm_client):
    messages = [{"role": "user", "content": "Hello"}]

    with pytest.raises(ValueError):
        llm_client.chat_completion(messages, provider="invalid")

def test_failover_to_secondary_provider(llm_client):
    messages = [{"role": "user", "content": "Hello"}]

    with patch('openai.ChatCompletion.create') as mock_openai:
        mock_openai.side_effect = Exception("OpenAI error")

        with patch('anthropic.Anthropic.messages.create') as mock_anthropic:
            mock_anthropic.return_value = Mock(content="Response from Claude")

            response = llm_client.chat_completion(messages)

            assert response == "Response from Claude"
```

**Run Tests**:
```bash
cd services/llm-service

# Run all tests
pytest

# Run with coverage
pytest --cov=app --cov-report=html

# Run specific test file
pytest tests/test_llm_client.py

# Run with verbose output
pytest -v

# Run tests matching pattern
pytest -k "test_chat"
```

#### JavaScript (Jest)

**Test Structure**:
```
services/approval-dashboard/backend/
├── src/
│   ├── controllers/
│   ├── services/
│   └── models/
└── tests/
    ├── unit/
    │   ├── controllers/
    │   └── services/
    └── integration/
```

**Example Test** (`tests/unit/services/auth.test.js`):
```javascript
const { AuthService } = require('../../../src/services/auth');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');

jest.mock('bcrypt');
jest.mock('jsonwebtoken');

describe('AuthService', () => {
  let authService;

  beforeEach(() => {
    authService = new AuthService();
  });

  describe('login', () => {
    it('should return tokens for valid credentials', async () => {
      const user = {
        id: 1,
        username: 'testuser',
        password_hash: 'hashed_password',
        role: 'approver'
      };

      bcrypt.compare.mockResolvedValue(true);
      jwt.sign.mockReturnValue('mock-token');

      const result = await authService.login('testuser', 'password123', user);

      expect(result).toHaveProperty('accessToken');
      expect(result).toHaveProperty('refreshToken');
      expect(result.user.username).toBe('testuser');
    });

    it('should throw error for invalid password', async () => {
      const user = {
        id: 1,
        username: 'testuser',
        password_hash: 'hashed_password'
      };

      bcrypt.compare.mockResolvedValue(false);

      await expect(
        authService.login('testuser', 'wrong_password', user)
      ).rejects.toThrow('Invalid credentials');
    });
  });
});
```

**Run Tests**:
```bash
cd services/approval-dashboard/backend

# Run all tests
npm test

# Run with coverage
npm test -- --coverage

# Run specific test
npm test -- auth.test.js

# Run in watch mode
npm test -- --watch
```

### Integration Testing

**API Integration Test** (`tests/integration/api.test.js`):
```javascript
const request = require('supertest');
const app = require('../../src/app');

describe('API Integration Tests', () => {
  let authToken;

  beforeAll(async () => {
    // Login to get token
    const response = await request(app)
      .post('/api/auth/login')
      .send({
        username: 'admin',
        password: 'admin123'
      });

    authToken = response.body.accessToken;
  });

  describe('GET /api/approvals', () => {
    it('should return list of approvals', async () => {
      const response = await request(app)
        .get('/api/approvals')
        .set('Authorization', `Bearer ${authToken}`);

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('approvals');
      expect(Array.isArray(response.body.approvals)).toBe(true);
    });

    it('should return 401 without token', async () => {
      const response = await request(app)
        .get('/api/approvals');

      expect(response.status).toBe(401);
    });
  });

  describe('POST /api/approvals', () => {
    it('should create new approval request', async () => {
      const response = await request(app)
        .post('/api/approvals')
        .set('Authorization', `Bearer ${authToken}`)
        .send({
          requestType: 'deployment_approval',
          requestData: {
            service: 'test-service',
            version: 'v1.0.0'
          }
        });

      expect(response.status).toBe(201);
      expect(response.body).toHaveProperty('id');
      expect(response.body.status).toBe('pending');
    });
  });
});
```

### End-to-End Testing

Use tools like Cypress or Playwright for frontend E2E tests:

**cypress/e2e/approval-flow.cy.js**:
```javascript
describe('Approval Flow', () => {
  beforeEach(() => {
    cy.visit('http://localhost:3001');
    cy.login('admin', 'admin123');
  });

  it('should display pending approvals', () => {
    cy.get('[data-testid="approvals-list"]').should('be.visible');
    cy.get('[data-testid="approval-item"]').should('have.length.at.least', 1);
  });

  it('should approve a request', () => {
    cy.get('[data-testid="approval-item"]').first().click();
    cy.get('[data-testid="approve-button"]').click();
    cy.get('[data-testid="comment-input"]').type('Approved after review');
    cy.get('[data-testid="confirm-approve"]').click();

    cy.get('[data-testid="success-message"]').should('contain', 'Approved successfully');
  });
});
```

### Test Coverage Goals

- **Unit Tests**: 80%+ coverage
- **Integration Tests**: Critical paths covered
- **E2E Tests**: User workflows covered

```bash
# Generate coverage report
pytest --cov=app --cov-report=html  # Python
npm test -- --coverage              # JavaScript

# View coverage
open htmlcov/index.html             # Python
open coverage/lcov-report/index.html # JavaScript
```

---

## Code Standards

### Python Code Style

Follow **PEP 8** and use automated tools:

#### Black (Code Formatter)
```bash
# Format code
black app/ tests/

# Check without modifying
black --check app/ tests/
```

#### Flake8 (Linter)
```bash
# Check code style
flake8 app/ tests/

# Configuration in .flake8
[flake8]
max-line-length = 100
exclude = .git,__pycache__,venv
ignore = E203, W503
```

#### MyPy (Type Checker)
```bash
# Run type checking
mypy app/

# Configuration in mypy.ini
[mypy]
python_version = 3.11
warn_return_any = True
warn_unused_configs = True
disallow_untyped_defs = True
```

#### Example Python Code:
```python
"""
Module for LLM client operations.

This module provides a unified interface for multiple LLM providers.
"""
from typing import List, Dict, Optional
from enum import Enum


class LLMProvider(str, Enum):
    """Enumeration of supported LLM providers."""

    OPENAI = "openai"
    ANTHROPIC = "anthropic"


class LLMClient:
    """
    Client for interacting with LLM APIs.

    Attributes:
        config: Configuration for LLM client
        clients: Dictionary of initialized provider clients
    """

    def __init__(self, config: LLMConfig) -> None:
        """
        Initialize LLM client.

        Args:
            config: LLM configuration object
        """
        self.config = config
        self.clients: Dict[LLMProvider, Any] = {}
        self._initialize_clients()

    def chat_completion(
        self,
        messages: List[Dict[str, str]],
        provider: Optional[LLMProvider] = None,
        temperature: float = 0.7,
    ) -> str:
        """
        Get chat completion from LLM.

        Args:
            messages: List of message dictionaries
            provider: LLM provider to use (uses default if None)
            temperature: Sampling temperature (0.0 to 2.0)

        Returns:
            Generated response text

        Raises:
            ValueError: If provider is invalid
            LLMError: If API call fails
        """
        provider = provider or self.config.preferred_provider

        if provider not in self.clients:
            raise ValueError(f"Provider {provider} not configured")

        # Implementation...
        return response
```

### JavaScript/TypeScript Code Style

Follow **Airbnb Style Guide**:

#### ESLint Configuration
```json
// .eslintrc.json
{
  "extends": ["airbnb-base", "prettier"],
  "env": {
    "node": true,
    "jest": true
  },
  "rules": {
    "no-console": "warn",
    "prefer-const": "error",
    "no-var": "error"
  }
}
```

#### Prettier Configuration
```json
// .prettierrc
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2
}
```

#### Run Linters:
```bash
# ESLint
eslint src/ --fix

# Prettier
prettier --write "src/**/*.js"

# Run both
npm run lint
npm run format
```

#### Example JavaScript Code:
```javascript
/**
 * Authentication service
 * Handles user authentication and JWT token management
 */
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { ValidationError, AuthenticationError } = require('../errors');

class AuthService {
  /**
   * Create authentication service
   * @param {Object} config - Service configuration
   */
  constructor(config) {
    this.config = config;
    this.jwtSecret = process.env.JWT_SECRET;
  }

  /**
   * Authenticate user and generate tokens
   * @param {string} username - Username
   * @param {string} password - Password
   * @param {Object} user - User object from database
   * @returns {Promise<Object>} Tokens and user info
   * @throws {AuthenticationError} If credentials are invalid
   */
  async login(username, password, user) {
    if (!user) {
      throw new AuthenticationError('Invalid credentials');
    }

    const isValid = await bcrypt.compare(password, user.password_hash);
    if (!isValid) {
      throw new AuthenticationError('Invalid credentials');
    }

    const accessToken = this.generateAccessToken(user);
    const refreshToken = this.generateRefreshToken(user);

    return {
      accessToken,
      refreshToken,
      user: {
        id: user.id,
        username: user.username,
        email: user.email,
        role: user.role,
      },
    };
  }

  /**
   * Generate JWT access token
   * @private
   * @param {Object} user - User object
   * @returns {string} JWT token
   */
  generateAccessToken(user) {
    return jwt.sign(
      {
        userId: user.id,
        username: user.username,
        role: user.role,
      },
      this.jwtSecret,
      { expiresIn: '15m' }
    );
  }
}

module.exports = AuthService;
```

### Commit Message Convention

Follow **Conventional Commits**:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types**:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting)
- `refactor`: Code refactoring
- `test`: Adding tests
- `chore`: Build process or auxiliary tool changes

**Examples**:
```bash
feat(llm-service): add streaming chat completion endpoint

Implements Server-Sent Events (SSE) for real-time streaming
of chat completions from OpenAI and Anthropic APIs.

Closes #123

---

fix(approval-backend): handle expired JWT tokens correctly

Previously, expired tokens would cause server error 500.
Now returns proper 401 with refresh token guidance.

Fixes #456

---

docs(api): update authentication flow documentation

Added examples for token refresh and WebSocket authentication.
```

---

## Git Workflow

### Branching Strategy

We use **Git Flow**:

```
main (production)
  └── develop (integration)
      ├── feature/user-authentication
      ├── feature/rag-pipeline
      ├── bugfix/fix-jwt-expiry
      └── hotfix/critical-security-patch
```

### Branch Naming

- `feature/<ticket-id>-<description>` - New features
- `bugfix/<ticket-id>-<description>` - Bug fixes
- `hotfix/<ticket-id>-<description>` - Critical fixes
- `refactor/<description>` - Code refactoring
- `docs/<description>` - Documentation updates

### Workflow

```bash
# 1. Create feature branch from develop
git checkout develop
git pull origin develop
git checkout -b feature/123-add-streaming-chat

# 2. Make changes and commit
git add .
git commit -m "feat(llm-service): add streaming chat endpoint"

# 3. Push branch
git push origin feature/123-add-streaming-chat

# 4. Create Pull Request on GitHub/GitLab

# 5. After review and approval, merge to develop

# 6. Delete feature branch
git branch -d feature/123-add-streaming-chat
git push origin --delete feature/123-add-streaming-chat
```

### Pre-commit Hooks

Setup pre-commit hooks to ensure code quality:

```bash
# Install pre-commit
pip install pre-commit

# Create .pre-commit-config.yaml
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.4.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-added-large-files

  - repo: https://github.com/psf/black
    rev: 23.3.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8

  - repo: https://github.com/pre-commit/mirrors-eslint
    rev: v8.44.0
    hooks:
      - id: eslint
        files: \.(js|jsx|ts|tsx)$
EOF

# Install hooks
pre-commit install

# Run manually
pre-commit run --all-files
```

---

## Contributing Guidelines

### Pull Request Process

1. **Create Issue**: Describe the feature/bug
2. **Create Branch**: From `develop` branch
3. **Implement Changes**: Follow code standards
4. **Write Tests**: Maintain >80% coverage
5. **Update Documentation**: If needed
6. **Create PR**: With clear description
7. **Code Review**: Address feedback
8. **Merge**: After approval

### Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Related Issue
Closes #123

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing performed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Comments added for complex code
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] Tests pass locally
```

### Code Review Guidelines

**For Authors**:
- Keep PRs small and focused
- Provide context and screenshots
- Respond to feedback promptly
- Test thoroughly before requesting review

**For Reviewers**:
- Be constructive and respectful
- Focus on logic and design, not style
- Approve only when satisfied
- Provide actionable feedback

---

## Debugging Tips

### Python Debugging

#### Using pdb
```python
import pdb

def problematic_function(data):
    pdb.set_trace()  # Debugger will pause here
    result = process_data(data)
    return result
```

#### Using VS Code Debugger
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Python: FastAPI",
      "type": "python",
      "request": "launch",
      "module": "uvicorn",
      "args": [
        "app.main:app",
        "--reload"
      ],
      "jinja": true,
      "justMyCode": false
    }
  ]
}
```

#### Logging
```python
import logging

logger = logging.getLogger(__name__)
logger.setLevel(logging.DEBUG)

# Add detailed logging
logger.debug(f"Processing request with data: {data}")
logger.info("Request completed successfully")
logger.error(f"Error occurred: {str(e)}", exc_info=True)
```

### JavaScript Debugging

#### Using Chrome DevTools
```bash
node --inspect-brk server.js
# Open chrome://inspect in Chrome
```

#### Using VS Code Debugger
```json
// .vscode/launch.json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Express",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/src/server.js",
      "restart": true,
      "runtimeExecutable": "nodemon",
      "console": "integratedTerminal"
    }
  ]
}
```

### Docker Debugging

```bash
# Check container logs
docker logs <container-name> -f

# Execute commands in container
docker exec -it <container-name> /bin/bash

# Inspect container
docker inspect <container-name>

# Check resource usage
docker stats
```

### Database Debugging

```bash
# Connect to PostgreSQL
docker exec -it postgres psql -U ai_user -d ai_system

# Check active connections
SELECT * FROM pg_stat_activity;

# Explain query performance
EXPLAIN ANALYZE SELECT * FROM approvals WHERE status = 'pending';

# Check table sizes
SELECT
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

---

## IDE Setup

### Visual Studio Code

**Recommended Extensions**:
- Python (Microsoft)
- Pylance
- ESLint
- Prettier
- Docker
- GitLens
- Thunder Client (API testing)
- Better Comments

**Settings** (`.vscode/settings.json`):
```json
{
  "python.linting.enabled": true,
  "python.linting.pylintEnabled": false,
  "python.linting.flake8Enabled": true,
  "python.formatting.provider": "black",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter"
  },
  "[javascript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  }
}
```

### PyCharm

**Configuration**:
1. Set Python interpreter to virtual environment
2. Enable Pylint and Black
3. Configure run/debug configurations
4. Setup database connection

### IntelliJ IDEA

**Configuration**:
1. Install Python plugin
2. Setup Node.js interpreter
3. Configure ESLint and Prettier
4. Setup run configurations for Express

---

## Conclusion

This development guide provides everything needed to start contributing to the AI-Driven Hybrid Kubernetes System. Follow the standards, write tests, and maintain clean code.

For questions or help:
- Check existing documentation
- Ask in team chat
- Create an issue on GitHub
- Join development meetings

Happy coding!
