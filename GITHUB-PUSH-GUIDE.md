# Quick Guide: Pushing to GitHub for Codespaces

## 🎯 Quick Start Steps

### 1. Initialize Git Repository (if not already done)

```bash
cd devops
git init
git branch -M main
```

### 2. Add Your GitHub Remote

Replace `YOUR_USERNAME` and `YOUR_REPO` with your actual values:

```bash
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git

# Verify remote
git remote -v
```

### 3. Stage All Files

```bash
# Stage all files
git add .

# Verify what will be committed
git status
```

### 4. Create Your First Commit

```bash
git commit -m "Initial commit: AI-Driven Hybrid Kubernetes System

- Complete LLM service with OpenAI, Anthropic, Gemini support
- RAG pipeline with Qdrant vector database
- Worker service for async task processing
- Approval dashboard (backend + frontend)
- Docker Compose setup for local development
- GitHub Codespaces configuration (.devcontainer)
- Kubernetes manifests and Helm charts
- Infrastructure as Code (Terraform)
- Comprehensive documentation

🤖 Generated with Claude Code
"
```

### 5. Push to GitHub

```bash
# Push to main branch
git push -u origin main
```

---

## 🔐 Before You Push: Security Checklist

### ✅ Verify .env is NOT Being Committed

```bash
# Check if .env is in .gitignore
cat .gitignore | grep -E '\.env'

# Verify .env won't be committed
git status | grep .env
```

**Expected**: `.env` should NOT appear in the list of files to be committed.

### ✅ Check for Sensitive Data

```bash
# Search for potential API keys in staged files
git grep -i "sk-proj-" $(git diff --cached --name-only)
git grep -i "sk-ant-" $(git diff --cached --name-only)
git grep -i "api.key" $(git diff --cached --name-only)
```

**Expected**: No matches found.

### ✅ Remove Sensitive Files if Accidentally Staged

```bash
# If .env was accidentally added:
git rm --cached .env

# If credentials.json was added:
git rm --cached credentials.json

# Then commit again
git commit --amend -m "Your commit message"
```

---

## 🚀 Setting Up GitHub Codespaces

### After Pushing to GitHub:

1. **Go to Your GitHub Repository**
   ```
   https://github.com/YOUR_USERNAME/YOUR_REPO
   ```

2. **Create a Codespace**
   - Click the **"Code"** button (green button)
   - Select **"Codespaces"** tab
   - Click **"Create codespace on main"**

3. **Wait for Build** (~5-10 minutes first time)
   - The `.devcontainer/devcontainer.json` will be used automatically
   - All dependencies will be installed
   - Services will start automatically

4. **Configure Secrets**

   Option A: Using GitHub UI
   - Go to: Settings → Codespaces → Codespaces secrets
   - Click "New secret"
   - Add:
     - Name: `OPENAI_API_KEY`
     - Value: Your OpenAI API key
   - Add:
     - Name: `ANTHROPIC_API_KEY`
     - Value: Your Anthropic API key

   Option B: In Codespace Terminal
   ```bash
   # Edit .env file
   nano /workspace/devops/.env

   # Update API keys
   OPENAI_API_KEY=sk-proj-YOUR_KEY_HERE
   ANTHROPIC_API_KEY=sk-ant-YOUR_KEY_HERE

   # Save and exit (Ctrl+O, Enter, Ctrl+X)
   ```

5. **Start Services**
   ```bash
   cd /workspace/devops
   docker-compose up -d
   ```

6. **Test Everything**
   ```bash
   cd /workspace/devops/services/llm-service
   python test_complete.py
   ```

---

## 📝 Recommended .gitignore Items

Your project already has a comprehensive `.gitignore`, but verify these critical items are included:

```gitignore
# Environment files
.env
.env.*
!.env.example

# Secrets
secrets/
*.key
*.pem
credentials.json

# API Keys
**/api-keys.txt
**/openai-key.txt
**/anthropic-key.txt

# Cloud provider credentials
.aws/
.gcp/
.azure/

# Local development
docker-compose.override.yml
local.yml
```

---

## 🔄 Updating Your Repository

### Workflow for Future Changes:

```bash
# 1. Make your changes
# Edit files...

# 2. Check what changed
git status
git diff

# 3. Stage changes
git add .
# Or stage specific files
git add path/to/file.py

# 4. Commit
git commit -m "Description of changes"

# 5. Push
git push
```

### Creating Feature Branches

```bash
# Create and switch to new branch
git checkout -b feature/new-feature

# Make changes and commit
git add .
git commit -m "Add new feature"

# Push branch to GitHub
git push -u origin feature/new-feature

# Create Pull Request on GitHub
# Then merge when ready
```

---

## 📚 Common Git Commands

```bash
# Check repository status
git status

# View commit history
git log --oneline

# View differences
git diff
git diff --staged

# Undo changes (before commit)
git restore path/to/file.py
git restore --staged path/to/file.py  # Unstage

# Undo last commit (keep changes)
git reset --soft HEAD~1

# View remote repositories
git remote -v

# Fetch latest from remote
git fetch origin

# Pull latest changes
git pull origin main

# Create new branch
git checkout -b branch-name

# Switch branches
git checkout main

# Delete branch
git branch -d branch-name

# View all branches
git branch -a
```

---

## 🐛 Troubleshooting

### Problem: "Permission denied (publickey)"

**Solution**: Set up SSH keys or use HTTPS with personal access token

```bash
# Use HTTPS instead
git remote set-url origin https://github.com/YOUR_USERNAME/YOUR_REPO.git

# Or set up SSH
ssh-keygen -t ed25519 -C "your_email@example.com"
cat ~/.ssh/id_ed25519.pub
# Add to GitHub: Settings → SSH and GPG keys → New SSH key
```

### Problem: "Failed to push - rejected"

**Solution**: Pull first, then push

```bash
git pull origin main --rebase
git push origin main
```

### Problem: Accidentally committed .env file

**Solution**: Remove from history

```bash
# Remove from last commit
git rm --cached .env
git commit --amend -m "Remove .env file"
git push --force

# If already pushed multiple commits ago:
# Use git filter-branch or BFG Repo-Cleaner
# (More complex - consult GitHub docs)
```

### Problem: Large files (>100MB)

**Solution**: Use Git LFS or .gitignore

```bash
# Install Git LFS
git lfs install

# Track large files
git lfs track "*.psd"
git lfs track "*.model"

# Commit .gitattributes
git add .gitattributes
git commit -m "Configure Git LFS"

# Then commit large files normally
git add large-file.model
git commit -m "Add model file"
git push
```

---

## 📖 Additional Resources

### GitHub Documentation
- **Codespaces**: https://docs.github.com/en/codespaces
- **Devcontainer Reference**: https://containers.dev/
- **Git Basics**: https://git-scm.com/book/en/v2

### Project Documentation
- **Codespaces Setup**: `CODESPACES-SETUP.md` (detailed guide)
- **Docker Compose**: `DOCKER-COMPOSE-GUIDE.md`
- **Test Results**: `TEST-RESULTS.md`
- **Architecture**: `README.md`

---

## ✅ Pre-Push Checklist

Before pushing to GitHub:

- [ ] `.env` file is in `.gitignore`
- [ ] No API keys in committed code
- [ ] No credentials or secrets
- [ ] Code is tested locally
- [ ] Docker Compose works (`docker-compose up -d`)
- [ ] LLM tests pass (`python test_complete.py`)
- [ ] Commit message is clear and descriptive
- [ ] All necessary files are staged
- [ ] `.devcontainer/` directory is included

---

## 🎉 Ready to Push!

Once you've completed these steps, your repository will be ready for GitHub Codespaces!

**Quick Command Summary:**

```bash
# Navigate to devops directory
cd devops

# Initialize (if needed)
git init
git branch -M main

# Add remote
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPO.git

# Stage, commit, push
git add .
git commit -m "Initial commit: AI-Driven Hybrid Kubernetes System"
git push -u origin main

# Go to GitHub and create a Codespace!
```

**Good luck! 🚀**

---

**Last Updated**: 2025-10-18
