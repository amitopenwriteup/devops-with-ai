# 🚀 Jenkins CI/CD Pipeline + Docker + Kubernetes + AI Agent Lab
## Workshop Guide — End-to-End DevOps with Intelligent Automation

---

## 📋 Table of Contents

1. [Workshop Overview](#1-workshop-overview)
2. [Prerequisites & Environment Setup](#2-prerequisites--environment-setup)
3. [Lab 1 — Jenkins Installation & Configuration](#3-lab-1--jenkins-installation--configuration)
4. [Lab 2 — Dockerizing Your Application](#4-lab-2--dockerizing-your-application)
5. [Lab 3 — Jenkins Pipeline with Docker Build](#5-lab-3--jenkins-pipeline-with-docker-build)
6. [Lab 4 — Kubernetes Cluster Setup](#6-lab-4--kubernetes-cluster-setup)
7. [Lab 5 — Deploying to Kubernetes via Jenkins](#7-lab-5--deploying-to-kubernetes-via-jenkins)
8. [Lab 6 — AI Agent Integration](#8-lab-6--ai-agent-integration)
9. [Lab 7 — Full End-to-End AI-Powered Pipeline](#9-lab-7--full-end-to-end-ai-powered-pipeline)
10. [Troubleshooting & Reference](#10-troubleshooting--reference)

---

## 1. Workshop Overview

### 🎯 Objectives

By the end of this workshop, you will be able to:

- Set up a production-grade Jenkins CI/CD server
- Build and push Docker images automatically on every commit
- Deploy containerized applications to a Kubernetes cluster
- Integrate an AI Agent that performs automated code review, security scanning, and deployment decisions
- Build a fully autonomous pipeline that self-heals and self-reports

### 🏗️ Architecture Diagram

```
Developer Push
     │
     ▼
┌─────────────────┐
│   GitHub Repo   │──── Webhook ────────────────────────┐
└─────────────────┘                                      │
                                                         ▼
                                              ┌─────────────────────┐
                                              │   Jenkins Master    │
                                              │  ┌───────────────┐  │
                                              │  │  Pipeline DSL │  │
                                              │  └───────┬───────┘  │
                                              └──────────┼──────────┘
                          ┌───────────────────────────────┤
                          │                               │
                          ▼                               ▼
               ┌─────────────────┐             ┌─────────────────────┐
               │  Docker Build   │             │    AI Agent Layer   │
               │  & Push to      │             │  ┌───────────────┐  │
               │  Registry       │             │  │ Code Review   │  │
               └────────┬────────┘             │  │ Sec Scanning  │  │
                        │                      │  │ Deploy Guard  │  │
                        ▼                      └──────────┬──────────┘
               ┌─────────────────┐                        │
               │  Kubernetes     │◄───────────────────────┘
               │  Cluster        │
               │  ┌───────────┐  │
               │  │  Dev NS   │  │
               │  │  Staging  │  │
               │  │  Prod NS  │  │
               └─────────────────┘
```

### 📦 Tech Stack

| Component      | Technology              | Version     |
|----------------|-------------------------|-------------|
| CI/CD Server   | Jenkins                 | 2.440+      |
| Containers     | Docker                  | 25.x        |
| Orchestration  | Kubernetes (k3s/EKS)    | 1.29+       |
| Registry       | Docker Hub / ECR        | —           |
| AI Agent       | Python + Anthropic API  | Claude 3.x  |
| SCM            | GitHub                  | —           |
| Monitoring     | Prometheus + Grafana    | —           |

---

## 2. Prerequisites & Environment Setup

### 🖥️ Hardware Requirements

| Role           | CPU  | RAM   | Disk   |
|----------------|------|-------|--------|
| Jenkins Master | 2    | 4 GB  | 30 GB  |
| Worker Node x2 | 2    | 4 GB  | 20 GB  |
| K8s Control    | 2    | 4 GB  | 20 GB  |
| K8s Worker x2  | 2    | 4 GB  | 20 GB  |

### 🔧 Software Prerequisites

```bash
# Verify installations on each node
docker --version          # Docker 25.x+
kubectl version --client  # v1.29+
java --version            # Java 17+
git --version             # 2.x+
python3 --version         # 3.10+
pip3 --version
```

### 🔑 Accounts & Access Needed

- GitHub account with a sample repo
- Docker Hub account (or AWS ECR access)
- Anthropic API key (for AI Agent lab)
- SSH access to all servers

### 📁 Workshop Repository Structure

```
workshop-repo/
├── app/
│   ├── app.py                  # Sample Python Flask app
│   ├── requirements.txt
│   └── tests/
│       └── test_app.py
├── Dockerfile
├── docker-compose.yml
├── Jenkinsfile                 # Main pipeline definition
├── k8s/
│   ├── namespace.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── hpa.yaml
├── ai-agent/
│   ├── agent.py               # AI Agent script
│   ├── requirements.txt
│   └── prompts/
│       ├── code_review.txt
│       ├── security_scan.txt
│       └── deploy_decision.txt
└── scripts/
    ├── setup-jenkins.sh
    ├── setup-k8s.sh
    └── rollback.sh
```

---

## 3. Lab 1 — Jenkins Installation & Configuration

### Step 1.1 — Install Jenkins on Ubuntu

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Java 17
sudo apt install -y openjdk-17-jdk

# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key \
  | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins

# Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

### Step 1.2 — Initial Jenkins Setup

```bash
# Retrieve the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open browser: `http://<your-server-ip>:8080`

1. Paste the initial admin password
2. Select **"Install suggested plugins"**
3. Create your admin user
4. Set Jenkins URL

### Step 1.3 — Install Required Plugins

Navigate to: **Manage Jenkins → Plugins → Available**

Install the following plugins:

```
✅ Pipeline
✅ Docker Pipeline
✅ Docker Commons
✅ Kubernetes CLI
✅ Kubernetes
✅ GitHub Integration
✅ Git
✅ Credentials Binding
✅ Blue Ocean (optional, for better UI)
✅ Slack Notification (optional)
✅ JUnit
✅ HTML Publisher
```

### Step 1.4 — Configure Docker Access for Jenkins

```bash
# Add jenkins user to docker group
sudo usermod -aG docker jenkins

# Restart Jenkins to apply
sudo systemctl restart jenkins

# Verify
sudo -u jenkins docker ps
```

### Step 1.5 — Configure Credentials in Jenkins

Navigate to: **Manage Jenkins → Credentials → System → Global credentials**

Add the following secrets:

| ID                    | Type             | Description              |
|-----------------------|------------------|--------------------------|
| `docker-hub-creds`    | Username/Password| Docker Hub login         |
| `github-token`        | Secret text      | GitHub PAT               |
| `kubeconfig`          | Secret file      | Kubernetes config file   |
| `anthropic-api-key`   | Secret text      | AI Agent API key         |

---

## 4. Lab 2 — Dockerizing Your Application

### Step 2.1 — Sample Flask Application

```python
# app/app.py
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        "message": "Hello from CI/CD Workshop!",
        "version": os.getenv("APP_VERSION", "1.0.0"),
        "environment": os.getenv("ENVIRONMENT", "development")
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy"}), 200

@app.route('/ready')
def ready():
    return jsonify({"status": "ready"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

```text
# app/requirements.txt
flask==3.0.0
gunicorn==21.2.0
pytest==7.4.0
pytest-cov==4.1.0
```

### Step 2.2 — Write the Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim AS base

# Security: run as non-root user
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

WORKDIR /app

# Install dependencies first (layer caching)
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ .

# Change ownership
RUN chown -R appuser:appgroup /app

USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:5000/health || exit 1

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

### Step 2.3 — Docker Compose for Local Testing

```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build: .
    ports:
      - "5000:5000"
    environment:
      - APP_VERSION=1.0.0
      - ENVIRONMENT=development
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Optional: local registry for testing
  registry:
    image: registry:2
    ports:
      - "5001:5000"
    volumes:
      - registry-data:/var/lib/registry

volumes:
  registry-data:
```

### Step 2.4 — Build and Test Locally

```bash
# Build the image
docker build -t workshop-app:latest .

# Run locally
docker run -p 5000:5000 workshop-app:latest

# Test endpoints
curl http://localhost:5000/
curl http://localhost:5000/health
curl http://localhost:5000/ready

# Run unit tests inside container
docker run --rm workshop-app:latest pytest tests/ -v

# Inspect image layers
docker history workshop-app:latest

# Scan for vulnerabilities (optional: install trivy first)
trivy image workshop-app:latest
```

---

## 5. Lab 3 — Jenkins Pipeline with Docker Build

### Step 3.1 — Jenkinsfile (Basic Pipeline)

```groovy
// Jenkinsfile
pipeline {
    agent any

    environment {
        DOCKER_IMAGE     = "yourdockerhub/workshop-app"
        DOCKER_TAG       = "${env.BUILD_NUMBER}-${env.GIT_COMMIT[0..7]}"
        DOCKER_REGISTRY  = "https://index.docker.io/v1/"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
    }

    stages {

        stage('🔍 Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_MSG = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()
                    echo "Commit: ${env.GIT_COMMIT_MSG}"
                }
            }
        }

        stage('🧪 Unit Tests') {
            steps {
                sh '''
                    python3 -m venv venv
                    source venv/bin/activate
                    pip install -r app/requirements.txt
                    pytest app/tests/ -v \
                      --junitxml=test-results/results.xml \
                      --cov=app \
                      --cov-report=xml:coverage.xml \
                      --cov-report=html:coverage-html
                '''
            }
            post {
                always {
                    junit 'test-results/results.xml'
                    publishHTML([
                        reportDir: 'coverage-html',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('🔒 Security Scan') {
            steps {
                sh '''
                    # Install and run bandit (Python security linter)
                    pip install bandit
                    bandit -r app/ -f json -o bandit-report.json || true
                    cat bandit-report.json
                '''
            }
        }

        stage('🐳 Docker Build') {
            steps {
                script {
                    docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                    docker.build("${DOCKER_IMAGE}:latest")
                }
            }
        }

        stage('🔬 Docker Image Scan') {
            steps {
                sh '''
                    # Scan with Trivy (install separately or use docker)
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy:latest image \
                      --exit-code 0 \
                      --severity HIGH,CRITICAL \
                      ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
        }

        stage('📤 Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                        docker logout
                    '''
                }
            }
        }

    }

    post {
        success {
            echo "✅ Pipeline succeeded! Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
        always {
            cleanWs()
        }
    }
}
```

### Step 3.2 — Create Jenkins Pipeline Job

1. Go to Jenkins dashboard → **New Item**
2. Enter name: `workshop-pipeline`
3. Select **Pipeline** → OK
4. Under **Pipeline** section:
   - Definition: `Pipeline script from SCM`
   - SCM: `Git`
   - Repository URL: your GitHub repo
   - Credentials: `github-token`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`
5. Under **Build Triggers**: check `GitHub hook trigger for GITScm polling`
6. Save and click **Build Now**

### Step 3.3 — Configure GitHub Webhook

In GitHub repo: **Settings → Webhooks → Add webhook**

```
Payload URL: http://<jenkins-ip>:8080/github-webhook/
Content type: application/json
Events: Push events, Pull request events
```

---

## 6. Lab 4 — Kubernetes Cluster Setup

### Step 4.1 — Install k3s (Lightweight Kubernetes)

```bash
# On control plane node
curl -sfL https://get.k3s.io | sh -

# Get node token
sudo cat /var/lib/rancher/k3s/server/node-token

# On each worker node (replace TOKEN and SERVER_IP)
curl -sfL https://get.k3s.io | K3S_URL=https://<SERVER_IP>:6443 \
  K3S_TOKEN=<TOKEN> sh -

# Verify cluster
sudo kubectl get nodes -o wide
```

### Step 4.2 — Configure kubectl

```bash
# Copy kubeconfig to local
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

# Or set KUBECONFIG
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Verify
kubectl cluster-info
kubectl get nodes
kubectl get pods -A
```

### Step 4.3 — Kubernetes Manifests

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: workshop-dev
  labels:
    environment: development
---
apiVersion: v1
kind: Namespace
metadata:
  name: workshop-staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: workshop-prod
  labels:
    environment: production
```

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  namespace: workshop-dev          # parameterized in pipeline
  labels:
    app: workshop-app
    version: "1.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0            # zero-downtime deployment
  template:
    metadata:
      labels:
        app: workshop-app
    spec:
      containers:
        - name: workshop-app
          image: yourdockerhub/workshop-app:latest    # replaced by pipeline
          ports:
            - containerPort: 5000
          env:
            - name: ENVIRONMENT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: environment
            - name: APP_VERSION
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['version']
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 15
            periodSeconds: 20
          readinessProbe:
            httpGet:
              path: /ready
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 10
```

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: workshop-app-svc
  namespace: workshop-dev
spec:
  selector:
    app: workshop-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: ClusterIP
---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: workshop-app-hpa
  namespace: workshop-dev
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: workshop-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: workshop-dev
data:
  environment: "development"
  log_level: "INFO"
```

---

## 7. Lab 5 — Deploying to Kubernetes via Jenkins

### Step 5.1 — Extended Jenkinsfile with K8s Deploy

```groovy
// Jenkinsfile (extended with K8s deployment stages)
pipeline {
    agent any

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['dev', 'staging', 'prod'],
            description: 'Target deployment environment'
        )
        booleanParam(
            name: 'RUN_AI_REVIEW',
            defaultValue: true,
            description: 'Enable AI Agent review before deploy'
        )
    }

    environment {
        DOCKER_IMAGE    = "yourdockerhub/workshop-app"
        DOCKER_TAG      = "${env.BUILD_NUMBER}-${env.GIT_COMMIT[0..7]}"
        K8S_NAMESPACE   = "workshop-${params.DEPLOY_ENV}"
        APP_VERSION     = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('🔍 Checkout') {
            steps { checkout scm }
        }

        stage('🧪 Test') {
            steps {
                sh '''
                    python3 -m venv venv
                    source venv/bin/activate
                    pip install -r app/requirements.txt
                    pytest app/tests/ -v --junitxml=test-results/results.xml
                '''
            }
            post {
                always { junit 'test-results/results.xml' }
            }
        }

        stage('🐳 Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker logout
                    '''
                }
            }
        }

        stage('🤖 AI Agent Review') {
            when {
                expression { params.RUN_AI_REVIEW == true }
            }
            steps {
                withCredentials([string(
                    credentialsId: 'anthropic-api-key',
                    variable: 'ANTHROPIC_API_KEY'
                )]) {
                    script {
                        def aiResult = sh(
                            script: '''
                                cd ai-agent
                                pip install -r requirements.txt -q
                                python3 agent.py \
                                  --mode code-review \
                                  --path ../app/ \
                                  --output ai-review.json
                            ''',
                            returnStdout: true
                        ).trim()

                        def review = readJSON file: 'ai-agent/ai-review.json'
                        echo "🤖 AI Review Score: ${review.score}/10"
                        echo "📋 Summary: ${review.summary}"

                        if (review.score < 6 && params.DEPLOY_ENV == 'prod') {
                            error("❌ AI Agent blocked production deploy. Score: ${review.score}/10")
                        }
                    }
                }
            }
        }

        stage('🚀 Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        # Apply namespace
                        kubectl apply -f k8s/namespace.yaml

                        # Apply configmap
                        kubectl apply -f k8s/configmap.yaml \
                          -n ${K8S_NAMESPACE}

                        # Update image in deployment
                        sed -i "s|yourdockerhub/workshop-app:latest|${DOCKER_IMAGE}:${DOCKER_TAG}|g" \
                          k8s/deployment.yaml

                        # Apply deployment
                        kubectl apply -f k8s/deployment.yaml \
                          -n ${K8S_NAMESPACE}

                        # Apply service and HPA
                        kubectl apply -f k8s/service.yaml \
                          -n ${K8S_NAMESPACE}
                        kubectl apply -f k8s/hpa.yaml \
                          -n ${K8S_NAMESPACE}

                        # Wait for rollout
                        kubectl rollout status deployment/workshop-app \
                          -n ${K8S_NAMESPACE} \
                          --timeout=5m
                    '''
                }
            }
        }

        stage('✅ Smoke Tests') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        # Port-forward and test
                        kubectl port-forward \
                          svc/workshop-app-svc 8888:80 \
                          -n ${K8S_NAMESPACE} &

                        sleep 5
                        curl -f http://localhost:8888/health || \
                          (echo "❌ Health check failed!" && exit 1)
                        curl -f http://localhost:8888/ready || \
                          (echo "❌ Readiness check failed!" && exit 1)

                        echo "✅ Smoke tests passed!"
                        kill %1 || true
                    '''
                }
            }
        }

    }

    post {
        success {
            echo "🎉 Deployed ${DOCKER_IMAGE}:${DOCKER_TAG} to ${K8S_NAMESPACE}"
        }
        failure {
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh '''
                    echo "🔄 Rolling back deployment..."
                    kubectl rollout undo deployment/workshop-app \
                      -n ${K8S_NAMESPACE} || true
                '''
            }
        }
        always {
            cleanWs()
        }
    }
}
```

### Step 5.2 — Zero-Downtime Deployment Verification

```bash
# Watch rollout in real-time
kubectl rollout status deployment/workshop-app -n workshop-dev -w

# Check revision history
kubectl rollout history deployment/workshop-app -n workshop-dev

# Manual rollback if needed
kubectl rollout undo deployment/workshop-app -n workshop-dev
kubectl rollout undo deployment/workshop-app -n workshop-dev --to-revision=2

# View pod logs
kubectl logs -l app=workshop-app -n workshop-dev --tail=50 -f

# Describe deployment events
kubectl describe deployment workshop-app -n workshop-dev
```

---

## 8. Lab 6 — AI Agent Integration

### Step 6.1 — AI Agent Architecture

The AI Agent acts as an intelligent gate in the pipeline with three responsibilities:

```
┌─────────────────────────────────────────────────────────┐
│                     AI Agent Layer                       │
│                                                          │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────┐ │
│  │   Code Review   │  │  Security Scan  │  │ Deploy  │ │
│  │                 │  │                 │  │ Guard   │ │
│  │ • Code quality  │  │ • Vuln patterns │  │         │ │
│  │ • Best practice │  │ • Secret leaks  │  │ Risk    │ │
│  │ • Complexity    │  │ • OWASP top 10  │  │ Score   │ │
│  │ • Suggestions   │  │ • CVE lookup    │  │ GO/NOGO │ │
│  └────────┬────────┘  └────────┬────────┘  └────┬────┘ │
│           │                    │                 │       │
│           └────────────────────┴─────────────────┘       │
│                               │                           │
│                        ┌──────▼──────┐                   │
│                        │   Report    │                   │
│                        │  Generator  │                   │
│                        └─────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### Step 6.2 — AI Agent Python Script

```python
# ai-agent/agent.py
import os
import sys
import json
import argparse
import anthropic
from pathlib import Path

def read_code_files(path: str) -> str:
    """Read all Python files from the given path."""
    code_content = []
    for py_file in Path(path).rglob("*.py"):
        if "__pycache__" in str(py_file):
            continue
        content = py_file.read_text()
        code_content.append(f"### File: {py_file}\n```python\n{content}\n```")
    return "\n\n".join(code_content)


def run_code_review(code_path: str, client: anthropic.Anthropic) -> dict:
    """Use Claude to review code quality."""
    code = read_code_files(code_path)

    prompt = f"""You are a senior software engineer conducting a code review.

Analyze the following Python code and provide:
1. A quality score from 1-10
2. Critical issues (blocking)
3. Major issues (should fix)
4. Minor suggestions
5. A brief summary

Return ONLY valid JSON in this exact format:
{{
    "score": <number>,
    "critical_issues": ["<issue1>", ...],
    "major_issues": ["<issue1>", ...],
    "suggestions": ["<suggestion1>", ...],
    "summary": "<brief summary>",
    "approved": <true/false>
}}

Code to review:
{code}
"""

    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}]
    )

    response_text = message.content[0].text
    return json.loads(response_text)


def run_security_scan(code_path: str, client: anthropic.Anthropic) -> dict:
    """Use Claude to identify security vulnerabilities."""
    code = read_code_files(code_path)

    prompt = f"""You are a security expert performing a security audit.

Analyze the following Python code for security vulnerabilities including:
- SQL injection, XSS, command injection
- Hardcoded secrets or credentials
- Insecure deserialization
- Improper error handling that exposes internals
- OWASP Top 10 vulnerabilities
- Insecure dependencies usage

Return ONLY valid JSON in this exact format:
{{
    "risk_level": "<LOW|MEDIUM|HIGH|CRITICAL>",
    "vulnerabilities": [
        {{
            "type": "<vulnerability type>",
            "severity": "<LOW|MEDIUM|HIGH|CRITICAL>",
            "file": "<filename>",
            "description": "<description>",
            "recommendation": "<how to fix>"
        }}
    ],
    "secrets_found": <true/false>,
    "summary": "<brief summary>",
    "block_deployment": <true/false>
}}

Code to scan:
{code}
"""

    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2000,
        messages=[{"role": "user", "content": prompt}]
    )

    response_text = message.content[0].text
    return json.loads(response_text)


def run_deploy_decision(
    code_review: dict,
    security_scan: dict,
    environment: str,
    client: anthropic.Anthropic
) -> dict:
    """AI makes final deployment decision."""

    prompt = f"""You are a DevOps lead making deployment approval decisions.

Environment: {environment}

Code Review Results:
{json.dumps(code_review, indent=2)}

Security Scan Results:
{json.dumps(security_scan, indent=2)}

Based on the above, make a deployment decision for the '{environment}' environment.
Be strict for production, lenient for development.

Return ONLY valid JSON:
{{
    "decision": "<APPROVE|REJECT|APPROVE_WITH_CONDITIONS>",
    "confidence": <0-100>,
    "reason": "<explanation>",
    "conditions": ["<condition if any>"],
    "risk_assessment": "<LOW|MEDIUM|HIGH>",
    "recommended_actions": ["<action1>", ...]
}}
"""

    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1000,
        messages=[{"role": "user", "content": prompt}]
    )

    response_text = message.content[0].text
    return json.loads(response_text)


def main():
    parser = argparse.ArgumentParser(description='AI Agent for CI/CD Pipeline')
    parser.add_argument('--mode', choices=['code-review', 'security', 'deploy-decision', 'full'],
                        required=True, help='Operation mode')
    parser.add_argument('--path', default='../app/', help='Path to code')
    parser.add_argument('--environment', default='dev', help='Target environment')
    parser.add_argument('--output', default='ai-report.json', help='Output file')
    args = parser.parse_args()

    api_key = os.environ.get('ANTHROPIC_API_KEY')
    if not api_key:
        print("❌ ANTHROPIC_API_KEY not set", file=sys.stderr)
        sys.exit(1)

    client = anthropic.Anthropic(api_key=api_key)
    results = {}

    print(f"🤖 AI Agent starting in '{args.mode}' mode...")

    if args.mode in ['code-review', 'full']:
        print("📋 Running code review...")
        results['code_review'] = run_code_review(args.path, client)
        print(f"   Score: {results['code_review']['score']}/10")

    if args.mode in ['security', 'full']:
        print("🔒 Running security scan...")
        results['security_scan'] = run_security_scan(args.path, client)
        print(f"   Risk Level: {results['security_scan']['risk_level']}")

    if args.mode in ['deploy-decision', 'full'] and len(results) >= 2:
        print("🚦 Making deployment decision...")
        results['deploy_decision'] = run_deploy_decision(
            results.get('code_review', {}),
            results.get('security_scan', {}),
            args.environment,
            client
        )
        print(f"   Decision: {results['deploy_decision']['decision']}")

    # Combine into final output
    final_output = {
        **results.get('code_review', {}),
        **results,
        "score": results.get('code_review', {}).get('score', 5),
        "summary": results.get('code_review', {}).get('summary', 'Review complete')
    }

    output_path = f"ai-agent/{args.output}"
    with open(output_path, 'w') as f:
        json.dump(final_output, f, indent=2)

    print(f"✅ AI Agent report saved to {output_path}")

    # Exit with error if deployment should be blocked
    if results.get('security_scan', {}).get('block_deployment', False):
        print("🚨 Security scan recommends blocking deployment!", file=sys.stderr)
        sys.exit(2)

    decision = results.get('deploy_decision', {}).get('decision', 'APPROVE')
    if decision == 'REJECT':
        print("🚫 AI Agent rejected this deployment!", file=sys.stderr)
        sys.exit(3)


if __name__ == '__main__':
    main()
```

```text
# ai-agent/requirements.txt
anthropic>=0.25.0
```

### Step 6.3 — Test the AI Agent Locally

```bash
cd ai-agent

# Install dependencies
pip install -r requirements.txt

# Set API key
export ANTHROPIC_API_KEY="your-api-key-here"

# Run code review
python3 agent.py --mode code-review --path ../app/ --output review.json
cat review.json | python3 -m json.tool

# Run security scan
python3 agent.py --mode security --path ../app/ --output security.json
cat security.json | python3 -m json.tool

# Run full analysis
python3 agent.py --mode full --path ../app/ --environment prod --output full-report.json
```

---

## 9. Lab 7 — Full End-to-End AI-Powered Pipeline

### Step 7.1 — Complete Production Jenkinsfile

```groovy
// Jenkinsfile.production
pipeline {
    agent { label 'docker-agent' }

    parameters {
        choice(name: 'DEPLOY_ENV',
               choices: ['dev', 'staging', 'prod'],
               description: 'Deployment target')
        booleanParam(name: 'AI_REVIEW',    defaultValue: true,  description: 'Run AI Agent review')
        booleanParam(name: 'AI_SECURITY',  defaultValue: true,  description: 'Run AI security scan')
        booleanParam(name: 'SKIP_TESTS',   defaultValue: false, description: 'Skip tests (emergency only)')
        string(name: 'ROLLBACK_VERSION',   defaultValue: '',    description: 'Rollback to this version (leave blank for new deploy)')
    }

    environment {
        DOCKER_IMAGE   = credentials('docker-image-name')
        DOCKER_TAG     = "${env.BUILD_NUMBER}-${GIT_COMMIT[0..7]}"
        K8S_NAMESPACE  = "workshop-${params.DEPLOY_ENV}"
        SLACK_CHANNEL  = '#deployments'
        BUILD_URL_SAFE = "${env.BUILD_URL}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 45, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
    }

    stages {

        // ──────────────────────────────────────────────
        stage('🔄 Rollback?') {
            when {
                expression { params.ROLLBACK_VERSION != '' }
            }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh """
                        echo "⏪ Rolling back to version ${params.ROLLBACK_VERSION}..."
                        kubectl set image deployment/workshop-app \
                          workshop-app=${DOCKER_IMAGE}:${params.ROLLBACK_VERSION} \
                          -n ${K8S_NAMESPACE}
                        kubectl rollout status deployment/workshop-app \
                          -n ${K8S_NAMESPACE} --timeout=5m
                        echo "✅ Rollback complete"
                    """
                }
            }
        }

        // ──────────────────────────────────────────────
        stage('🔍 Checkout & Validate') {
            when { expression { params.ROLLBACK_VERSION == '' } }
            steps {
                checkout scm
                sh '''
                    echo "Branch: $(git branch --show-current)"
                    echo "Commit: $(git log -1 --oneline)"
                    echo "Author: $(git log -1 --format='%an <%ae>')"
                    echo "Files changed: $(git diff --name-only HEAD~1 HEAD | wc -l)"
                '''
            }
        }

        // ──────────────────────────────────────────────
        stage('🧪 Tests & Coverage') {
            when {
                allOf {
                    expression { params.ROLLBACK_VERSION == '' }
                    expression { params.SKIP_TESTS == false }
                }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install -r app/requirements.txt -q
                            pytest app/tests/ -v \
                              --junitxml=results/unit.xml \
                              --cov=app \
                              --cov-report=xml:results/coverage.xml \
                              --cov-fail-under=70
                        '''
                    }
                }
                stage('Lint') {
                    steps {
                        sh '''
                            pip install flake8 pylint -q
                            flake8 app/ --max-line-length=120 \
                              --format=pylint > results/flake8.txt || true
                            pylint app/*.py --output-format=json \
                              > results/pylint.json || true
                        '''
                    }
                }
            }
            post {
                always {
                    junit 'results/unit.xml'
                }
            }
        }

        // ──────────────────────────────────────────────
        stage('🤖 AI Code Review') {
            when {
                allOf {
                    expression { params.ROLLBACK_VERSION == '' }
                    expression { params.AI_REVIEW == true }
                }
            }
            steps {
                withCredentials([string(credentialsId: 'anthropic-api-key',
                                       variable: 'ANTHROPIC_API_KEY')]) {
                    script {
                        sh '''
                            pip install anthropic -q
                            python3 ai-agent/agent.py \
                              --mode code-review \
                              --path app/ \
                              --output review.json
                        '''

                        def review = readJSON file: 'ai-agent/review.json'
                        env.AI_SCORE = "${review.score}"

                        echo "╔══════════════════════════════════╗"
                        echo "║      AI Code Review Results      ║"
                        echo "╠══════════════════════════════════╣"
                        echo "║ Score: ${review.score}/10                  ║"
                        echo "║ Summary: ${review.summary}"
                        echo "╚══════════════════════════════════╝"

                        if (review.critical_issues?.size() > 0) {
                            echo "⚠️  Critical Issues Found:"
                            review.critical_issues.each { echo "   - ${it}" }
                        }

                        if (review.score < 5 && params.DEPLOY_ENV == 'prod') {
                            error("AI Agent: Code quality score ${review.score}/10 too low for production")
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────
        stage('🔒 AI Security Scan') {
            when {
                allOf {
                    expression { params.ROLLBACK_VERSION == '' }
                    expression { params.AI_SECURITY == true }
                }
            }
            steps {
                withCredentials([string(credentialsId: 'anthropic-api-key',
                                       variable: 'ANTHROPIC_API_KEY')]) {
                    script {
                        def exitCode = sh(
                            script: '''
                                python3 ai-agent/agent.py \
                                  --mode security \
                                  --path app/ \
                                  --output security.json
                            ''',
                            returnStatus: true
                        )

                        def scan = readJSON file: 'ai-agent/security.json'
                        echo "🔒 Security Risk Level: ${scan.risk_level}"

                        if (scan.risk_level == 'CRITICAL') {
                            error("🚨 CRITICAL security vulnerabilities found. Deployment blocked!")
                        }
                        if (scan.risk_level == 'HIGH' && params.DEPLOY_ENV == 'prod') {
                            error("🚨 HIGH risk vulnerabilities found. Production deployment blocked!")
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────
        stage('🐳 Build & Push Image') {
            when { expression { params.ROLLBACK_VERSION == '' } }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "🐳 Building Docker image..."
                        docker build \
                          --build-arg APP_VERSION=${DOCKER_TAG} \
                          --label "jenkins.build=${BUILD_NUMBER}" \
                          --label "git.commit=${GIT_COMMIT}" \
                          -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                          -t ${DOCKER_IMAGE}:${DEPLOY_ENV}-latest \
                          .

                        echo "📤 Pushing to registry..."
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:${DEPLOY_ENV}-latest
                        docker logout
                        echo "✅ Image pushed: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    '''
                }
            }
        }

        // ──────────────────────────────────────────────
        stage('🚦 AI Deploy Decision') {
            when {
                allOf {
                    expression { params.ROLLBACK_VERSION == '' }
                    expression { params.AI_REVIEW == true }
                    expression { params.DEPLOY_ENV == 'prod' }
                }
            }
            steps {
                withCredentials([string(credentialsId: 'anthropic-api-key',
                                       variable: 'ANTHROPIC_API_KEY')]) {
                    script {
                        sh '''
                            python3 ai-agent/agent.py \
                              --mode deploy-decision \
                              --environment prod \
                              --output decision.json
                        '''

                        def decision = readJSON file: 'ai-agent/decision.json'
                        echo "🤖 AI Deploy Decision: ${decision.decision}"
                        echo "📊 Confidence: ${decision.confidence}%"
                        echo "📝 Reason: ${decision.reason}"

                        if (decision.decision == 'REJECT') {
                            error("AI Agent rejected production deployment: ${decision.reason}")
                        }

                        if (decision.decision == 'APPROVE_WITH_CONDITIONS') {
                            echo "⚠️  Approved with conditions:"
                            decision.conditions.each { echo "   • ${it}" }
                        }
                    }
                }
            }
        }

        // ──────────────────────────────────────────────
        stage('🚀 Deploy to Kubernetes') {
            when { expression { params.ROLLBACK_VERSION == '' } }
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        echo "🚀 Deploying to ${K8S_NAMESPACE}..."

                        # Apply all manifests
                        kubectl apply -f k8s/ -n ${K8S_NAMESPACE}

                        # Update deployment image
                        kubectl set image deployment/workshop-app \
                          workshop-app=${DOCKER_IMAGE}:${DOCKER_TAG} \
                          -n ${K8S_NAMESPACE}

                        # Annotate with build info
                        kubectl annotate deployment/workshop-app \
                          jenkins.build="${BUILD_NUMBER}" \
                          git.commit="${GIT_COMMIT}" \
                          deployed.at="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                          -n ${K8S_NAMESPACE} \
                          --overwrite

                        # Wait for rollout
                        kubectl rollout status deployment/workshop-app \
                          -n ${K8S_NAMESPACE} \
                          --timeout=10m

                        echo "✅ Deployment complete!"
                        kubectl get pods -n ${K8S_NAMESPACE} -l app=workshop-app
                    '''
                }
            }
        }

        // ──────────────────────────────────────────────
        stage('✅ Smoke & Integration Tests') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl port-forward svc/workshop-app-svc 9999:80 \
                          -n ${K8S_NAMESPACE} &
                        PF_PID=$!
                        sleep 5

                        echo "Running smoke tests..."
                        curl -sf http://localhost:9999/health && echo "✅ Health OK"
                        curl -sf http://localhost:9999/ready  && echo "✅ Ready OK"
                        curl -sf http://localhost:9999/       && echo "✅ App OK"

                        kill $PF_PID || true
                        echo "✅ All smoke tests passed!"
                    '''
                }
            }
        }

    }

    // ──────────────────────────────────────────────
    post {
        success {
            echo """
            ╔═══════════════════════════════════════════╗
            ║   ✅  DEPLOYMENT SUCCESSFUL               ║
            ║   Image:  ${DOCKER_IMAGE}:${DOCKER_TAG}
            ║   Target: ${K8S_NAMESPACE}
            ║   Build:  #${BUILD_NUMBER}
            ╚═══════════════════════════════════════════╝
            """
        }
        failure {
            echo "❌ Pipeline failed — initiating auto-rollback..."
            withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh '''
                    kubectl rollout undo deployment/workshop-app \
                      -n ${K8S_NAMESPACE} || true
                    echo "🔄 Rollback complete"
                '''
            }
        }
        always {
            archiveArtifacts artifacts: 'ai-agent/*.json,results/*.xml', allowEmptyArchive: true
            cleanWs()
        }
    }
}
```

---

## 10. Troubleshooting & Reference

### 🔧 Common Issues & Fixes

| Issue | Symptom | Fix |
|-------|---------|-----|
| Docker permission denied | `Got permission denied while trying to connect to the Docker daemon` | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| kubeconfig not found | `error: stat kubeconfig: no such file` | Re-upload kubeconfig credential in Jenkins |
| Image pull error | `ErrImagePull` in K8s | Check Docker Hub credentials and image name |
| AI Agent timeout | Python script hangs | Add `--timeout 60` to API calls |
| Webhook not triggering | Pipeline not starting on push | Check GitHub webhook delivery log |
| Port-forward fails | `bind: address already in use` | Kill existing: `pkill -f "kubectl port-forward"` |

### 📊 Useful kubectl Commands

```bash
# Cluster overview
kubectl get nodes -o wide
kubectl top nodes
kubectl top pods -n workshop-dev

# Deployment management
kubectl get deployments -n workshop-dev
kubectl get pods -n workshop-dev -o wide
kubectl get events -n workshop-dev --sort-by='.lastTimestamp'

# Debugging
kubectl logs -l app=workshop-app -n workshop-dev --tail=100
kubectl exec -it <pod-name> -n workshop-dev -- /bin/sh
kubectl describe pod <pod-name> -n workshop-dev

# Resource usage
kubectl get hpa -n workshop-dev
kubectl describe hpa workshop-app-hpa -n workshop-dev
```

### 🔐 Security Checklist

```
✅ Never hardcode secrets in Jenkinsfile or Dockerfile
✅ Use Jenkins Credentials for all sensitive values
✅ Kubernetes RBAC: give Jenkins service account minimal permissions
✅ Scan Docker images before pushing (Trivy)
✅ Enable Pod Security Admission in Kubernetes
✅ Use non-root users in Docker containers
✅ Rotate API keys regularly
✅ Enable audit logging in Jenkins
✅ Use signed commits (GPG)
✅ Set resource limits on all K8s containers
```

### 📈 Pipeline Best Practices

```
✅ Use declarative pipeline syntax (not scripted)
✅ Always use `cleanWs()` in post block
✅ Set pipeline-level timeouts
✅ Archive artifacts for debugging
✅ Use parallel stages where possible
✅ Tag images with build number + git SHA (never just "latest" in prod)
✅ Always wait for rollout status before smoke tests
✅ Implement automatic rollback on failure
✅ Keep secrets out of pipeline logs
✅ Use Blue Ocean for visual pipeline debugging
```

### 🧩 Workshop Exercise Checklist

```
□ Lab 1: Jenkins installed and accessible at :8080
□ Lab 1: All plugins installed and Docker access configured
□ Lab 2: Flask app builds and runs locally
□ Lab 2: Docker image passes health check
□ Lab 3: Jenkins pipeline triggers on git push
□ Lab 3: Docker image pushed to registry
□ Lab 4: k3s cluster shows all nodes Ready
□ Lab 4: kubectl works from Jenkins server
□ Lab 5: Application deployed to workshop-dev namespace
□ Lab 5: Smoke tests pass
□ Lab 6: AI Agent produces review.json
□ Lab 6: Security scan outputs risk level
□ Lab 7: Full pipeline runs end-to-end
□ Lab 7: Auto-rollback tested by introducing a bug
```

---

## 📚 Additional Resources

- [Jenkins Official Documentation](https://www.jenkins.io/doc/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Anthropic API Reference](https://docs.anthropic.com/)
- [k3s Quick Start](https://docs.k3s.io/)
- [Trivy Vulnerability Scanner](https://trivy.dev/)

---

*Workshop Guide v1.0 — Jenkins CI/CD + Docker + Kubernetes + AI Agent*  
*Estimated Duration: 6–8 hours (full workshop) | 2–3 hours (condensed)*
