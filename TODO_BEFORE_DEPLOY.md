# ✅ What's Already Configured vs ⚠️ What You Need to Get

I've updated your `deploy-to-azure.sh` script with values from your existing `.env` file. Here's the status:

---

## ✅ **ALREADY CONFIGURED** (No Action Needed)

These values were automatically filled from your .env file:

| Configuration | Value | Status |
|---------------|-------|--------|
| **OpenAI API Key** | `sk-proj-hamj8...` | ✅ Ready |
| **Anthropic API Key** | `sk-mega-4cc0...` | ✅ Ready |
| **Qdrant URL** | `https://881655c1-4563...` | ✅ Ready |
| **Qdrant API Key** | `eyJhbGciOiJ...` | ✅ Ready |
| **Slack Bot Token** | `xoxb-9451841...` | ✅ Ready |
| **Slack Channel** | `#devopsalerts` | ✅ Ready |
| **JWT Secret** | `p8K3mN9xV2qR...` | ✅ Ready |
| **Database Password** | `devops123` | ✅ Ready |
| **RabbitMQ Password** | `devops123` | ✅ Ready |

---

## ⚠️ **ONLY 2 THINGS YOU NEED TO ADD**

### **1. SUBSCRIPTION_ID** (REQUIRED)

**What is it?**
Your Azure subscription ID that has the $100 credits.

**How to get it:**

**Step 1:** Open [CLOUD_PROVIDER] Cloud Shell
- Go to https://portal.azure.com
- Click the `>_` icon (Cloud Shell) at the top

**Step 2:** Run this command:
```bash
az account list --output table
```

**Step 3:** You'll see output like this:
```
Name                    SubscriptionId                        IsDefault
----------------------  ------------------------------------  -----------
Pay-As-You-Go          12345678-90ab-cdef-1234-567890abcdef  True
Visual Studio          87654321-4321-4321-4321-cba987654321  False
```

**Step 4:** Copy the `SubscriptionId` value (the long one with dashes)

**Step 5:** Update the script:
```bash
SUBSCRIPTION_ID="12345678-90ab-cdef-1234-567890abcdef"  # ← Paste your actual ID here
```

---

### **2. SLACK_WEBHOOK_URL** (REQUIRED)

**What is it?**
A special URL that lets the notifier-agent send messages to your Slack channel.

**How to get it:**

**Step 1:** Go to Slack App Management
- Open: https://api.slack.com/apps

**Step 2:** Create a new app (if you don't have one)
- Click **"Create New App"**
- Choose **"From scratch"**
- App name: `DevOps AI Alerts`
- Choose your workspace
- Click **"Create App"**

**Step 3:** Enable Incoming Webhooks
- In the left sidebar, click **"Incoming Webhooks"**
- Toggle **"Activate Incoming Webhooks"** to **ON**

**Step 4:** Add webhook to your workspace
- Scroll down and click **"Add New Webhook to Workspace"**
- Select channel: `#devopsalerts` (the one you already have in .env)
- Click **"Allow"**

**Step 5:** Copy the webhook URL
- You'll see a URL like:
  ```
  https://hooks.slack.com/services/T01234ABC/B01234DEF/abc123xyz456
  ```
- Copy this entire URL

**Step 6:** Update the script:
```bash
SLACK_WEBHOOK_URL="https://hooks.slack.com/services/T01234ABC/B01234DEF/abc123xyz456"  # ← Paste your webhook URL
```

---

## 📝 **Summary: What You Need to Do**

1. **Get [CLOUD_PROVIDER] Subscription ID:**
   - Run: `az account list --output table`
   - Copy the SubscriptionId
   - Update line 21 in `deploy-to-azure.sh`

2. **Get Slack Webhook URL:**
   - Go to https://api.slack.com/apps
   - Create app → Enable Incoming Webhooks → Add to workspace
   - Copy the webhook URL
   - Update line 51 in `deploy-to-azure.sh`

---

## 🚀 **After You Add These 2 Values**

You're ready to deploy! Just run:

```bash
cd devops
chmod +x deploy-to-azure.sh
./deploy-to-azure.sh
```

The script will:
- ✅ Create all [CLOUD_PROVIDER] resources
- ✅ Build Docker images in the cloud (no local Docker needed)
- ✅ Deploy all 8 services
- ✅ Configure everything automatically

**Total deployment time:** ~20-30 minutes

---

## 💡 **Optional: Gemini API Key**

If you want to add Google Gemini as a fallback LLM provider:

1. Go to: https://makersuite.google.com/app/apikey
2. Sign in with Google account
3. Click **"Get API Key"**
4. Copy the key (starts with `AIzaSy...`)
5. Update line 42 in `deploy-to-azure.sh`:
   ```bash
   GEMINI_API_KEY="AIzaSyAbc123xyz..."
   ```

**But this is optional!** You already have OpenAI and Anthropic configured, which is plenty.

---

## ❓ **Questions?**

### Q: My OpenAI key looks different - is it still valid?
**A:** Yes! OpenAI keys can start with `sk-proj-` (project keys) or just `sk-` (legacy keys). Both work fine.

### Q: Do I need to create a Qdrant cluster?
**A:** No! You already have one configured. Your URL shows it's in AWS US-West-1 region.

### Q: Can I use a different Slack channel?
**A:** Yes! Just change `SLACK_CHANNEL="#devopsalerts"` to your preferred channel name. Make sure the channel exists first.

### Q: What if I don't want Slack notifications?
**A:** You can skip the webhook URL for now, but the notifier-agent will fail to start. If you want to disable it entirely, you can comment it out from the deployment script later.

---

## 🎯 **Next Steps**

1. Add SUBSCRIPTION_ID (2 minutes)
2. Add SLACK_WEBHOOK_URL (3-5 minutes)
3. Run `deploy-to-azure.sh` (20-30 minutes)
4. Access your dashboard and test!

---

**You're almost there! Just 2 quick values and you can deploy!** 🚀
