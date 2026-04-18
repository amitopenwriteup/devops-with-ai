# 🤖 AI in Jenkins & GitHub Actions Workshop
### *Powered by Microsoft Copilot Studio (Free Tier)*

---

## 📋 Workshop Overview

| Detail | Info |
|--------|------|
| **Duration** | 4 Hours |
| **Level** | Intermediate |
| **Tools** | Jenkins, GitHub Actions, Microsoft Copilot Studio (Free) |
| **Prerequisites** | Basic CI/CD knowledge, GitHub account |

---

## 🎯 Learning Objectives

By the end of this workshop, participants will be able to:

- Set up Microsoft Copilot Studio (free tier) and connect it to CI/CD pipelines
- Use AI to generate Jenkins pipelines and GitHub Actions workflows from plain English
- Automatically detect and debug failed pipeline runs using an AI copilot
- Integrate a Copilot Studio bot into GitHub via webhooks

---

## 🧰 Pre-Workshop Setup

### 1. Create a Free Copilot Studio Account

1. Go to [https://copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Sign in with a **free Microsoft account** (outlook.com / hotmail.com)
3. Select **"Start free"** — no credit card required
4. You will land on the home screen: **"What would you like to build?"**
5. Click the **`Agent`** button *(previously called "Copilot" — Microsoft renamed this in 2025)*
6. In the description box type:
   > *"You are a DevOps assistant that generates Jenkins pipelines and GitHub Actions workflows, and debugs failed CI/CD errors"*
7. Click the **→ arrow** — Copilot Studio will auto-create your agent
8. The agent will be named **`CI CD Assistant`** automatically

> **💡 UI Note:** The term **"Copilot"** has been replaced by **"Agent"** throughout the Copilot Studio interface. Wherever this workshop says "Copilot", read it as "Agent".

> **💡 Free Tier Limits:** 25,000 messages/month — more than enough for this workshop and personal projects.

#### Terminology Reference

| This Workshop Says | Actual UI Shows |
|--------------------|----------------|
| Create a new Copilot | Click **Agent** button |
| Copilot | Agent |
| Topics | Topics *(unchanged)* |
| Generative answers node | Create generative answers *(unchanged)* |

### 2. Tools Required

```bash
# Check Java (for Jenkins)
java -version   # Requires Java 17+

# Check GitHub CLI
gh --version

# Node.js (for webhook bridge)
node --version  # Requires v18+
```

### 3. Install Jenkins (Local / Docker)

```bash
# Option A: Docker (Recommended)
docker run -d \
  --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts

# Option B: Direct Install (Ubuntu)
sudo apt update
sudo apt install openjdk-17-jdk
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo apt-key add -
sudo apt install jenkins
```

---

## 🏗️ Module 1: AI-Assisted Pipeline Creation

**Duration: 90 minutes**

### 1.1 — Configure Copilot Studio for Pipeline Generation

#### Step 1: Create a Custom Topic in Copilot Studio

1. Open your **CI CD Assistant** agent
2. Click **Topics** in the top navigation bar
3. Click **+ New Topic → "From blank"**
4. Name the topic: `Generate Pipeline`
5. In the **"Describe what the topic does"** box (shown under *"The agent chooses"*), type:
   ```
   User asks to create a pipeline
   ```

> **💡 UI Note:** The new Copilot Studio UI uses a **description field** (*"Describe what the topic does"*) instead of a list of trigger phrases. The agent's AI automatically matches user messages to topics based on this description.

#### Step 2: Design the Conversation Flow

Click the **`+`** button below the trigger node to add nodes:

```
[The agent chooses]  ← auto-generated trigger node
  "User asks to create a pipeline"
     ↓  (+)
[Question node]  ← change Message → Question via ···  menu
  "What language/framework is your project?"
  Quick reply options: Node.js | Python | Java | .NET | Go | Other
  Save response as variable: {Topic.Language}
     ↓  (+)
[Question node]
  "Which CI/CD platform?"
  Quick reply options: Jenkins | GitHub Actions | Both
  Save response as variable: {Topic.Platform}
     ↓  (+)
[Create generative answers]  ← Add via + → Advanced → Generative answers
  Input: Activity.Text
  (configure settings — see Step 3)
```

> **💡 UI Tip:** The first node added after the trigger is a **Message** node by default. Click its **`···`** menu → **"Change to Question"** to convert it so it captures the user's answer into a variable.

#### Step 3: Configure the "Create Generative Answers" Node

Click **`+`** → **Advanced** → **Generative answers**. The node shows two fields: **Input** and **Data sources**.

**A) Fix the Input field** *(fixes "Missing required property 'UserInput'" error)*

Click inside the **"Enter or select a value"** Input box → click the **`···`** button → select **System variable** → **`Activity.Text`**

```
Input
┌─────────────────────────────────┐
│  Activity.Text                  │  ← set this
└─────────────────────────────────┘
```

**B) Configure Data Sources — click `Edit`**

The Properties panel will open on the right. Configure exactly as follows:

| Setting | Value |
|---------|-------|
| Search only selected sources | ⚫ **OFF** |
| Web search | 🔵 **ON** — enables searching public websites |
| Allow the AI to use its own general knowledge *(preview)* | 🔵 **ON** ← critical |
| Content moderation level → Customize | ☐ leave unchecked |

**C) Customize Your Prompt** *(paste into the 0/8000 prompt box)*

```
You are a DevOps expert. The user wants a CI/CD pipeline.

Generate a complete, production-ready pipeline based on their request:
- GitHub Actions → output valid YAML
- Jenkins → output valid Declarative Groovy Jenkinsfile
- Include build, test, lint, docker, deploy stages
- Add inline comments on every stage
- Never hardcode secrets — always use environment variables
- Wrap output in a code block
```

**D) Click Save** (top right of the topic editor)

> ⚠️ **Common Error:** *"Missing required property 'UserInput'"* appears when the Input field is empty. Setting it to `Activity.Text` resolves this immediately.

---

### 1.2 — Live Demo: Generate a Node.js Pipeline

**User tells the Agent:**
> *"Create a GitHub Actions workflow for a Node.js app with build, test, and Docker push steps"*

**Agent generates:**

```yaml
# 🤖 Generated by CI-CD-Assistant (Copilot Studio)
# GitHub Actions Workflow — Node.js CI/CD Pipeline

name: Node.js CI/CD

on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main" ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    name: 🔨 Build & Test
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: [18.x, 20.x]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run linter
        run: npm run lint

      - name: Run tests with coverage
        run: npm test -- --coverage

      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  docker-push:
    name: 🐳 Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: build-and-test
    if: github.ref == 'refs/heads/main'

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha
            type=ref,event=branch
            latest

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

  notify:
    name: 📬 Notify Team
    runs-on: ubuntu-latest
    needs: [build-and-test, docker-push]
    if: always()

    steps:
      - name: Send Slack notification
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,workflow
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

### 1.3 — Generate a Jenkins Pipeline via the Agent

**User tells the Agent:**
> *"Write a Jenkins pipeline for a Python Flask app with pytest, Docker build, and deploy to staging"*

**Agent generates:**

```groovy
// 🤖 Generated by CI-CD-Assistant (Copilot Studio)
// Declarative Jenkins Pipeline — Python Flask App

pipeline {
    agent any

    environment {
        DOCKER_REGISTRY  = 'your-registry.azurecr.io'
        IMAGE_NAME       = 'flask-app'
        STAGING_SERVER   = credentials('staging-server-url')
        DOCKER_CREDS     = credentials('docker-registry-creds')
        PYTHON_VERSION   = '3.11'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    triggers {
        githubPush()
    }

    stages {

        stage('📥 Checkout') {
            steps {
                checkout scm
                sh 'git log --oneline -5'
            }
        }

        stage('🐍 Setup Python') {
            steps {
                sh """
                    python${PYTHON_VERSION} -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install -r requirements-dev.txt
                """
            }
        }

        stage('🔍 Lint & Format') {
            parallel {
                stage('Flake8') {
                    steps {
                        sh '. venv/bin/activate && flake8 app/ --max-line-length=120'
                    }
                }
                stage('Black Check') {
                    steps {
                        sh '. venv/bin/activate && black --check app/'
                    }
                }
            }
        }

        stage('🧪 Run Tests') {
            steps {
                sh """
                    . venv/bin/activate
                    pytest tests/ \
                        --cov=app \
                        --cov-report=xml:coverage.xml \
                        --cov-report=html:htmlcov \
                        --junitxml=test-results.xml \
                        -v
                """
            }
            post {
                always {
                    junit 'test-results.xml'
                    publishHTML([
                        allowMissing: false,
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('🐳 Docker Build') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            steps {
                script {
                    def imageTag = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-creds') {
                        def image = docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${imageTag}")
                        image.push()
                        image.push('latest')
                        env.IMAGE_TAG = imageTag
                    }
                }
            }
        }

        stage('🚀 Deploy to Staging') {
            when { branch 'develop' }
            steps {
                sh """
                    ssh deploy@${STAGING_SERVER} \
                    "docker pull ${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} && \
                     docker-compose -f /opt/app/docker-compose.staging.yml up -d"
                """
            }
        }

    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
            slackSend(
                color: 'good',
                message: "✅ *${env.JOB_NAME}* #${env.BUILD_NUMBER} succeeded! (<${env.BUILD_URL}|View>)"
            )
        }
        failure {
            echo "❌ Pipeline failed!"
            slackSend(
                color: 'danger',
                message: "❌ *${env.JOB_NAME}* #${env.BUILD_NUMBER} FAILED! (<${env.BUILD_URL}|View Logs>)"
            )
            // 🔔 Trigger Copilot auto-debug (see Module 2)
            sh 'curl -X POST $COPILOT_WEBHOOK_URL -d \'{"jobName":"${JOB_NAME}","buildUrl":"${BUILD_URL}"}\''
        }
        always {
            cleanWs()
        }
    }
}
```

---

## 🐛 Module 2: Auto-Debugging Failed Pipelines

**Duration: 90 minutes**

### 2.1 — How the Auto-Debug System Works

```
┌─────────────────────────────────────────────────────────────┐
│                    AUTO-DEBUG ARCHITECTURE                  │
│                                                             │
│  Pipeline Fails                                             │
│       │                                                     │
│       ▼                                                     │
│  Webhook fires ──► Node.js Bridge ──► Copilot Studio API   │
│                         │                    │             │
│                    Fetch Logs           AI Analysis         │
│                         │                    │             │
│                         └────────────────────┘             │
│                                  │                          │
│                                  ▼                          │
│                    Debug Report Generated                   │
│                         │                                   │
│               ┌─────────┼─────────┐                        │
│               ▼         ▼         ▼                        │
│           GitHub       Slack    Email                       │
│           Comment    Message  Notification                  │
└─────────────────────────────────────────────────────────────┘
```

---

### 2.2 — Create the "Debug Pipeline" Topic in Copilot Studio

#### Step 1: New Topic → `Auto Debug Failed Pipeline`

1. Go to **Topics → + New Topic → "From blank"**
2. Name it: `Auto Debug Failed Pipeline`
3. In the **"Describe what the topic does"** field, type:
   ```
   User wants to debug or analyze a failed CI/CD pipeline error log
   ```

#### Step 2: Conversation + AI Analysis Flow

```
[The agent chooses]
  "User wants to debug a failed pipeline"
     ↓  (+)
[Question node]
  "Paste your error log or build URL below"
  Save response as: {Topic.ErrorLog}
     ↓  (+)
[Create generative answers]
  Input: Activity.Text
  Data sources:
    ✅ Web search ON
    ✅ Allow AI general knowledge ON
  Custom prompt: (see Step 3 below)
     ↓  (+)
[Message node]
  "Would you like a suggested fix? (yes/no)"
```

#### Step 3: Paste into the "Customize your prompt" box (0/8000)

In the **Create generative answers** node → click **Edit** on Data sources → scroll to the prompt box and paste:

```
You are a senior DevOps engineer and CI/CD expert.
Analyze the pipeline error log provided and respond with:

1. ROOT CAUSE — What exactly failed and why (1-2 sentences)
2. ERROR TYPE — Classify as one of:
   [Dependency] [Config] [Permissions] [Network] [Test Failure]
   [Docker] [Timeout] [Syntax] [Secret/Credential] [Resource]
3. AFFECTED STAGE — Which pipeline stage failed
4. SEVERITY — Critical / High / Medium / Low
5. STEP-BY-STEP FIX — Numbered list of exact commands or config changes
6. PREVENTION — How to prevent this in future (add to pipeline)
7. FIXED SNIPPET — Corrected pipeline code block for the failing stage

Keep the tone technical but clear. Format output with markdown headers.
```

---

### 2.3 — Real Error Examples & AI Debug Responses

#### ❌ Error Example 1: Missing Dependency

**Pipeline Log:**
```
FAILED: npm run build
npm ERR! Cannot find module '@babel/core'
npm ERR! Require stack:
npm ERR! - /app/node_modules/.bin/babel
Exit code: 1
```

**Copilot Debug Response:**

```
## 🔍 Debug Report — Build Failure

### Root Cause
The build failed because `@babel/core` is missing from node_modules.
This usually means `npm ci` was skipped or `package-lock.json` is out of sync.

### Error Type: [Dependency]
### Affected Stage: Build
### Severity: High

### Step-by-Step Fix
1. Verify `@babel/core` is in your `package.json` devDependencies
2. Run `npm install --save-dev @babel/core` locally
3. Commit the updated `package-lock.json`
4. Ensure pipeline uses `npm ci` (not `npm install`) for reproducibility

### Prevention
Add a dependency validation step before build:

### Fixed Pipeline Snippet
```yaml
- name: Validate dependencies
  run: |
    npm ci
    node -e "require('@babel/core')" && echo "✅ Dependencies OK"

- name: Build
  run: npm run build
```
```

---

#### ❌ Error Example 2: Docker Permission Denied

**Pipeline Log:**
```
ERROR: permission denied while trying to connect to the Docker daemon socket
at unix:///var/run/docker.sock: dial unix /var/run/docker.sock:
connect: permission denied
```

**Copilot Debug Response:**

```
## 🔍 Debug Report — Docker Permission Error

### Root Cause
The Jenkins agent user does not have permission to access the Docker socket.
The user running the pipeline is not in the `docker` group.

### Error Type: [Permissions]
### Affected Stage: Docker Build
### Severity: Critical

### Step-by-Step Fix
1. On the Jenkins agent server, run:
   sudo usermod -aG docker jenkins
2. Restart the Jenkins service:
   sudo systemctl restart jenkins
3. Verify: id jenkins  (should show docker group)
4. If using Docker-in-Docker, add privileged: true to agent config

### Prevention
Add a pre-flight check stage to your Jenkinsfile:
```groovy
stage('Pre-flight Check') {
    steps {
        sh 'docker info || (echo "Docker not accessible!" && exit 1)'
    }
}
```
```

---

#### ❌ Error Example 3: GitHub Actions Secret Missing

**Pipeline Log:**
```
Error: Missing required environment variable: AWS_ACCESS_KEY_ID
Error: HttpError: Bad credentials (401)
Deploy step failed with exit code 1
```

**Copilot Debug Response:**

```
## 🔍 Debug Report — Missing Secret / Credential

### Root Cause
The workflow references `AWS_ACCESS_KEY_ID` but it is not configured
in the repository's GitHub Secrets, causing a 401 authentication error.

### Error Type: [Secret/Credential]
### Affected Stage: Deploy
### Severity: Critical

### Step-by-Step Fix
1. Go to your repo → Settings → Secrets and variables → Actions
2. Click "New repository secret"
3. Add: AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
4. For org-wide use, add to Organization Secrets instead
5. If using OIDC (recommended), replace static keys:

### Improved Workflow (Using OIDC — No Long-lived Keys)
```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS credentials via OIDC
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
      aws-region: us-east-1
```

### Prevention
Use `secrets.GITHUB_TOKEN` where possible and audit secrets quarterly.
```

---

### 2.4 — Webhook Bridge: Connect GitHub Actions to Copilot

Create a lightweight Node.js webhook bridge:

**`webhook-bridge/server.js`**

```javascript
// Webhook Bridge — GitHub Actions ↔ Copilot Studio
const express = require('express');
const axios   = require('axios');
const app     = express();

app.use(express.json());

const COPILOT_DIRECTLINE_SECRET = process.env.COPILOT_DIRECTLINE_SECRET;
const GITHUB_TOKEN              = process.env.GITHUB_TOKEN;

// GitHub Actions sends failure events here
app.post('/pipeline-failed', async (req, res) => {
    const { workflow, repo, run_id, logs_url, sha } = req.body;

    console.log(`🔴 Pipeline failed: ${workflow} in ${repo}`);

    try {
        // 1. Fetch the failed job logs from GitHub
        const logsRes = await axios.get(logs_url, {
            headers: { Authorization: `Bearer ${GITHUB_TOKEN}` }
        });

        const errorLog = logsRes.data.slice(-3000); // Last 3000 chars

        // 2. Send to Copilot Studio via Direct Line API
        const tokenRes = await axios.post(
            'https://directline.botframework.com/v3/directline/tokens/generate',
            {},
            { headers: { Authorization: `Bearer ${COPILOT_DIRECTLINE_SECRET}` } }
        );

        const dlToken = tokenRes.data.token;

        // 3. Start conversation with Copilot
        const convRes = await axios.post(
            'https://directline.botframework.com/v3/directline/conversations',
            {},
            { headers: { Authorization: `Bearer ${dlToken}` } }
        );

        const conversationId = convRes.data.conversationId;

        // 4. Send error log to Copilot for analysis
        await axios.post(
            `https://directline.botframework.com/v3/directline/conversations/${conversationId}/activities`,
            {
                type: 'message',
                from: { id: 'webhook-bridge' },
                text: `Analyze this pipeline failure:\n\nWorkflow: ${workflow}\nRepo: ${repo}\n\nError Log:\n${errorLog}`
            },
            { headers: { Authorization: `Bearer ${dlToken}` } }
        );

        // 5. Poll for Copilot response
        await new Promise(resolve => setTimeout(resolve, 5000));

        const activitiesRes = await axios.get(
            `https://directline.botframework.com/v3/directline/conversations/${conversationId}/activities`,
            { headers: { Authorization: `Bearer ${dlToken}` } }
        );

        const botReply = activitiesRes.data.activities
            .filter(a => a.from.id !== 'webhook-bridge')
            .map(a => a.text)
            .join('\n');

        // 6. Post debug report as GitHub commit comment
        await axios.post(
            `https://api.github.com/repos/${repo}/commits/${sha}/comments`,
            {
                body: `## 🤖 AI Debug Report (Copilot Studio)\n\n${botReply}\n\n---\n*Auto-generated by CI-CD-Assistant*`
            },
            { headers: { Authorization: `Bearer ${GITHUB_TOKEN}` } }
        );

        console.log('✅ Debug report posted to GitHub');
        res.json({ status: 'ok', message: 'Debug report posted' });

    } catch (err) {
        console.error('❌ Bridge error:', err.message);
        res.status(500).json({ error: err.message });
    }
});

app.listen(3000, () => console.log('🚀 Webhook bridge running on :3000'));
```

**`package.json`**

```json
{
  "name": "copilot-cicd-bridge",
  "version": "1.0.0",
  "dependencies": {
    "axios": "^1.6.0",
    "express": "^4.18.0"
  }
}
```

---

### 2.5 — GitHub Actions: Send Failure to Copilot

Add this job to any existing workflow to enable auto-debugging:

```yaml
# Add to your existing workflow file
  ai-debug-on-failure:
    name: 🤖 AI Auto-Debug
    runs-on: ubuntu-latest
    needs: [build-and-test]   # Reference your failing job name
    if: failure()             # Only runs when previous job fails

    steps:
      - name: Send failure details to Copilot Studio
        env:
          WEBHOOK_URL: ${{ secrets.COPILOT_WEBHOOK_URL }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          curl -X POST "$WEBHOOK_URL/pipeline-failed" \
            -H "Content-Type: application/json" \
            -d '{
              "workflow": "${{ github.workflow }}",
              "repo": "${{ github.repository }}",
              "run_id": "${{ github.run_id }}",
              "sha": "${{ github.sha }}",
              "logs_url": "https://api.github.com/repos/${{ github.repository }}/actions/runs/${{ github.run_id }}/logs"
            }'
```

---

## ⚙️ Module 3: Jenkins + Copilot Integration

**Duration: 30 minutes**

### 3.1 — Jenkins Shared Library with Copilot Calls

Create a reusable shared library step:

**`vars/copilotDebug.groovy`** (in your Jenkins shared library repo)

```groovy
// Jenkins Shared Library — Copilot Auto-Debug Step
def call(Map config = [:]) {
    def webhookUrl  = config.webhookUrl  ?: env.COPILOT_WEBHOOK_URL
    def jobName     = config.jobName     ?: env.JOB_NAME
    def buildUrl    = config.buildUrl    ?: env.BUILD_URL
    def buildNumber = config.buildNumber ?: env.BUILD_NUMBER

    echo "🤖 Triggering Copilot Studio auto-debug..."

    def payload = groovy.json.JsonOutput.toJson([
        jobName    : jobName,
        buildUrl   : buildUrl,
        buildNumber: buildNumber,
        logsUrl    : "${buildUrl}consoleText"
    ])

    sh """
        curl -s -X POST '${webhookUrl}/jenkins-failed' \\
          -H 'Content-Type: application/json' \\
          -d '${payload}' \\
          || echo 'Warning: Could not reach Copilot webhook'
    """
}
```

**Use in any Jenkinsfile:**

```groovy
@Library('your-shared-library') _

pipeline {
    agent any
    stages {
        stage('Build') {
            steps { sh 'npm run build' }
        }
    }
    post {
        failure {
            // One-line AI debug trigger
            copilotDebug()
        }
    }
}
```

---

## 🧪 Module 4: Hands-On Lab Exercises

**Duration: 30 minutes**

### Lab 1 — Generate Your Own Pipeline (15 min)

1. Open your **CI CD Assistant** agent in Copilot Studio
2. Click **"Test your agent"** panel on the right
3. Type: *"Create a GitHub Actions workflow for a Java Spring Boot app with Maven build, JUnit tests, and deployment to Azure App Service"*
4. Copy the generated YAML into `.github/workflows/main.yml`
5. Push to your repo and verify the workflow runs

**Checkpoint:** Workflow file created ✅ | Pipeline triggered ✅ | Tests pass ✅

---

### Lab 2 — Simulate and Debug a Failure (15 min)

Intentionally break your pipeline:

```yaml
# Break it by changing this:
- name: Install dependencies
  run: npm ci

# To this (introduces a typo):
- name: Install dependencies
  run: npm ci --wrong-flag
```

1. Commit and push → watch the pipeline fail
2. Copy the error from the Actions log
3. Ask the Agent in the **"Test your agent"** panel: *"Debug this error: [paste log]"*
4. Apply Copilot's suggested fix
5. Push the fix → verify pipeline turns green

**Checkpoint:** Error identified ✅ | Fix applied ✅ | Pipeline green ✅

---

## 📊 Workshop Summary

| Feature | Copilot Studio Free | What It Enables |
|---------|--------------------:|----------------|
| Pipeline generation from English | ✅ | Skip boilerplate, start smart |
| Error log analysis | ✅ | Instant root cause in seconds |
| GitHub comment integration | ✅ | Debug reports on every failure |
| Jenkins webhook support | ✅ | Works with any pipeline |
| Monthly message limit | 25,000 | Enough for any team |
| Custom AI instructions | ✅ | Tune for your tech stack |

---

## 🔗 Resources & Next Steps

- **Copilot Studio Docs:** https://learn.microsoft.com/en-us/microsoft-copilot-studio/
- **GitHub Actions Docs:** https://docs.github.com/en/actions
- **Jenkins Pipeline Syntax:** https://www.jenkins.io/doc/book/pipeline/syntax/
- **Direct Line API:** https://learn.microsoft.com/en-us/azure/bot-service/rest-api/bot-framework-rest-direct-line-3-0-concepts

### Extend This Workshop

- Add **Teams** or **Slack** notification when the Agent posts a debug report
- Train the Agent on your **custom error patterns** using Knowledge Sources
- Use **Power Automate** (free tier) to orchestrate multi-step debug workflows
- Publish the Agent to **Microsoft Teams** for team-wide access (free)

---

*Workshop created for DevOps teams exploring AI-augmented CI/CD workflows.*
*All tools used are available on free tiers — no credit card required.*
