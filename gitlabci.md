# GitLab CI/CD + Ollama Local AI Agent
## Step-by-Step Workshop Guide — 100% Free, No API Keys, Runs Locally

---

## Table of Contents

1. [Workshop Overview](#1-workshop-overview)
2. [Prerequisites](#2-prerequisites)
3. [Lab 1 — Install and Run Ollama](#3-lab-1--install-and-run-ollama)
4. [Lab 2 — Pull AI Models](#4-lab-2--pull-ai-models)
5. [Lab 3 — Test Ollama API](#5-lab-3--test-ollama-api)
6. [Lab 4 — Build the AI Agent Script](#6-lab-4--build-the-ai-agent-script)
7. [Lab 5 — Connect Ollama to GitLab CI](#7-lab-5--connect-ollama-to-gitlab-ci)
8. [Lab 6 — Full .gitlab-ci.yml with Ollama AI Stages](#8-lab-6--full-gitlab-ciyml-with-ollama-ai-stages)
9. [Lab 7 — Deploy to Kind Kubernetes](#9-lab-7--deploy-to-kind-kubernetes)
10. [Lab 8 — Demo Run and Verification](#10-lab-8--demo-run-and-verification)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Workshop Overview

### What You Will Build

A complete CI/CD pipeline where Ollama runs AI models locally on your machine — no internet, no API keys, no costs — and GitLab CI calls it automatically on every git push to:

- Review code quality
- Scan for security vulnerabilities
- Make deployment approval decisions

### Architecture

```
Developer Git Push
       |
       v
+------------------+     Webhook      +------------------------+
|   GitLab Repo    | ---------------> |   GitLab Runner        |
+------------------+                  |                        |
                                      |  Stage 1: test         |
                                      |  Stage 2: docker       |
                                      |  Stage 3: ai-analysis  |---+
                                      |  Stage 4: deploy       |   |
                                      |  Stage 5: smoke        |   |
                                      +------------------------+   |
                                                                   | HTTP
                                                                   | localhost:11434
                                                         +---------v----------+
                                                         |   Ollama Server    |
                                                         |   llama3.2:1b      |
                                                         |   codellama        |
                                                         |   mistral          |
                                                         |   Runs 100% LOCAL  |
                                                         +--------------------+
                                                                   |
                                                         +---------v----------+
                                                         |  Kind Kubernetes   |
                                                         |  workshop-dev      |
                                                         |  workshop-prod     |
                                                         +--------------------+
```

### Why Ollama?

| Feature    | Ollama         | Cloud APIs            |
|------------|----------------|-----------------------|
| Cost       | FREE           | Pay per token         |
| Internet   | Not needed     | Required              |
| API Key    | None           | Required              |
| Privacy    | 100% local     | Data sent to cloud    |
| Speed      | Fast (GPU/CPU) | Depends on network    |
| Models     | 100+ available | Limited to provider   |

### Tech Stack

| Component    | Technology   | Notes                    |
|--------------|--------------|--------------------------|
| Local AI     | Ollama       | Runs models locally      |
| AI Model     | llama3.2:1b  | Fast, good quality, ~2GB |
| CI/CD        | GitLab CI    | .gitlab-ci.yml           |
| Containers   | Docker       | Already installed        |
| Kubernetes   | Kind         | Already set up           |
| Agent Script | Python 3     | Calls Ollama REST API    |

---

## 2. Prerequisites

### Already Installed (Your Setup)

```bash
# Verify your existing setup
docker --version
kubectl version --client
kind version
python3 --version        # 3.8+
git --version
```

### GitLab Runner Requirements

The GitLab Runner must run on the same machine as Ollama (or have network access to it). A self-hosted runner is required — shared GitLab.com runners cannot reach your local Ollama server.

```bash
# Verify runner is registered and running
gitlab-runner status
gitlab-runner list
```

### Minimum Hardware for Ollama

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM      | 8 GB    | 16 GB       |
| Disk     | 10 GB   | 20 GB       |
| CPU      | 4 cores | 8 cores     |
| GPU      | Not required | Speeds up 10x |

### Project Structure

```
your-repo/
+-- .gitlab-ci.yml           <- Main pipeline (Lab 6)
+-- Dockerfile               <- Your existing Dockerfile
+-- app/
|   +-- app.py
|   +-- requirements.txt
|   +-- tests/
|       +-- test_app.py
+-- ai-agent/
|   +-- ollama_agent.py      <- AI Agent script (Lab 4)
|   +-- requirements.txt
+-- k8s/
    +-- deployment.yaml
    +-- service.yaml
    +-- configmap.yaml
```

---

## 3. Lab 1 — Install and Run Ollama

### Step 1.1 — Install Ollama

**Linux (Ubuntu/Debian) — Recommended for GitLab Runner server:**

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama --version
```

**macOS:**

```bash
brew install ollama
```

**Windows:**

```
Download installer from: https://ollama.com/download/windows
Run the installer — Ollama runs as a system service
```

---

### Step 1.2 — Start Ollama Server

```bash
# Start Ollama (runs on port 11434 by default)
ollama serve

# You should see:
# time=... level=INFO msg="Listening on 127.0.0.1:11434 (version x.x.x)"
```

Keep this terminal open. Ollama must be running when GitLab CI calls it.

**To run Ollama as a background service on Linux:**

```bash
sudo systemctl status ollama
sudo systemctl start ollama
sudo systemctl enable ollama
sudo journalctl -u ollama -f
```

---

### Step 1.3 — Make Ollama Accessible to the GitLab Runner

By default Ollama only listens on `127.0.0.1`. If the runner is on the same machine this is fine. If the runner is on a different server:

```bash
sudo systemctl edit ollama

# Add in the editor:
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"

sudo systemctl daemon-reload
sudo systemctl restart ollama

# Verify
ss -tlnp | grep 11434
```

Security note: only do this in your lab/demo network. In production, restrict with a firewall rule.

---

### Step 1.4 — Verify Ollama is Running

```bash
curl http://localhost:11434/
# Expected: Ollama is running
```

---

## 4. Lab 2 — Pull AI Models

### Step 2.1 — Pull the Recommended Model

```bash
ollama pull llama3.2:1b
# Downloads ~2 GB — do this before the demo
```

### Step 2.2 — Optional: Pull Other Models

```bash
ollama pull codellama:7b    # Better code understanding, 4 GB, needs 8 GB+ RAM
ollama pull mistral:7b      # Balanced quality, 4.1 GB, needs 8 GB+ RAM

ollama list                 # See all downloaded models
```

### Step 2.3 — Test the Model Works

```bash
ollama run llama3.2:1b "Say hello in one sentence"
# Exit with: /bye
```

---

## 5. Lab 3 — Test Ollama API

### Step 3.1 — Test with curl

```bash
curl http://localhost:11434/api/generate \
  -d '{
    "model": "llama3.2:1b",
    "prompt": "Review this Python code quality in one sentence: x=1+1",
    "stream": false
  }'
```

### Step 3.2 — Test Chat Format

```bash
curl http://localhost:11434/api/chat \
  -d '{
    "model": "llama3.2:1b",
    "stream": false,
    "messages": [
      {
        "role": "system",
        "content": "You are a code reviewer. Reply only in JSON."
      },
      {
        "role": "user",
        "content": "Review: def add(a,b): return a+b. Give score 1-10 as JSON: {\"score\": N}"
      }
    ]
  }'
```

### Step 3.3 — Useful Ollama API Endpoints

```bash
# List all pulled models
curl http://localhost:11434/api/tags

# Check model info
curl http://localhost:11434/api/show \
  -d '{"name": "llama3.2:1b"}'

# Check running models
curl http://localhost:11434/api/ps

# Delete a model
curl -X DELETE http://localhost:11434/api/delete \
  -d '{"name": "llama3.2:1b"}'
```

---

## 6. Lab 4 — Build the AI Agent Script

Create `ai-agent/ollama_agent.py` in your repository:

```python
#!/usr/bin/env python3
"""
Ollama AI Agent for GitLab CI/CD Pipeline
==========================================
100% FREE -- Runs locally -- No API keys needed
Calls Ollama REST API for:
  1. Code Review and Quality Score
  2. Security Vulnerability Scan
  3. Deployment Decision (Approve / Reject)
"""

import os
import sys
import json
import time
import argparse
import urllib.request
import urllib.error
from pathlib import Path


# -- Configuration -------------------------------------------------------------
OLLAMA_HOST  = os.environ.get("OLLAMA_HOST", "http://localhost:11434")
OLLAMA_MODEL = os.environ.get("OLLAMA_MODEL", "llama3.2:1b")


# -- Helpers -------------------------------------------------------------------

def check_ollama_running():
    try:
        urllib.request.urlopen(OLLAMA_HOST, timeout=5)
        return True
    except Exception:
        print(f"ERROR: Cannot reach Ollama at {OLLAMA_HOST}", file=sys.stderr)
        print("  Make sure Ollama is running: ollama serve", file=sys.stderr)
        print("  Or check OLLAMA_HOST environment variable", file=sys.stderr)
        sys.exit(1)


def check_model_available(model: str):
    try:
        url  = f"{OLLAMA_HOST}/api/tags"
        req  = urllib.request.urlopen(url, timeout=10)
        data = json.loads(req.read())
        models = [m["name"] for m in data.get("models", [])]
        available = any(model in m or m in model for m in models)
        if not available:
            print(f"ERROR: Model '{model}' not found. Pull it first:", file=sys.stderr)
            print(f"  ollama pull {model}", file=sys.stderr)
            sys.exit(1)
        print(f"OK: Model '{model}' is available")
    except Exception as e:
        print(f"WARNING: Could not verify model (continuing): {e}", file=sys.stderr)


def call_ollama(system_prompt: str, user_prompt: str, timeout: int = 120) -> str:
    payload = json.dumps({
        "model":  OLLAMA_MODEL,
        "stream": False,
        "options": {
            "temperature": 0.1,
            "num_predict": 1024,
        },
        "messages": [
            {"role": "system", "content": system_prompt},
            {"role": "user",   "content": user_prompt},
        ],
    }).encode("utf-8")

    url = f"{OLLAMA_HOST}/api/chat"
    req = urllib.request.Request(
        url,
        data=payload,
        headers={"Content-Type": "application/json"},
        method="POST",
    )

    start = time.time()
    try:
        with urllib.request.urlopen(req, timeout=timeout) as resp:
            data = json.loads(resp.read())
            elapsed = time.time() - start
            print(f"  Ollama responded in {elapsed:.1f}s")
            return data["message"]["content"].strip()
    except urllib.error.URLError as e:
        print(f"ERROR: Ollama request failed: {e}", file=sys.stderr)
        sys.exit(1)


def safe_parse_json(text: str, fallback: dict) -> dict:
    text = text.strip()
    if "```" in text:
        parts = text.split("```")
        for part in parts:
            part = part.strip().lstrip("json").strip()
            if part.startswith("{"):
                text = part
                break

    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    start = text.find("{")
    end   = text.rfind("}")
    if start != -1 and end != -1:
        try:
            return json.loads(text[start:end+1])
        except json.JSONDecodeError:
            pass

    print(f"WARNING: Could not parse JSON, using fallback. Raw response:\n{text[:300]}")
    return fallback


def read_source_files(path: str, max_chars: int = 5000) -> str:
    exts = (".py", ".js", ".ts", ".go", ".java", ".rb", ".php")
    files = []
    for ext in exts:
        for f in Path(path).rglob(f"*{ext}"):
            if any(skip in str(f) for skip in
                   ["__pycache__", "node_modules", ".git", "venv", ".tox"]):
                continue
            try:
                content = f.read_text(errors="replace")
                files.append(f"### {f.name}\n{content}")
            except Exception:
                pass

    combined = "\n\n".join(files)
    if len(combined) > max_chars:
        combined = combined[:max_chars] + "\n\n[... truncated for context window ...]"
    return combined if combined else "# No source files found in path"


# -- Stage 1: Code Review ------------------------------------------------------

def run_code_review(code_path: str) -> dict:
    print("\n" + "-"*50)
    print("STAGE 1: AI Code Review")
    print("-"*50)

    code = read_source_files(code_path)

    system = """You are a senior software engineer doing a code review.
You MUST respond with ONLY valid JSON. No explanation. No markdown. No extra text.
Just the raw JSON object."""

    user = f"""Review the following code and return ONLY this JSON:
{{
  "score": <integer 1-10>,
  "grade": "<A|B|C|D|F>",
  "critical_issues": ["<issue description>"],
  "major_issues": ["<issue description>"],
  "suggestions": ["<improvement suggestion>"],
  "summary": "<one sentence summary>",
  "approved": <true or false>
}}

Rules:
- score 8-10 = excellent, 6-7 = good, 4-5 = needs work, 1-3 = poor
- approved = true if score >= 6

Code to review:
{code}"""

    raw    = call_ollama(system, user)
    result = safe_parse_json(raw, fallback={
        "score": 5, "grade": "C",
        "critical_issues": [], "major_issues": [],
        "suggestions": ["Review manually"],
        "summary": "AI review inconclusive",
        "approved": True
    })

    print(f"\n  Score   : {result.get('score', 5)}/10  (Grade: {result.get('grade', '?')})")
    print(f"  Summary : {result.get('summary', 'N/A')}")
    for issue in result.get("critical_issues", []):
        print(f"  CRITICAL: {issue}")
    for issue in result.get("major_issues", []):
        print(f"  MAJOR   : {issue}")
    for tip in result.get("suggestions", []):
        print(f"  TIP     : {tip}")

    return result


# -- Stage 2: Security Scan ----------------------------------------------------

def run_security_scan(code_path: str) -> dict:
    print("\n" + "-"*50)
    print("STAGE 2: AI Security Scan")
    print("-"*50)

    code = read_source_files(code_path)

    system = """You are an application security expert doing a security audit.
You MUST respond with ONLY valid JSON. No explanation. No markdown. No extra text."""

    user = f"""Scan for security vulnerabilities and return ONLY this JSON:
{{
  "risk_level": "<LOW|MEDIUM|HIGH|CRITICAL>",
  "score": <integer 1-10 where 10=most secure>,
  "vulnerabilities": [
    {{
      "type": "<vulnerability type>",
      "severity": "<LOW|MEDIUM|HIGH|CRITICAL>",
      "file": "<filename>",
      "line": "<line number or unknown>",
      "description": "<what the issue is>",
      "recommendation": "<how to fix it>"
    }}
  ],
  "secrets_found": <true or false>,
  "owasp_issues": ["<OWASP Top 10 category if applicable>"],
  "summary": "<one sentence>",
  "block_deployment": <true or false>
}}

Rules:
- block_deployment = true ONLY if CRITICAL vulnerabilities found OR secrets found
- Check for: hardcoded passwords, SQL injection, command injection,
  insecure deserialization, path traversal, XSS, weak crypto

Code to scan:
{code}"""

    raw    = call_ollama(system, user)
    result = safe_parse_json(raw, fallback={
        "risk_level": "UNKNOWN",
        "score": 5,
        "vulnerabilities": [],
        "secrets_found": False,
        "owasp_issues": [],
        "summary": "Security scan inconclusive",
        "block_deployment": False
    })

    risk = result.get("risk_level", "UNKNOWN")
    print(f"\n  Risk Level : {risk}")
    print(f"  Sec Score  : {result.get('score', '?')}/10")
    print(f"  Summary    : {result.get('summary', 'N/A')}")
    print(f"  Secrets    : {'YES - WARNING' if result.get('secrets_found') else 'None found'}")

    for v in result.get("vulnerabilities", []):
        print(f"  {v.get('severity','?')} [{v.get('type','?')}]"
              f" in {v.get('file','?')}: {v.get('description','')[:70]}")
        print(f"    Fix: {v.get('recommendation','')[:70]}")

    return result


# -- Stage 3: Deploy Decision --------------------------------------------------

def run_deploy_decision(code_review: dict, security_scan: dict, environment: str) -> dict:
    print("\n" + "-"*50)
    print(f"STAGE 3: AI Deploy Decision  [{environment.upper()}]")
    print("-"*50)

    system = """You are a DevOps lead making deployment approval decisions.
Be STRICT for production. Be lenient for dev/staging.
You MUST respond with ONLY valid JSON. No explanation. No markdown."""

    user = f"""Make a deployment decision and return ONLY this JSON:
{{
  "decision": "<APPROVE|REJECT|APPROVE_WITH_CONDITIONS>",
  "confidence": <integer 0-100>,
  "risk_assessment": "<LOW|MEDIUM|HIGH>",
  "reason": "<clear explanation max 2 sentences>",
  "conditions": ["<condition to meet if APPROVE_WITH_CONDITIONS>"],
  "recommended_actions": ["<action to take before or after deploy>"],
  "estimated_rollback_needed": <true or false>
}}

Environment target: {environment}

Code Review Results:
- Score: {code_review.get('score', 'N/A')}/10
- Grade: {code_review.get('grade', 'N/A')}
- Critical Issues: {code_review.get('critical_issues', [])}
- Major Issues: {code_review.get('major_issues', [])}
- Approved: {code_review.get('approved', True)}

Security Scan Results:
- Risk Level: {security_scan.get('risk_level', 'UNKNOWN')}
- Secrets Found: {security_scan.get('secrets_found', False)}
- Block Deployment: {security_scan.get('block_deployment', False)}
- Vulnerabilities Count: {len(security_scan.get('vulnerabilities', []))}
- Summary: {security_scan.get('summary', 'N/A')}

Decision rules:
- REJECT if: secrets found OR CRITICAL security risk OR code score < 3
- REJECT for prod if: code score < 5 OR HIGH security risk
- APPROVE_WITH_CONDITIONS if: code score 5-6 OR MEDIUM security risk
- APPROVE if: code score >= 7 AND LOW security risk
"""

    raw    = call_ollama(system, user)
    result = safe_parse_json(raw, fallback={
        "decision": "APPROVE",
        "confidence": 50,
        "risk_assessment": "MEDIUM",
        "reason": "AI decision inconclusive -- manual review recommended",
        "conditions": [],
        "recommended_actions": ["Review manually before proceeding"],
        "estimated_rollback_needed": False
    })

    decision = result.get("decision", "APPROVE")
    print(f"\n  Decision   : {decision}")
    print(f"  Confidence : {result.get('confidence', '?')}%")
    print(f"  Risk       : {result.get('risk_assessment', 'N/A')}")
    print(f"  Reason     : {result.get('reason', 'N/A')}")

    for cond in result.get("conditions", []):
        print(f"  Condition  : {cond}")
    for action in result.get("recommended_actions", []):
        print(f"  Action     : {action}")

    return result


# -- Main ----------------------------------------------------------------------

def main():
    parser = argparse.ArgumentParser(
        description="Ollama AI Agent for GitLab CI/CD -- 100% Local and Free"
    )
    parser.add_argument("--path",        default="./app",          help="Path to source code")
    parser.add_argument("--environment", default="dev",            help="Target environment")
    parser.add_argument("--output",      default="ai_report.json", help="Output JSON report file")
    parser.add_argument("--model",       default="",               help="Override Ollama model")
    args = parser.parse_args()

    global OLLAMA_MODEL
    if args.model:
        OLLAMA_MODEL = args.model

    print("\n" + "="*50)
    print("  Ollama AI Agent -- GitLab CI/CD")
    print(f"  Host  : {OLLAMA_HOST}")
    print(f"  Model : {OLLAMA_MODEL}")
    print(f"  Path  : {args.path}")
    print(f"  Target: {args.environment}")
    print("="*50)

    check_ollama_running()
    check_model_available(OLLAMA_MODEL)

    code_review   = run_code_review(args.path)
    security_scan = run_security_scan(args.path)
    deploy_dec    = run_deploy_decision(code_review, security_scan, args.environment)

    report = {
        "ollama_host":     OLLAMA_HOST,
        "ollama_model":    OLLAMA_MODEL,
        "environment":     args.environment,
        "code_review":     code_review,
        "security_scan":   security_scan,
        "deploy_decision": deploy_dec,
        "score":           code_review.get("score", 5),
        "risk_level":      security_scan.get("risk_level", "UNKNOWN"),
        "decision":        deploy_dec.get("decision", "APPROVE"),
        "summary":         code_review.get("summary", ""),
    }

    with open(args.output, "w") as fh:
        json.dump(report, fh, indent=2)

    print("\n" + "="*50)
    print(f"  Report   -> {args.output}")
    print(f"  Score    -> {report['score']}/10")
    print(f"  Risk     -> {report['risk_level']}")
    print(f"  Decision -> {report['decision']}")
    print("="*50 + "\n")

    if security_scan.get("block_deployment", False):
        print("BLOCKED: Security scan is blocking deployment!", file=sys.stderr)
        sys.exit(2)

    if deploy_dec.get("decision") == "REJECT":
        print("REJECTED: AI Agent rejected this deployment!", file=sys.stderr)
        sys.exit(3)

    print("OK: AI Agent complete -- pipeline may proceed.")


if __name__ == "__main__":
    main()
```

```text
# ai-agent/requirements.txt
# No external dependencies needed.
# The agent uses only Python stdlib (urllib, json, pathlib).
```

---

## 7. Lab 5 — Connect Ollama to GitLab CI

### Step 5.1 — Register a Self-Hosted GitLab Runner

The runner must run on the same machine as Ollama (or have network access to it).

```bash
# Install GitLab Runner
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt install gitlab-runner

# Register the runner
# Go to: GitLab project -> Settings -> CI/CD -> Runners -> New project runner
# Copy the registration token, then run:
sudo gitlab-runner register \
  --url "https://gitlab.com" \
  --token "<YOUR_RUNNER_TOKEN>" \
  --executor "shell" \
  --description "ollama-runner" \
  --tag-list "self-hosted"

# Start the runner
sudo systemctl start gitlab-runner
sudo systemctl enable gitlab-runner
sudo systemctl status gitlab-runner
```

Using the `shell` executor means the runner jobs run directly on the host where Ollama is installed, so `localhost:11434` works without any extra network configuration.

---

### Step 5.2 — Add CI/CD Variables in GitLab UI

Navigate to: **Project -> Settings -> CI/CD -> Variables -> Add variable**

| Key                  | Value                        | Type        | Masked | Notes                              |
|----------------------|------------------------------|-------------|--------|------------------------------------|
| `OLLAMA_HOST`        | `http://localhost:11434`     | Variable    | No     | Change if Ollama is on another host |
| `DOCKER_HUB_USER`    | your Docker Hub username     | Variable    | No     |                                    |
| `DOCKER_HUB_PASSWORD`| your Docker Hub password     | Variable    | Yes    | Always mask this                   |
| `KUBECONFIG_CONTENT` | contents of ~/.kube/config   | File        | No     | Set type to File                   |

---

### Step 5.3 — Give the Runner Python Access

```bash
# On the runner host
which python3
python3 --version    # must be 3.8+

# If not installed:
sudo apt install -y python3 python3-pip

# Verify the gitlab-runner user can run python3
sudo -u gitlab-runner python3 --version
```

---

### Step 5.4 — Place Agent Script in Your Repo

```bash
mkdir -p ai-agent
# Copy ollama_agent.py into ai-agent/
git add ai-agent/ollama_agent.py
git commit -m "Add Ollama AI agent for GitLab CI pipeline"
git push
```

---

### Step 5.5 — Test the Agent Manually Before the Pipeline

```bash
# On the runner host, in your repo directory:
export OLLAMA_HOST="http://localhost:11434"
export OLLAMA_MODEL="llama3.2:1b"

python3 ai-agent/ollama_agent.py \
  --path ./app \
  --environment dev \
  --output ai_report.json

cat ai_report.json | python3 -m json.tool
```

---

## 8. Lab 6 — Full .gitlab-ci.yml with Ollama AI Stages

Create `.gitlab-ci.yml` in your repository root:

```yaml
# ================================================================
#  .gitlab-ci.yml -- CI/CD with Local Ollama AI Agent
#  Stack  : GitLab CI + Docker + Kind (Kubernetes)
#  AI     : Ollama (100% local, FREE, no API key needed)
#  Stages : test -> docker -> ai-analysis -> deploy -> smoke
# ================================================================

default:
  image: python:3.11-slim
  tags:
    - self-hosted          # runner that has access to Ollama
  interruptible: true
  timeout: 60 minutes

stages:
  - test
  - docker
  - ai-analysis
  - deploy
  - smoke

variables:
  # Change to match your setup
  DOCKER_IMAGE: "your-dockerhub-user/workshop-app"
  DOCKER_TAG: "${CI_PIPELINE_IID}-${CI_COMMIT_SHORT_SHA}"

  # Ollama -- set OLLAMA_HOST as a CI/CD variable in GitLab UI
  OLLAMA_MODEL: "llama3.2:1b"

  # Deploy target -- override with a pipeline trigger variable
  DEPLOY_ENV: "dev"
  K8S_NAMESPACE: "workshop-${DEPLOY_ENV}"

  # Gate flags
  AI_BLOCK_ON_REJECT: "true"    # set to "false" for advisory-only mode
  SKIP_AI: "false"              # set to "true" for emergency deploys
  SKIP_TESTS: "false"           # set to "true" for emergency deploys

  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

# ----------------------------------------------------------------
# STAGE 1 -- Unit Tests
# ----------------------------------------------------------------
unit-tests:
  stage: test
  rules:
    - if: $SKIP_TESTS == "true"
      when: never
    - when: always
  script:
    - echo "Branch  - $CI_COMMIT_BRANCH"
    - echo "Commit  - $CI_COMMIT_SHORT_SHA"
    - echo "Author  - $CI_COMMIT_AUTHOR"
    - python3 -m venv venv
    - source venv/bin/activate
    - pip install -r app/requirements.txt -q
    - mkdir -p test-results
    - pytest app/tests/ -v
        --junitxml=test-results/results.xml
        --tb=short
    - echo "Tests complete"
  artifacts:
    when: always
    reports:
      junit: test-results/results.xml
    expire_in: 7 days

# ----------------------------------------------------------------
# STAGE 2 -- Docker Build and Push
# ----------------------------------------------------------------
docker-build:
  stage: docker
  image: docker:24
  services:
    - docker:24-dind
  needs:
    - job: unit-tests
      optional: true
  script:
    - echo "Building - ${DOCKER_IMAGE}:${DOCKER_TAG}"
    - docker build
        --build-arg BUILD_NUMBER=${CI_PIPELINE_IID}
        --label "git.commit=${CI_COMMIT_SHA}"
        --label "gitlab.pipeline=${CI_PIPELINE_IID}"
        -t ${DOCKER_IMAGE}:${DOCKER_TAG}
        -t ${DOCKER_IMAGE}:${DEPLOY_ENV}-latest
        .
    - echo "${DOCKER_HUB_PASSWORD}" | docker login
        -u "${DOCKER_HUB_USER}" --password-stdin
    - docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
    - docker push ${DOCKER_IMAGE}:${DEPLOY_ENV}-latest
    - docker logout
    - echo "Image pushed - ${DOCKER_IMAGE}:${DOCKER_TAG}"

# ----------------------------------------------------------------
# STAGE 3 -- Ollama AI Analysis
# Runs all 3 AI checks via the Python agent and gates on results.
# ----------------------------------------------------------------
ai-analysis:
  stage: ai-analysis
  needs:
    - docker-build
  rules:
    - if: $SKIP_AI == "true"
      when: never
    - when: always
  script:
    - echo "Running Ollama AI Agent"
    - echo "Host  - ${OLLAMA_HOST}"
    - echo "Model - ${OLLAMA_MODEL}"

    # Run the full AI agent (code review + security + deploy decision)
    # Exit codes: 0=ok, 2=security blocked, 3=AI rejected
    - |
      python3 ai-agent/ollama_agent.py \
        --path app/ \
        --environment "${DEPLOY_ENV}" \
        --output ai_report.json \
        --model "${OLLAMA_MODEL}"
      AI_EXIT=$?

      # Parse the report and apply gate logic
      python3 - <<'EOF'
      import json, sys, os

      with open("ai_report.json") as f:
          r = json.load(f)

      score    = r.get("score", r.get("code_review", {}).get("score", 5))
      grade    = r.get("code_review", {}).get("grade", "N/A")
      risk     = r.get("risk_level", r.get("security_scan", {}).get("risk_level", "UNKNOWN"))
      blocked  = r.get("security_scan", {}).get("block_deployment", False)
      secrets  = r.get("security_scan", {}).get("secrets_found", False)
      decision = r.get("decision", r.get("deploy_decision", {}).get("decision", "APPROVE"))
      conf     = r.get("deploy_decision", {}).get("confidence", "N/A")
      reason   = r.get("deploy_decision", {}).get("reason", "N/A")

      print(f"""
--- CODE REVIEW ---
Score   : {score}/10  (Grade: {grade})
Summary : {r.get("summary", "N/A")}

--- SECURITY SCAN ---
Risk Level : {risk}
Blocked    : {blocked}
Secrets    : {secrets}

--- DEPLOY DECISION ---
Decision   : {decision}
Confidence : {conf}%
Reason     : {reason[:80]}
""")

      deploy_env = os.environ.get("DEPLOY_ENV", "dev")
      block      = os.environ.get("AI_BLOCK_ON_REJECT", "true").lower() == "true"

      if block:
          if risk == "CRITICAL":
              print("BLOCKED: CRITICAL security risk. Deployment blocked by AI.", file=sys.stderr)
              sys.exit(1)
          if secrets:
              print("BLOCKED: Secrets found in code. Deployment blocked.", file=sys.stderr)
              sys.exit(1)
          if deploy_env == "prod":
              if risk == "HIGH":
                  print("BLOCKED: HIGH security risk -- production deploy blocked.", file=sys.stderr)
                  sys.exit(1)
              if score < 5:
                  print(f"BLOCKED: Code quality score {score}/10 too low for production.", file=sys.stderr)
                  sys.exit(1)
              if decision == "REJECT":
                  print(f"BLOCKED: AI rejected production deployment: {reason}", file=sys.stderr)
                  sys.exit(1)
          if decision == "REJECT":
              print(f"WARNING: AI recommends not deploying to {deploy_env} (non-blocking)")
          if score < 6:
              print(f"WARNING: Code quality score {score}/10 -- consider improving before prod")
      else:
          print("INFO: AI_BLOCK_ON_REJECT=false -- AI result is advisory only")

      print("OK: AI gate passed.")
      EOF

  artifacts:
    when: always
    paths:
      - ai_report.json
    expire_in: 30 days

# ----------------------------------------------------------------
# STAGE 4 -- Deploy to Kubernetes
# ----------------------------------------------------------------
deploy-k8s:
  stage: deploy
  image: bitnami/kubectl:latest
  needs:
    - ai-analysis
  script:
    - echo "Deploying to - ${K8S_NAMESPACE}"
    - export KUBECONFIG="${KUBECONFIG_CONTENT}"

    - kubectl get namespace ${K8S_NAMESPACE} 2>/dev/null
        || kubectl create namespace ${K8S_NAMESPACE}

    - kubectl apply -f k8s/ -n ${K8S_NAMESPACE}

    - kubectl set image deployment/workshop-app
        workshop-app=${DOCKER_IMAGE}:${DOCKER_TAG}
        -n ${K8S_NAMESPACE}

    - kubectl annotate deployment/workshop-app
        gitlab.pipeline="${CI_PIPELINE_IID}"
        git.commit="${CI_COMMIT_SHA}"
        deployed.at="$(date -u +%Y-%m-%dT%H:%M:%SZ)"
        -n ${K8S_NAMESPACE} --overwrite

    - kubectl rollout status deployment/workshop-app
        -n ${K8S_NAMESPACE}
        --timeout=5m

    - kubectl get pods -n ${K8S_NAMESPACE}
        -l app=workshop-app
        -o wide

  after_script:
    - |
      if [ "$CI_JOB_STATUS" = "failed" ]; then
        echo "Deploy failed -- rolling back..."
        export KUBECONFIG="${KUBECONFIG_CONTENT}"
        kubectl rollout undo deployment/workshop-app \
          -n ${K8S_NAMESPACE} 2>/dev/null \
          && echo "Rollback complete" \
          || echo "Rollback skipped (deploy may not have started)"
      fi

# ----------------------------------------------------------------
# STAGE 5 -- Smoke Tests
# ----------------------------------------------------------------
smoke-tests:
  stage: smoke
  image: curlimages/curl:latest
  needs:
    - deploy-k8s
  script:
    - echo "Running smoke tests against ${K8S_NAMESPACE}..."

    # Uses in-cluster DNS -- runner must be inside the cluster,
    # or replace with a NodePort/Ingress URL.
    - SVC_URL="http://workshop-app-svc.${K8S_NAMESPACE}.svc.cluster.local"

    - FAIL=0
    - curl -sf ${SVC_URL}/health
        && echo "PASS /health"
        || { echo "FAIL /health"; FAIL=1; }
    - curl -sf ${SVC_URL}/ready
        && echo "PASS /ready"
        || { echo "FAIL /ready"; FAIL=1; }
    - curl -sf ${SVC_URL}/
        && echo "PASS /"
        || { echo "FAIL /"; FAIL=1; }

    - |
      if [ $FAIL -ne 0 ]; then
        echo "Smoke tests failed!"
        exit 1
      fi
      echo "All smoke tests passed."
```

---

## 9. Lab 7 — Deploy to Kind Kubernetes

### Step 7.1 — Kubernetes Manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workshop-app
  labels:
    app: workshop-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: workshop-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: workshop-app
    spec:
      containers:
        - name: workshop-app
          image: your-dockerhub-user/workshop-app:latest
          ports:
            - containerPort: 5000
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
            initialDelaySeconds: 10
            periodSeconds: 15
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
spec:
  selector:
    app: workshop-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 5000
  type: ClusterIP
```

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  environment: "development"
  log_level: "INFO"
```

### Step 7.2 — Verify Kind Cluster

```bash
kind get clusters
kubectl cluster-info
kubectl get nodes

kubectl create namespace workshop-dev     2>/dev/null || true
kubectl create namespace workshop-staging 2>/dev/null || true
kubectl create namespace workshop-prod    2>/dev/null || true
```

---

## 10. Lab 8 — Demo Run and Verification

### Step 8.1 — Pre-Demo Checklist

```bash
# 1. Ollama running?
curl http://localhost:11434/
# Expected: Ollama is running

# 2. Model pulled?
ollama list
# Expected: llama3.2:1b listed

# 3. GitLab Runner running?
sudo gitlab-runner status

# 4. Kind cluster up?
kubectl get nodes
# Expected: control-plane + worker nodes Ready

# 5. Docker running?
docker ps

# 6. Test AI agent manually
cd /path/to/your/repo
python3 ai-agent/ollama_agent.py \
  --path ./app \
  --environment dev \
  --output test_report.json

cat test_report.json | python3 -m json.tool
```

### Step 8.2 — Trigger the Pipeline

```bash
# Push any change to trigger the pipeline
git commit --allow-empty -m "trigger: test pipeline"
git push

# Or trigger via GitLab UI:
# Project -> CI/CD -> Pipelines -> Run pipeline
```

To override variables for a single run, go to **CI/CD -> Pipelines -> Run pipeline** and add key-value pairs such as `DEPLOY_ENV=prod` or `SKIP_AI=true`.

### Step 8.3 — Watch the Pipeline

1. Open your GitLab project
2. Go to **CI/CD -> Pipelines**
3. Click the running pipeline
4. Click any job to see live logs
5. The `ai-analysis` job will show the full AI output including scores, risk levels, and the deploy decision

### Step 8.4 — Verify Deployment

```bash
# Watch pods come up
kubectl get pods -n workshop-dev -w

# Check deployment status
kubectl describe deployment workshop-app -n workshop-dev

# Test the running app via port-forward
kubectl port-forward svc/workshop-app-svc 8888:80 -n workshop-dev &
curl http://localhost:8888/health
curl http://localhost:8888/

# View the AI report artifact
# GitLab -> CI/CD -> Pipelines -> pick run -> ai-analysis job -> Browse artifacts
```

### Step 8.5 — Demo: Force AI Rejection

To demonstrate the AI blocking a bad deployment:

```bash
cat >> app/app.py << 'EOF'

# BAD PRACTICE -- for demo purposes
DB_PASSWORD = "admin123"
SECRET_KEY  = "hardcoded-secret-do-not-do-this"
EOF

git add app/app.py
git commit -m "demo: introduce security issues"
git push
```

Then run the pipeline with `DEPLOY_ENV=prod`. The AI should detect the hardcoded secrets and block the production deployment.

---

## 11. Troubleshooting

### "Cannot reach Ollama" in the runner job

```bash
# Check if Ollama is running
sudo systemctl status ollama
sudo systemctl start ollama

# If runner is in Docker and Ollama is on the host, use the host IP
ip route | grep docker | awk '{print $9}'
# Set OLLAMA_HOST to that IP, e.g.: http://172.17.0.1:11434

# Allow Ollama on all interfaces
sudo OLLAMA_HOST=0.0.0.0 ollama serve
```

### "Model not found" error

```bash
ollama pull llama3.2:1b
ollama list

# Check disk space (models need 2-4 GB)
df -h
```

### AI response is not valid JSON

The model sometimes adds text around JSON. The agent handles this with `safe_parse_json()`. If you still see issues, switch to a more instruction-following model:

```bash
ollama pull mistral:7b
# Then set OLLAMA_MODEL=mistral:7b in your CI/CD variables
```

### Ollama is slow (60+ seconds per call)

```bash
# Check if GPU is being used
ollama ps

# Use a smaller model for faster demos
ollama pull llama3.2:1b   # ~1 GB, faster on CPU

# Or increase timeout in ollama_agent.py:
# Change timeout=120 to timeout=300
```

### kubectl rollout timeout

```bash
kubectl describe pods -n workshop-dev
kubectl get events -n workshop-dev --sort-by='.lastTimestamp'

# Common cause: image pull failed -- check Docker Hub credentials in CI/CD variables
kubectl get pods -n workshop-dev
kubectl logs <pod-name> -n workshop-dev
```

### Runner cannot find python3

```bash
# On the runner host
sudo apt install -y python3 python3-pip

# Verify the gitlab-runner user has access
sudo -u gitlab-runner python3 --version
```

### Pipeline not triggering

```bash
# Check runner is online and tagged correctly
# GitLab -> Project -> Settings -> CI/CD -> Runners

# Confirm .gitlab-ci.yml has the matching tag
# default:
#   tags:
#     - self-hosted    <- must match your runner's tag
```

---

## Quick Reference

```
OLLAMA COMMANDS
-------------------------------------------------
ollama serve              Start Ollama server
ollama pull llama3.2:1b   Download model
ollama list               Show downloaded models
ollama run llama3.2:1b    Interactive chat
ollama ps                 Show running models + GPU/CPU usage
ollama rm llama3.2:1b     Delete a model

MODEL RECOMMENDATIONS
-------------------------------------------------
llama3.2:1b   1.3 GB   4 GB+ RAM   Fast demo, low RAM
llama3.2:1b   2.0 GB   8 GB+ RAM   Workshop default
mistral:7b    4.1 GB   8 GB+ RAM   Better JSON quality
codellama:7b  3.8 GB   8 GB+ RAM   Best for code review

AI AGENT EXIT CODES
-------------------------------------------------
0   Success -- pipeline proceeds
2   Security scan blocked deployment
3   AI deploy decision = REJECT

GITLAB CI/CD VARIABLES REQUIRED
-------------------------------------------------
OLLAMA_HOST          Variable   http://localhost:11434
DOCKER_HUB_USER      Variable   Docker Hub username
DOCKER_HUB_PASSWORD  Variable   Docker Hub password (masked)
KUBECONFIG_CONTENT   File       Contents of ~/.kube/config

JENKINS TO GITLAB CI MAPPING
-------------------------------------------------
Jenkinsfile              .gitlab-ci.yml
parameters { choice }    CI/CD Variables + Run pipeline overrides
withCredentials          CI/CD Variables (Settings -> CI/CD -> Variables)
agent any                default.image + tags: [self-hosted]
post { failure }         after_script checking $CI_JOB_STATUS
archiveArtifacts         artifacts.paths
junit publisher          artifacts.reports.junit
when { expression }      rules: - if: $VAR == "true" when: never
```

---

*Workshop Guide -- GitLab CI + Ollama Local AI Agent*
*100% Free | No API Keys | Runs Fully Offline*
*Estimated Duration: 3-4 hours*
