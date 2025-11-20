# Yocto Files Auto-Sync to Vectorstore - Complete Documentation

## Table of Contents
1. [Overview](#overview)
2. [What This Pipeline Does](#what-this-pipeline-does)
3. [How It Works](#how-it-works)
4. [Prerequisites](#prerequisites)
5. [GitHub Repository Setup](#github-repository-setup)
6. [Jenkins Setup](#jenkins-setup)
7. [GitHub Webhook Configuration](#github-webhook-configuration)
8. [Network & Security Setup](#network--security-setup)
9. [Testing the Integration](#testing-the-integration)
10. [Troubleshooting](#troubleshooting)
11. [Maintenance](#maintenance)

***

## Overview

### What is This Pipeline?

This is an **automated CI/CD pipeline** that keeps your AI knowledge base synchronized with your Yocto project files in real-time. When you push changes to Yocto recipe files (`.bb`, `.bbappend`, `.conf`, etc.) in your GitHub repository, this pipeline:

1. **Detects** the changes automatically via webhook
2. **Filters** only Yocto-related files
3. **Processes** them into AI-readable chunks
4. **Updates** the vectorstore (FAISS index)
5. **Refreshes** the RAG API to use updated knowledge

### Key Benefits

- 🔄 **Automatic Sync**: Zero manual effort to update knowledge base
- ⚡ **Real-time**: Changes reflected in AI within minutes
- 🎯 **Smart Filtering**: Only processes relevant Yocto files
- 🛡️ **Safe**: Won't break existing vectorstore data
- 📊 **Traceable**: Full build history and logs

***

## What This Pipeline Does

### Simple Explanation

```
Developer pushes Yocto file changes to GitHub
    ↓
GitHub webhook triggers Jenkins
    ↓
Jenkins detects changed .bb/.conf files
    ↓
Files are embedded into vectors
    ↓
Vectorstore updated with new knowledge
    ↓
RAG API refreshed
    ↓
AI now knows about your latest recipes!
```

### Real-World Scenario

**Before (Manual):**
```
1. Developer updates custom-recipe.bb
2. Commits to GitHub
3. DevOps manually copies file to RAG server
4. Manually runs Python script to update vectorstore
5. Manually restarts RAG API
Total time: 15-30 minutes, error-prone
```

**After (Automated):**
```
1. Developer updates custom-recipe.bb
2. Commits to GitHub
3. Pipeline automatically triggers
4. Vectorstore updated in 2-3 minutes
5. Done!
Total time: 2-3 minutes, zero manual work
```

***

## How It Works

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Repository                         │
│              (Yocto recipes + Jenkinsfile)                   │
└────────────────────────┬────────────────────────────────────┘
                         │ Push/Commit
                         │
                         ▼
                    ┌─────────┐
                    │ Webhook │ (Configured in GitHub)
                    └────┬────┘
                         │ HTTP POST
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    Jenkins Server                            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  GenericTrigger Plugin                               │   │
│  │  (Filters: *.bb, *.bbappend, *.conf, *.inc)         │   │
│  └───────────────────┬──────────────────────────────────┘   │
│                      ▼                                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Pipeline Stages:                                    │   │
│  │  1. Initialize (detect changed files)                │   │
│  │  2. Stage Documents (copy to temp dir)               │   │
│  │  3. Update Vectorstore (Python/LangChain)            │   │
│  │  4. Refresh Index (API call)                         │   │
│  │  5. Cleanup                                           │   │
│  └───────────────────┬──────────────────────────────────┘   │
└──────────────────────┼──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              RAG System (localhost:8000)                     │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Vectorstore (FAISS)                                 │   │
│  │  - index.faiss (updated with new recipes)            │   │
│  │  - index.pkl (metadata)                              │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  File Watcher (detects index.faiss changes)          │   │
│  │  → Automatically reloads in-memory index             │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Pipeline Stages Breakdown

#### 1. **Initialize**
- Checks out code from GitHub
- Compares current commit with previous
- Filters only Yocto files (`.bb`, `.bbappend`, `.conf`, `.inc`, `.bbclass`)
- Counts changed files
- Skips if no Yocto files changed

#### 2. **Stage Documents**
- Creates temporary directory: `vectorstore/yocto-staging`
- Copies changed files preserving directory structure
- Prepares for embedding process

#### 3. **Update Vectorstore**
- Activates Python virtual environment
- Uses LangChain to load documents
- Splits into 1000-character chunks (200 overlap)
- Generates embeddings via Ollama (nomic-embed-text)
- Updates or creates FAISS index
- Saves to disk (`index.faiss` + `index.pkl`)

#### 4. **Refresh Index**
- Calls RAG API `/api/rebuild` endpoint
- Triggers in-memory reload of vectorstore
- Falls back to file watcher if API unreachable

#### 5. **Cleanup**
- Removes temporary staging directory
- Cleans workspace
- Preserves vectorstore files

***

## Prerequisites

### System Requirements

```
Jenkins Server:
- Jenkins 2.300+
- Java 11+
- Git installed
- Python 3.10+ with pip
- Network access to GitHub
- Network access to RAG API

Required Jenkins Plugins:
- Pipeline (workflow-aggregator)
- Git Plugin
- Generic Webhook Trigger Plugin
- Pipeline: Groovy

RAG System:
- Running on same server or accessible network
- Port 8000 accessible
- Vectorstore directory writable
- Python virtual environment configured
```

### Network Requirements

```
GitHub → Jenkins: Webhook on port 8080 (or your Jenkins port)
Jenkins → RAG API: HTTP on port 8000
Jenkins → GitHub: HTTPS on port 443 (for git clone)
```

***

## GitHub Repository Setup

### Step 1: Create/Prepare Your Repository

Your repository should contain:
```
your-yocto-repo/
├── recipes-custom/
│   ├── hello-world/
│   │   └── hello-world_1.0.bb
│   └── custom-image/
│       └── custom-image.bb
├── conf/
│   ├── layer.conf
│   └── local.conf.sample
├── classes/
│   └── custom.bbclass
├── Jenkinsfile              # ← This pipeline
└── README.md
```

### Step 2: Add Jenkinsfile to Repository

Create `Jenkinsfile` in repository root:

```bash
cd /path/to/your-yocto-repo

# Create Jenkinsfile
cat > Jenkinsfile << 'EOF'
# Paste the entire pipeline code here
EOF

# Commit and push
git add Jenkinsfile
git commit -m "Add auto-sync pipeline for vectorstore"
git push origin main
```

### Step 3: Verify File Patterns

The pipeline filters these file extensions:
- `*.bb` - BitBake recipes
- `*.bbappend` - Recipe extensions
- `*.conf` - Configuration files
- `*.inc` - Include files
- `*.bbclass` - BitBake classes
- `layer.conf` - Special case

**To add more patterns**, edit the pipeline:

```groovy
regexpFilterExpression: '.*(\\.bb|\\.bbappend|\\.conf|layer\\.conf|\\.inc|\\.bbclass|\\.YOUR_EXT).*'
```

***

## Jenkins Setup

### Step 1: Install Required Plugins

1. Navigate to **Manage Jenkins** → **Manage Plugins**
2. Go to **Available** tab
3. Search and install:
   - Generic Webhook Trigger Plugin
   - Pipeline Plugin (if not installed)
   - Git Plugin (if not installed)

4. Click **Install without restart**
5. Wait for installation to complete

### Step 2: Create Pipeline Job

1. **Create New Item**
   - Click **"New Item"** in Jenkins dashboard
   - Enter name: `Yocto-Vectorstore-Sync`
   - Select: **"Pipeline"**
   - Click **"OK"**

2. **General Configuration**
   - ✅ Check **"GitHub project"**
   - Project url: `https://github.com/your-org/your-yocto-repo`

3. **Build Triggers**
   - The pipeline already has `GenericTrigger` defined in code
   - Note the token: `yocto-build-sync` (you'll need this for webhook)

4. **Pipeline Configuration**
   - Definition: **"Pipeline script from SCM"**
   - SCM: **"Git"**
   - Repository URL: `https://github.com/your-org/your-yocto-repo.git`
   - Credentials: Add your GitHub credentials if repo is private
   - Branch: `*/main` (or your default branch)
   - Script Path: `Jenkinsfile`

5. **Save** the configuration

### Step 3: Configure Pipeline Parameters

The pipeline has these default parameters (can be customized):

```groovy
RAG_BASE = '/home/azureuser/rag-system'
VECTORSTORE_DIR = 'modular_code_base/vectorstore'
PYTHON_ENV = '/home/azureuser/rag-system/rag_env'
API_ENDPOINT = 'http://localhost:8000'
```

**To customize:**
- Click **"Configure"** on the pipeline
- Scroll to **"This project is parameterized"**
- Modify default values
- Click **"Save"**

### Step 4: Test Manual Trigger

Before setting up webhook, test manually:

1. Click **"Build Now"**
2. Watch console output
3. Verify stages complete successfully
4. Check vectorstore files updated:

```bash
ls -lht /home/azureuser/rag-system/modular_code_base/vectorstore/
# Should show recent timestamps on index.faiss and index.pkl
```

***

## GitHub Webhook Configuration

### Step 1: Get Jenkins Webhook URL

Your webhook URL format:
```
http://YOUR_JENKINS_IP:8080/generic-webhook-trigger/invoke?token=yocto-build-sync
```

**Example:**
```
http://20.6.35.135:8080/generic-webhook-trigger/invoke?token=yocto-build-sync
```

**Components:**
- `YOUR_JENKINS_IP`: Your Jenkins server IP or domain
- `8080`: Jenkins port (default)
- `token=yocto-build-sync`: Must match token in pipeline

### Step 2: Configure Webhook in GitHub

1. **Open Repository Settings**
   - Go to your GitHub repository
   - Click **"Settings"** tab
   - Click **"Webhooks"** in left sidebar

2. **Add Webhook**
   - Click **"Add webhook"** button
   
3. **Configure Webhook**
   ```
   Payload URL: http://YOUR_JENKINS_IP:8080/generic-webhook-trigger/invoke?token=yocto-build-sync
   Content type: application/json
   Secret: (leave empty or add shared secret)
   SSL verification: Disable (if using HTTP)
   
   Which events would you like to trigger this webhook?
   ○ Just the push event (SELECTED)
   
   ✅ Active
   ```

4. **Save** webhook

### Step 3: Test Webhook

1. **Make a test commit:**
   ```bash
   cd your-yocto-repo
   echo "# Test" >> recipes-custom/test.bb
   git add recipes-custom/test.bb
   git commit -m "Test webhook trigger"
   git push origin main
   ```

2. **Check GitHub webhook delivery:**
   - Go to **Settings** → **Webhooks**
   - Click on your webhook
   - Click **"Recent Deliveries"** tab
   - Should see a delivery with ✓ (green checkmark)

3. **Check Jenkins:**
   - Pipeline should auto-trigger within seconds
   - Build #2 (or next number) should appear
   - Console output should show "Triggered by Yocto file changes"

***

## Network & Security Setup

### Scenario 1: Jenkins on Secure VM (Private IP)

If your Jenkins is behind firewall/security group:

#### Option A: Allow GitHub IP Ranges

**GitHub's webhook IP ranges** (as of 2025):
```
192.30.252.0/22
185.199.108.0/22
140.82.112.0/20
143.55.64.0/20
2a0a:a440::/29
2606:50c0::/32
```

**Configure firewall rules:**

```bash
# Ubuntu/Debian (UFW)
sudo ufw allow from 192.30.252.0/22 to any port 8080
sudo ufw allow from 185.199.108.0/22 to any port 8080
sudo ufw allow from 140.82.112.0/20 to any port 8080
sudo ufw allow from 143.55.64.0/20 to any port 8080

# Azure Network Security Group
az network nsg rule create \
  --resource-group your-rg \
  --nsg-name your-nsg \
  --name AllowGitHubWebhook \
  --priority 100 \
  --source-address-prefixes 192.30.252.0/22 185.199.108.0/22 140.82.112.0/20 143.55.64.0/20 \
  --destination-port-ranges 8080 \
  --access Allow \
  --protocol Tcp

# AWS Security Group
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 8080 \
  --cidr 192.30.252.0/22

# Repeat for other IP ranges
```

**Verify GitHub IPs** (they change occasionally):
- Check: https://api.github.com/meta
- Look for "hooks" IP ranges

#### Option B: Use Webhook Proxy (Recommended)

Use **GitHub Actions + Jenkins API** as proxy:

**Create `.github/workflows/trigger-jenkins.yml`:**

```yaml
name: Trigger Jenkins

on:
  push:
    paths:
      - '**.bb'
      - '**.bbappend'
      - '**.conf'
      - '**.inc'
      - '**.bbclass'

jobs:
  trigger:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Jenkins Pipeline
        run: |
          curl -X POST "http://YOUR_JENKINS_IP:8080/generic-webhook-trigger/invoke?token=yocto-build-sync" \
            -H "Content-Type: application/json" \
            -d '{
              "ref": "${{ github.ref }}",
              "commits": [{
                "modified": ["${{ github.event.commits[0].modified }}"],
                "added": ["${{ github.event.commits[0].added }}"]
              }]
            }'
```

This runs on GitHub's infrastructure and triggers your Jenkins.

### Scenario 2: Jenkins on Public IP

If Jenkins has public IP:

```bash
# Simply allow port 8080 from anywhere (less secure)
sudo ufw allow 8080/tcp

# Or restrict to GitHub IPs (more secure, use Option A above)
```

### Scenario 3: Using ngrok (Development/Testing)

For temporary/local testing:

```bash
# Install ngrok
wget https://bin.equinox.io/c/bNyj1mQVY4c/ngrok-v3-stable-linux-amd64.tgz
tar xvzf ngrok-v3-stable-linux-amd64.tgz
sudo mv ngrok /usr/local/bin/

# Start tunnel
ngrok http 8080

# Use the ngrok URL in GitHub webhook:
# https://xxxx-xx-xxx-xxx-xx.ngrok-free.app/generic-webhook-trigger/invoke?token=yocto-build-sync
```

### Security Best Practices

1. **Use HTTPS** (configure Jenkins with SSL certificate)
2. **Add webhook secret** for request validation
3. **Restrict IP ranges** to GitHub only
4. **Use Jenkins authentication** (don't allow anonymous triggers)
5. **Monitor webhook logs** for suspicious activity

***

## Testing the Integration

### End-to-End Test

#### Test 1: Simple Recipe Update

```bash
cd your-yocto-repo

# Create/modify a recipe
cat > recipes-custom/test/test-recipe.bb << 'EOF'
SUMMARY = "Test recipe for webhook"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "file://${COMMON_LICENSE_DIR}/MIT;md5=xxx"

do_install() {
    echo "Test installation"
}
EOF

# Commit and push
git add recipes-custom/test/test-recipe.bb
git commit -m "Add test recipe"
git push origin main
```

**Expected Result:**
- GitHub webhook fires immediately
- Jenkins pipeline triggers within 10 seconds
- Build #X starts automatically
- Console shows: "Processing 1 Yocto files"
- Vectorstore updated
- Build completes successfully

#### Test 2: Multiple File Changes

```bash
# Modify several files
echo "# Comment" >> recipes-custom/recipe1.bb
echo "# Comment" >> conf/local.conf
echo "# Comment" >> classes/custom.bbclass

git add -A
git commit -m "Update multiple Yocto files"
git push origin main
```

**Expected Result:**
- Pipeline processes 3 files
- All embedded into vectorstore
- Console shows: "Processing 3 Yocto files"

#### Test 3: Non-Yocto Files (Should Skip)

```bash
# Add non-Yocto file
echo "Documentation" > README.md
git add README.md
git commit -m "Update documentation"
git push origin main
```

**Expected Result:**
- Webhook fires
- Pipeline starts
- "No Yocto files changed" message
- Build marked as NOT_BUILT
- Vectorstore NOT updated

### Verification Steps

**1. Check Jenkins Build:**
```
Build #X
Status: SUCCESS
Duration: ~2 minutes
Console Output: No errors
```

**2. Verify Vectorstore Files:**
```bash
ls -lht /home/azureuser/rag-system/modular_code_base/vectorstore/

# Expected output (recent timestamps):
-rw-rw-r-- 1 user user 1.5M Nov 20 15:30 index.faiss
-rw-rw-r-- 1 user user 250K Nov 20 15:30 index.pkl
```

**3. Test RAG API:**
```bash
curl -X POST http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "messages": [{"role": "user", "content": "What is in test-recipe.bb?"}]
  }'

# Should return information about your newly added recipe
```

**4. Check File Watcher Logs:**
```bash
# If RAG API running in foreground:
# Should see:
# "File changed: ./vectorstore/index.faiss"
# "Vectorstore reloaded from disk"
```

***

## Troubleshooting

### Issue 1: Webhook Not Triggering Pipeline

**Symptom:**
- Push to GitHub
- No build starts in Jenkins

**Diagnosis:**

```bash
# Check GitHub webhook delivery
GitHub → Settings → Webhooks → Recent Deliveries
Look for: Response code

# Check Jenkins logs
tail -f /var/log/jenkins/jenkins.log | grep generic-webhook
```

**Solutions:**

| Problem | Solution |
|---------|----------|
| Response: Connection refused | Check Jenkins is running: `systemctl status jenkins` |
| Response: 404 Not Found | Verify webhook URL format and token |
| Response: Timeout | Check firewall rules (see Network Setup) |
| No delivery attempts | Webhook not saved properly, re-add it |

### Issue 2: Pipeline Triggers but Fails

**Symptom:**
```
Stage 'Initialize' - SUCCESS
Stage 'Stage Documents' - SUCCESS
Stage 'Update Vectorstore' - FAILURE
```

**Check these:**

```bash
# 1. Python environment exists
ls -la /home/azureuser/rag-system/rag_env/

# 2. Required packages installed
source /home/azureuser/rag-system/rag_env/bin/activate
pip list | grep -E "langchain|faiss|ollama"

# 3. Ollama is running
ollama list
systemctl status ollama

# 4. Permissions on vectorstore directory
ls -ld /home/azureuser/rag-system/modular_code_base/vectorstore/
# Should be writable by Jenkins user
```

**Fix permissions:**
```bash
sudo chown -R jenkins:jenkins /home/azureuser/rag-system/modular_code_base/vectorstore/
sudo chmod -R 755 /home/azureuser/rag-system/modular_code_base/vectorstore/
```

### Issue 3: "No Yocto files changed"

**Symptom:**
```
Build result: NOT_BUILT
Reason: No Yocto files changed
```

**Cause:** You pushed non-Yocto files (e.g., README.md, docs/)

**Solution:** This is expected behavior. Only pushes with `.bb`, `.conf`, etc. will process.

**To verify filtering:**
```bash
# Check what files were in the commit
git show --name-only

# Verify they match the pattern
echo "test.bb" | grep -E '\.(bb|bbappend|conf|inc|bbclass)$'  # Should match
echo "README.md" | grep -E '\.(bb|bbappend|conf|inc|bbclass)$'  # Should NOT match
```

### Issue 4: API Refresh Fails

**Symptom:**
```
Stage 'Refresh Index':
WARNING: API endpoint unreachable, index will auto-refresh on file detection
```

**Diagnosis:**
```bash
# Check RAG API status
curl http://localhost:8000/health

# If fails, check API is running
ps aux | grep "python.*app.main"

# Check API logs
tail -f /home/azureuser/rag-system/modular_code_base/app.log
```

**Solution:**

This is actually **NOT critical** because:
1. File watcher will detect `index.faiss` change
2. Vectorstore reloads automatically within 3 seconds
3. No manual intervention needed

**To fix if you want explicit API reload:**
```bash
# Restart RAG API
cd /home/azureuser/rag-system/modular_code_base
source ../rag_env/bin/activate
python -m app.main
```

### Issue 5: Jenkins Out of Disk Space

**Symptom:**
```
ERROR: No space left on device
```

**Check:**
```bash
df -h

# Check Jenkins workspace
du -sh /var/lib/jenkins/workspace/*
```

**Solution:**
```bash
# Clean old workspaces
cd /var/lib/jenkins/workspace
rm -rf Yocto-Vectorstore-Sync@*  # Removes temp branches

# Clean old builds (automatic via buildDiscarder in pipeline)
# Keeps last 50 builds only

# Increase disk or add cleanup
```

***

## Maintenance

**Monitor webhook deliveries:**
```bash
# Check GitHub webhook page
# Look for failed deliveries
# Investigate any 4xx/5xx errors
```



**Review build history:**
```bash
# Check Jenkins pipeline
# Look for patterns:
# - Are most builds successful?
# - Any recurring failures?
# - Average build time stable?
```

**Update GitHub IP whitelist:**
```bash
# GitHub occasionally changes IPs
# Check: https://api.github.com/meta
# Update firewall rules if needed
```

**Clean vectorstore:**
```bash
# Check size
du -sh /home/azureuser/rag-system/modular_code_base/vectorstore/

# If too large (>5GB), consider rebuild
# Or implement periodic cleanup of old chunks
```

### Backup Strategy

```bash
# Backup vectorstore (before major changes)
cd /home/azureuser/rag-system/modular_code_base
tar -czf vectorstore-backup-$(date +%Y%m%d).tar.gz vectorstore/

# Backup Jenkins job config
cp /var/lib/jenkins/jobs/Yocto-Vectorstore-Sync/config.xml \
   config-backup-$(date +%Y%m%d).xml
```

***

## Advanced Configuration

### Custom File Patterns

Add more file extensions to sync:

```groovy
// In pipeline properties
regexpFilterExpression: '.*(\\.bb|\\.bbappend|\\.conf|\\.inc|\\.bbclass|\\.YOUR_EXT).*'
```

### Conditional Processing

Only process specific branches:

```groovy
// Add in Initialize stage
if (env.GIT_BRANCH != 'main' && env.GIT_BRANCH != 'develop') {
    echo "Skipping non-main branch: ${env.GIT_BRANCH}"
    currentBuild.result = 'NOT_BUILT'
    error("Branch filter")
}
```

### Notifications

Add Slack notifications:

```groovy
post {
    success {
        slackSend(
            color: 'good',
            message: "Vectorstore updated: ${env.FILE_COUNT} files from ${env.GIT_COMMIT[0..7]}"
        )
    }
    failure {
        slackSend(
            color: 'danger',
            message: "Failed to update vectorstore: ${env.BUILD_URL}"
        )
    }
}
```

### Parallel Processing

For large changesets:

```groovy
// In Update Vectorstore stage
parallel {
    stage('Embed Files 1-50') {
        steps {
            // Process first batch
        }
    }
    stage('Embed Files 51-100') {
        steps {
            // Process second batch
        }
    }
}
```

***

## Summary

This automated pipeline provides **seamless synchronization** between your Yocto project and AI knowledge base:

### What You Achieved

✅ **Zero Manual Work** - Push code, vectorstore updates automatically  
✅ **Real-time Sync** - Changes reflected in AI within 2-3 minutes  
✅ **Smart Filtering** - Only processes relevant Yocto files  
✅ **Production Ready** - Handles failures gracefully, auto-retries  
✅ **Traceable** - Full audit trail in Jenkins build history  



**Your Yocto knowledge base now auto-updates with every commit!**