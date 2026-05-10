# Contributing to DevOps Platform

Thank you for your interest in contributing to the DevOps Platform! This document provides guidelines and instructions for contributing to the project.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Coding Standards](#coding-standards)
- [Testing Guidelines](#testing-guidelines)
- [Commit Messages](#commit-messages)
- [Pull Request Process](#pull-request-process)
- [Documentation](#documentation)
- [Issue Reporting](#issue-reporting)
- [Community](#community)

## Code of Conduct

### Our Pledge

We are committed to providing a welcoming and inclusive environment for all contributors. We expect all participants to:

- Be respectful and considerate
- Accept constructive criticism gracefully
- Focus on what is best for the community
- Show empathy towards other community members

### Unacceptable Behavior

- Harassment, discrimination, or offensive comments
- Personal attacks or trolling
- Publishing others' private information
- Any conduct that would be considered inappropriate in a professional setting

## Getting Started

### Prerequisites

Before you begin, ensure you have the following installed:

- **Python 3.10+** - For LLM and Worker services
- **Node.js 18+** - For Approval backend and frontend
- **Docker & Docker Compose** - For containerized development
- **Git** - For version control
- **Make** - For build automation (optional but recommended)

### Setting Up Development Environment

1. **Clone the repository**
   ```bash
   git clone https://github.com/your-org/devops-platform.git
   cd devops-platform
   ```

2. **Set up environment variables**
   ```bash
   cp .env.example .env
   # Edit .env with your configuration
   ```

3. **Install dependencies**
   ```bash
   make install
   # Or manually:
   # make install-python
   # make install-node
   ```

4. **Start development services**
   ```bash
   make start
   # Or: docker-compose up -d
   ```

5. **Verify installation**
   ```bash
   make health
   ```

### Project Structure

```
devops/
├── services/           # Microservices
│   ├── llm-service/   # AI/LLM service (Python/FastAPI)
│   ├── worker/        # Background worker (Python)
│   ├── approval-backend/  # API backend (Node.js/Express)
│   └── approval-frontend/ # Web UI (React/Next.js)
├── infrastructure/    # Infrastructure configuration
├── k8s/              # Kubernetes manifests
├── ci-cd/            # CI/CD pipelines
├── monitoring/       # Monitoring configuration
├── docs/             # Documentation
└── tests/            # Integration and E2E tests
```

## Development Workflow

### Branch Naming Convention

Use descriptive branch names following this pattern:

- `feature/description` - New features
- `bugfix/description` - Bug fixes
- `hotfix/description` - Urgent production fixes
- `refactor/description` - Code refactoring
- `docs/description` - Documentation updates
- `test/description` - Test additions or modifications

Examples:
- `feature/add-user-authentication`
- `bugfix/fix-memory-leak`
- `docs/update-api-documentation`

### Development Process

1. **Create a new branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Make your changes**
   - Write clean, readable code
   - Follow coding standards (see below)
   - Add tests for new functionality
   - Update documentation as needed

3. **Test your changes**
   ```bash
   make test
   make lint
   ```

4. **Commit your changes**
   ```bash
   git add .
   git commit -m "feat: add user authentication"
   ```

5. **Push to your fork**
   ```bash
   git push origin feature/your-feature-name
   ```

6. **Create a Pull Request**
   - Provide a clear description
   - Reference related issues
   - Request reviews from maintainers

## Coding Standards

### Python (LLM Service, Worker)

- Follow **PEP 8** style guide
- Use **Black** for code formatting
- Use **type hints** for all functions
- Maximum line length: **88 characters**
- Use **docstrings** for all modules, classes, and functions

```python
from typing import List, Optional

def process_data(items: List[str], max_items: Optional[int] = None) -> dict:
    """
    Process a list of items and return statistics.

    Args:
        items: List of items to process
        max_items: Maximum number of items to process (optional)

    Returns:
        Dictionary containing processing statistics
    """
    # Implementation here
    pass
```

**Tools:**
```bash
# Format code
black src/

# Lint code
flake8 src/
pylint src/

# Type checking
mypy src/
```

### Node.js/TypeScript (Approval Backend, Frontend)

- Follow **Airbnb JavaScript Style Guide**
- Use **Prettier** for code formatting
- Use **TypeScript** for type safety
- Use **ESLint** for linting
- Maximum line length: **100 characters**

```typescript
interface UserData {
  id: string;
  name: string;
  email: string;
}

/**
 * Fetch user data from the API
 * @param userId - The ID of the user to fetch
 * @returns Promise resolving to user data
 */
async function fetchUser(userId: string): Promise<UserData> {
  // Implementation here
}
```

**Tools:**
```bash
# Format code
npm run format

# Lint code
npm run lint

# Type checking
npm run type-check
```

### General Best Practices

- **DRY (Don't Repeat Yourself)** - Avoid code duplication
- **SOLID Principles** - Follow object-oriented design principles
- **KISS (Keep It Simple, Stupid)** - Prefer simple solutions
- **YAGNI (You Aren't Gonna Need It)** - Don't add unnecessary features
- **Write self-documenting code** - Use clear variable and function names
- **Handle errors gracefully** - Always include proper error handling
- **Log appropriately** - Use appropriate log levels

## Testing Guidelines

### Test Structure

Each service should have comprehensive tests:

- **Unit Tests** - Test individual functions and methods
- **Integration Tests** - Test service interactions
- **E2E Tests** - Test complete user workflows

### Writing Tests

#### Python Tests (pytest)

```python
import pytest
from src.services.llm_service import generate_response

def test_generate_response_success():
    """Test successful response generation"""
    result = generate_response("Hello, world!")
    assert result is not None
    assert isinstance(result, str)
    assert len(result) > 0

def test_generate_response_empty_input():
    """Test response generation with empty input"""
    with pytest.raises(ValueError):
        generate_response("")
```

#### Node.js Tests (Jest)

```typescript
import { createUser } from './userService';

describe('User Service', () => {
  it('should create a new user', async () => {
    const userData = { name: 'John Doe', email: 'john@example.com' };
    const user = await createUser(userData);

    expect(user).toBeDefined();
    expect(user.id).toBeTruthy();
    expect(user.name).toBe(userData.name);
  });

  it('should throw error for invalid email', async () => {
    const userData = { name: 'John Doe', email: 'invalid-email' };

    await expect(createUser(userData)).rejects.toThrow();
  });
});
```

### Running Tests

```bash
# Run all tests
make test

# Run specific service tests
make test-python
make test-node

# Run with coverage
make test-coverage

# Run integration tests
make test-integration

# Run E2E tests
make test-e2e
```

### Test Coverage Requirements

- **Minimum coverage**: 80%
- **Critical paths**: 100% coverage
- Pull requests must not decrease overall coverage

## Commit Messages

We follow the **Conventional Commits** specification:

### Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Types

- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation changes
- **style**: Code style changes (formatting, etc.)
- **refactor**: Code refactoring
- **test**: Adding or updating tests
- **chore**: Maintenance tasks
- **perf**: Performance improvements
- **ci**: CI/CD changes

### Examples

```
feat(llm): add support for GPT-4 Turbo model

Implemented support for the new GPT-4 Turbo model with enhanced
context window and improved performance.

Closes #123
```

```
fix(worker): resolve memory leak in task processing

Fixed a memory leak caused by unclosed database connections in
the task processing loop.

Fixes #456
```

```
docs(api): update API documentation for v2 endpoints

Added comprehensive documentation for new v2 API endpoints with
examples and response schemas.
```

## Pull Request Process

### Before Submitting

1. **Update your branch**
   ```bash
   git checkout main
   git pull origin main
   git checkout your-branch
   git rebase main
   ```

2. **Run tests and linting**
   ```bash
   make test
   make lint
   ```

3. **Update documentation**
   - Update README if needed
   - Update API documentation
   - Add/update inline comments

### PR Description Template

```markdown
## Description
Brief description of the changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Related Issues
Closes #123

## Changes Made
- Added X feature
- Fixed Y bug
- Updated Z documentation

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Screenshots (if applicable)
[Add screenshots here]

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Comments added to complex code
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] Tests added/updated
- [ ] All tests passing
```

### Review Process

1. **Automated checks** must pass:
   - Linting
   - Tests
   - Security scans
   - Build validation

2. **Code review** required:
   - At least 2 approvals from maintainers
   - Address all review comments
   - Resolve all conversations

3. **Final checks**:
   - No merge conflicts
   - Up to date with main branch
   - CI/CD pipeline passes

## Documentation

### Documentation Standards

- Use **Markdown** for documentation
- Include code examples
- Add diagrams where helpful
- Keep documentation up to date with code changes

### Types of Documentation

1. **Code Documentation**
   - Inline comments for complex logic
   - Docstrings for functions and classes
   - README files in each service directory

2. **API Documentation**
   - OpenAPI/Swagger specifications
   - Example requests and responses
   - Error codes and handling

3. **User Documentation**
   - Setup guides
   - Feature documentation
   - Troubleshooting guides

### Generating Documentation

```bash
# Generate API documentation
make docs

# Serve documentation locally
make docs-serve
```

## Issue Reporting

### Before Creating an Issue

1. **Search existing issues** to avoid duplicates
2. **Check documentation** for solutions
3. **Verify the issue** in the latest version

### Issue Template

```markdown
## Description
Clear description of the issue

## Steps to Reproduce
1. Step one
2. Step two
3. Step three

## Expected Behavior
What should happen

## Actual Behavior
What actually happens

## Environment
- OS: [e.g., Ubuntu 22.04]
- Docker version: [e.g., 24.0.0]
- Service version: [e.g., 1.2.3]

## Additional Context
Any other relevant information

## Screenshots
[If applicable]
```

### Issue Labels

- `bug` - Something isn't working
- `enhancement` - New feature or request
- `documentation` - Documentation improvements
- `good first issue` - Good for newcomers
- `help wanted` - Extra attention needed
- `priority:high` - High priority issue
- `priority:medium` - Medium priority issue
- `priority:low` - Low priority issue

## Community

### Getting Help

- **GitHub Discussions** - Ask questions and share ideas
- **Slack Channel** - Real-time chat with the community
- **Stack Overflow** - Tag questions with `devops-platform`
- **Email** - devops-platform@example.com

### Contributing to Discussions

- Be respectful and constructive
- Search before posting
- Provide context and examples
- Follow up on your questions

### Maintainers

Current maintainers:
- @maintainer1 - Project Lead
- @maintainer2 - Backend Lead
- @maintainer3 - Frontend Lead
- @maintainer4 - DevOps Lead

## Recognition

Contributors are recognized in:
- README contributors section
- Release notes
- Annual contributor list

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

Thank you for contributing to DevOps Platform!
