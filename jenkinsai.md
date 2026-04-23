# 🦙 Jenkins CI/CD + Ollama Local AI Agent
## Step-by-Step Workshop Guide — 100% Free, No API Keys, Runs Locally

---

## 📋 Table of Contents

1. [Workshop Overview](#1-workshop-overview)
2. [Prerequisites](#2-prerequisites)
3. [Lab 1 — Install & Run Ollama](#3-lab-1--install--run-ollama)
4. [Lab 2 — Pull AI Models](#4-lab-2--pull-ai-models)
5. [Lab 3 — Test Ollama API](#5-lab-3--test-ollama-api)
6. [Lab 4 — Build the AI Agent Script](#6-lab-4--build-the-ai-agent-script)
7. [Lab 5 — Connect Ollama to Jenkins](#7-lab-5--connect-ollama-to-jenkins)
8. [Lab 6 — Full Jenkinsfile with Ollama AI Stages](#8-lab-6--full-jenkinsfile-with-ollama-ai-stages)
9. [Lab 7 — Deploy to Kind Kubernetes](#9-lab-7--deploy-to-kind-kubernetes)
10. [Lab 8 — Demo Run & Verification](#10-lab-8--demo-run--verification)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Workshop Overview

### 🎯 What You Will Build

A complete CI/CD pipeline where **Ollama runs AI models locally on your machine** — no internet, no API keys, no costs — and Jenkins calls it automatically on every git push to:

- ✅ Review code quality
- ✅ Scan for security vulnerabilities
- ✅ Make deployment approval decisions

### 🏗️ Architecture

```
Developer Git Push
       │
       ▼
┌─────────────────┐     Webhook      ┌──────────────────────┐
│   GitHub Repo   │ ───────────────► │   Jenkins Server     │
└─────────────────┘                  │                      │
                                     │  Stage 1: Checkout   │
                                     │  Stage 2: Tests      │
                                     │  Stage 3: Docker     │
                                     │  Stage 4: AI Review  │──────┐
                                     │  Stage 5: AI SecScan │      │
                                     │  Stage 6: AI Deploy  │      │
                                     │  Stage 7: K8s Deploy │      │
                                     └──────────────────────┘      │
                                                                    │ HTTP
                                                                    │ localhost:11434
                                                          ┌─────────▼──────────┐
                                                          │   Ollama Server    │
                                                          │  ┌──────────────┐  │
                                                          │  │ llama3.2:3b  │  │
                                                          │  │ codellama    │  │
                                                          │  │ mistral      │  │
                                                          │  └──────────────┘  │
                                                          │  Runs 100% LOCAL   │
                                                          └────────────────────┘
                                                                    │
                                                          ┌─────────▼──────────┐
                                                          │  Kind Kubernetes   │
                                                          │  workshop-dev      │
                                                          │  workshop-prod     │
                                                          └────────────────────┘
```

### 💡 Why Ollama?

| Feature | Ollama | Cloud APIs |
|---------|--------|-----------|
| Cost | **FREE** | Pay per token |
| Internet | **Not needed** | Required |
| API Key | **None** | Required |
| Privacy | **100% local** | Data sent to cloud |
| Speed | Fast (GPU/CPU) | Depends on network |
| Models | 100+ available | Limited to provider |

### 📦 Tech Stack

| Component | Technology | Notes |
|-----------|-----------|-------|
| Local AI | **Ollama** | Runs models locally |
| AI Model | **llama3.2:3b** | Fast, good quality, ~2GB |
| CI/CD | **Jenkins** | Already installed |
| Containers | **Docker** | Already installed |
| Kubernetes | **Kind** | Already set up |
| Agent Script | **Python 3** | Calls Ollama REST API |

---

## 2. Prerequisites

### ✅ Already Installed (Your Setup)

```bash
# Verify your existing setup
docker --version         # Docker is running
kubectl version --client  # kubectl available
kind version             # Kind cluster exists
java --version           # Java 17+ for Jenkins
jenkins --version        # Jenkins running
python3 --version        # Python 3.8+
```

### 🖥️ Minimum Hardware for Ollama

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| RAM | 8 GB | 16 GB |
| Disk | 10 GB free | 20 GB free |
| CPU | 4 cores | 8 cores |
| GPU | Not required | Speeds up 10x |

### 📁 Final Project Structure

```
your-repo/
├── Jenkinsfile              ← Main pipeline (Lab 6)
├── Dockerfile               ← Your existing Dockerfile
├── app/
│   ├── app.py
│   ├── requirements.txt
│   └── tests/
│       └── test_app.py
├── ai-agent/
│   ├── ollama_agent.py      ← AI Agent script (Lab 4)
│   └── requirements.txt
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

---

## 3. Lab 1 — Install & Run Ollama

### Step 1.1 — Install Ollama

**Linux (Ubuntu/Debian) — Recommended for Jenkins server:**

```bash
# One-line install
curl -fsSL https://ollama.com/install.sh | sh

# Verify installation
ollama --version
```

**macOS:**

```bash
# Via Homebrew
brew install ollama

# Or download the app from https://ollama.com
```

**Windows:**

```
Download installer from: https://ollama.com/download/windows
Run the installer → Ollama runs as a system service
```

---

### Step 1.2 — Start Ollama Server

```bash
# Start Ollama (runs on port 11434 by default)
ollama serve

# You should see:
# time=... level=INFO msg="Listening on 127.0.0.1:11434 (version x.x.x)"
```

> 💡 **Keep this terminal open.** Ollama must be running when Jenkins calls it.

**To run Ollama as a background service on Linux:**

```bash
# Check if already running as systemd service (after install.sh)
sudo systemctl status ollama

# If not running, start it
sudo systemctl start ollama
sudo systemctl enable ollama     # auto-start on boot

# Check logs
sudo journalctl -u ollama -f
```

---

### Step 1.3 — Make Ollama Accessible to Jenkins

By default Ollama only listens on `127.0.0.1`. If Jenkins runs on the same machine, you are fine. If Jenkins is on a different server:

```bash
# Allow Ollama to listen on all interfaces
# Edit the systemd service
sudo systemctl edit ollama

# Add these lines in the editor:
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"

# Save and restart
sudo systemctl daemon-reload
sudo systemctl restart ollama

# Verify it listens on 0.0.0.0
ss -tlnp | grep 11434
```

> ⚠️ **Security note:** Only do this in your lab/demo network. In production, use a firewall rule to restrict access.

---

### Step 1.4 — Verify Ollama is Running

```bash
# Basic health check
curl http://localhost:11434/

# Expected response:
# Ollama is running
```

---

## 4. Lab 2 — Pull AI Models

### Step 2.1 — Pull the Recommended Model

For this workshop we use **llama3.2:3b** — it is small (2 GB), fast on CPU, and good enough for code review tasks.

```bash
# Pull the model (downloads ~2 GB — do this before the demo!)
ollama pull llama3.2:3b

# Watch the download progress:
# pulling manifest
# pulling 6a0746a1ec1a... 100% ████████ 2.0 GB
# verifying sha256 digest
# writing manifest
# success
```

### Step 2.2 — Optional: Pull Other Models

```bash
# Better code understanding (4 GB, needs 8GB+ RAM)
ollama pull codellama:7b

# Fast and lightweight (1.5 GB)
ollama pull llama3.2:1b

# Balanced quality (4.1 GB, needs 8GB+ RAM)
ollama pull mistral:7b

# See all available models
ollama list

# See all models on Ollama hub
# https://ollama.com/library
```

### Step 2.3 — Test the Model Works

```bash
# Quick interactive test
ollama run llama3.2:3b "Say hello in one sentence"

# Expected: Hello! How can I assist you today?

# Exit with: /bye
```

---

## 5. Lab 3 — Test Ollama API

Ollama exposes a simple REST API. The AI agent script uses this directly.

### Step 3.1 — Test with curl

```bash
# Basic API call — this is what our Python script will do
curl http://localhost:11434/api/generate \
  -d '{
    "model": "llama3.2:3b",
    "prompt": "Review this Python code quality in one sentence: x=1+1",
    "stream": false
  }'

# Expected JSON response:
# {"model":"llama3.2:3b","response":"The code is simple...","done":true,...}
```

### Step 3.2 — Test Chat Format (used in our agent)

```bash
curl http://localhost:11434/api/chat \
  -d '{
    "model": "llama3.2:3b",
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
  -d '{"name": "llama3.2:3b"}'

# Check running models
curl http://localhost:11434/api/ps

# Delete a model
curl -X DELETE http://localhost:11434/api/delete \
  -d '{"name": "llama3.2:3b"}'
```

---

## 6. Lab 4 — Build the AI Agent Script

Create the file `ai-agent/ollama_agent.py` in your repository:

```python
#!/usr/bin/env python3
"""
Ollama AI Agent for Jenkins CI/CD Pipeline
==========================================
100% FREE — Runs locally — No API keys needed
Calls Ollama REST API for:
  1. Code Review & Quality Score
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


# ── Configuration ──────────────────────────────────────────────────────────────
OLLAMA_HOST  = os.environ.get("OLLAMA_HOST", "http://localhost:11434")
OLLAMA_MODEL = os.environ.get("OLLAMA_MODEL", "llama3.2:3b")


# ── Helpers ────────────────────────────────────────────────────────────────────

def check_ollama_running():
    """Verify Ollama server is reachable before doing anything."""
    try:
        req = urllib.request.urlopen(OLLAMA_HOST, timeout=5)
        return True
    except Exception:
        print(f"❌ Cannot reach Ollama at {OLLAMA_HOST}", file=sys.stderr)
        print("   → Make sure Ollama is running: ollama serve", file=sys.stderr)
        print("   → Or check OLLAMA_HOST environment variable", file=sys.stderr)
        sys.exit(1)


def check_model_available(model: str):
    """Check that the required model is pulled."""
    try:
        url  = f"{OLLAMA_HOST}/api/tags"
        req  = urllib.request.urlopen(url, timeout=10)
        data = json.loads(req.read())
        models = [m["name"] for m in data.get("models", [])]
        # Match partial names e.g. "llama3.2:3b" matches "llama3.2:3b"
        available = any(model in m or m in model for m in models)
        if not available:
            print(f"❌ Model '{model}' not found. Pull it first:", file=sys.stderr)
            print(f"   ollama pull {model}", file=sys.stderr)
            sys.exit(1)
        print(f"✅ Model '{model}' is available")
    except Exception as e:
        print(f"⚠️  Could not verify model (continuing): {e}", file=sys.stderr)


def call_ollama(system_prompt: str, user_prompt: str, timeout: int = 120) -> str:
    """
    Call Ollama /api/chat endpoint.
    Returns the model's response text.
    Uses only stdlib — no pip install required for the HTTP call itself.
    """
    payload = json.dumps({
        "model":  OLLAMA_MODEL,
        "stream": False,
        "options": {
            "temperature": 0.1,    # low temp = more deterministic JSON
            "num_predict": 1024,   # max output tokens
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
            print(f"   ⏱  Ollama responded in {elapsed:.1f}s")
            return data["message"]["content"].strip()
    except urllib.error.URLError as e:
        print(f"❌ Ollama request failed: {e}", file=sys.stderr)
        sys.exit(1)


def safe_parse_json(text: str, fallback: dict) -> dict:
    """
    Strip markdown fences and parse JSON.
    If parsing fails, try to extract first {...} block.
    Returns fallback dict on complete failure.
    """
    # Remove ```json ... ``` fences
    text = text.strip()
    if "```" in text:
        parts = text.split("```")
        for part in parts:
            part = part.strip().lstrip("json").strip()
            if part.startswith("{"):
                text = part
                break

    # Try direct parse
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        pass

    # Try to find first { ... } block
    start = text.find("{")
    end   = text.rfind("}")
    if start != -1 and end != -1:
        try:
            return json.loads(text[start:end+1])
        except json.JSONDecodeError:
            pass

    print(f"⚠️  Could not parse JSON, using fallback. Raw response:\n{text[:300]}")
    return fallback


def read_source_files(path: str, max_chars: int = 5000) -> str:
    """Read source files from path, truncated to max_chars to fit model context."""
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


# ── Stage 1: Code Review ───────────────────────────────────────────────────────

def run_code_review(code_path: str) -> dict:
    print("\n" + "─"*50)
    print("📋 STAGE 1: AI Code Review")
    print("─"*50)

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
- Be specific in issues, not generic

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

    score = result.get("score", 5)
    grade = result.get("grade", "?")
    print(f"\n  🎯 Score   : {score}/10  (Grade: {grade})")
    print(f"  📝 Summary : {result.get('summary', 'N/A')}")

    for issue in result.get("critical_issues", []):
        print(f"  🔴 CRITICAL: {issue}")
    for issue in result.get("major_issues", []):
        print(f"  🟡 MAJOR   : {issue}")
    for tip in result.get("suggestions", []):
        print(f"  💡 TIP     : {tip}")

    return result


# ── Stage 2: Security Scan ─────────────────────────────────────────────────────

def run_security_scan(code_path: str) -> dict:
    print("\n" + "─"*50)
    print("🔒 STAGE 2: AI Security Scan")
    print("─"*50)

    code = read_source_files(code_path)

    system = """You are an application security expert doing a security audit.
You MUST respond with ONLY valid JSON. No explanation. No markdown. No extra text."""

    user = f"""Scan for security vulnerabilities and return ONLY this JSON:
{{
  "risk_level": "<LOW|MEDIUM|HIGH|CRITICAL>",
  "score": <integer 1-10 where 10=most secure>,
  "vulnerabilities": [
    {{
      "type": "<vulnerability type e.g. SQL Injection>",
      "severity": "<LOW|MEDIUM|HIGH|CRITICAL>",
      "file": "<filename>",
      "line": "<line number or 'unknown'>",
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
- If code looks clean, still list risk_level as LOW

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

    risk  = result.get("risk_level", "UNKNOWN")
    score = result.get("score", "?")
    risk_icon = {"LOW": "🟢", "MEDIUM": "🟡", "HIGH": "🟠", "CRITICAL": "🔴"}.get(risk, "⚪")

    print(f"\n  {risk_icon} Risk Level : {risk}")
    print(f"  🔐 Sec Score: {score}/10")
    print(f"  📝 Summary  : {result.get('summary', 'N/A')}")
    print(f"  🔑 Secrets  : {'YES ⚠️' if result.get('secrets_found') else 'None found ✅'}")

    for v in result.get("vulnerabilities", []):
        sev_icon = {"CRITICAL": "🔴", "HIGH": "🟠", "MEDIUM": "🟡", "LOW": "🟢"}.get(
            v.get("severity", ""), "⚪")
        print(f"  {sev_icon} {v.get('severity','?')} [{v.get('type','?')}]"
              f" in {v.get('file','?')}: {v.get('description','')[:70]}")
        print(f"     Fix: {v.get('recommendation','')[:70]}")

    return result


# ── Stage 3: Deploy Decision ───────────────────────────────────────────────────

def run_deploy_decision(code_review: dict, security_scan: dict, environment: str) -> dict:
    print("\n" + "─"*50)
    print(f"🚦 STAGE 3: AI Deploy Decision  [{environment.upper()}]")
    print("─"*50)

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
        "reason": "AI decision inconclusive — manual review recommended",
        "conditions": [],
        "recommended_actions": ["Review manually before proceeding"],
        "estimated_rollback_needed": False
    })

    decision   = result.get("decision", "APPROVE")
    confidence = result.get("confidence", "?")
    dec_icon   = {"APPROVE": "✅", "REJECT": "❌",
                  "APPROVE_WITH_CONDITIONS": "⚠️"}.get(decision, "❓")

    print(f"\n  {dec_icon} Decision   : {decision}")
    print(f"  📊 Confidence: {confidence}%")
    print(f"  ⚠️  Risk      : {result.get('risk_assessment', 'N/A')}")
    print(f"  📝 Reason    : {result.get('reason', 'N/A')}")

    for cond in result.get("conditions", []):
        print(f"  ⚙️  Condition : {cond}")
    for action in result.get("recommended_actions", []):
        print(f"  🔧 Action   : {action}")

    return result


# ── Main ───────────────────────────────────────────────────────────────────────

def main():
    parser = argparse.ArgumentParser(
        description="Ollama AI Agent for Jenkins CI/CD — 100% Local & Free"
    )
    parser.add_argument("--path",        default="./app",         help="Path to source code")
    parser.add_argument("--environment", default="dev",           help="Target environment")
    parser.add_argument("--output",      default="ai_report.json",help="Output JSON report file")
    parser.add_argument("--model",       default="",              help="Override Ollama model")
    args = parser.parse_args()

    # Allow CLI override of model
    global OLLAMA_MODEL
    if args.model:
        OLLAMA_MODEL = args.model

    print("\n╔══════════════════════════════════════════════════╗")
    print("║   🦙 Ollama AI Agent — Jenkins CI/CD            ║")
    print(f"║   Host  : {OLLAMA_HOST:<38}║")
    print(f"║   Model : {OLLAMA_MODEL:<38}║")
    print(f"║   Path  : {args.path:<38}║")
    print(f"║   Target: {args.environment:<38}║")
    print("╚══════════════════════════════════════════════════╝")

    # Pre-flight checks
    check_ollama_running()
    check_model_available(OLLAMA_MODEL)

    # Run all three AI stages
    code_review   = run_code_review(args.path)
    security_scan = run_security_scan(args.path)
    deploy_dec    = run_deploy_decision(code_review, security_scan, args.environment)

    # Compile full report
    report = {
        "ollama_host":     OLLAMA_HOST,
        "ollama_model":    OLLAMA_MODEL,
        "environment":     args.environment,
        "code_review":     code_review,
        "security_scan":   security_scan,
        "deploy_decision": deploy_dec,
        # Top-level shortcuts for Jenkinsfile readJSON
        "score":           code_review.get("score", 5),
        "risk_level":      security_scan.get("risk_level", "UNKNOWN"),
        "decision":        deploy_dec.get("decision", "APPROVE"),
        "summary":         code_review.get("summary", ""),
    }

    with open(args.output, "w") as fh:
        json.dump(report, fh, indent=2)

    print("\n" + "═"*50)
    print(f"  📄 Report  → {args.output}")
    print(f"  🎯 Score   → {report['score']}/10")
    print(f"  🔒 Risk    → {report['risk_level']}")
    print(f"  🚦 Decision→ {report['decision']}")
    print("═"*50 + "\n")

    # Exit codes used by Jenkinsfile pipeline
    if security_scan.get("block_deployment", False):
        print("🚨 Security scan is blocking deployment!", file=sys.stderr)
        sys.exit(2)

    if deploy_dec.get("decision") == "REJECT":
        print("🚫 AI Agent rejected this deployment!", file=sys.stderr)
        sys.exit(3)

    print("✅ AI Agent complete — pipeline may proceed.")


if __name__ == "__main__":
    main()
```

```text
# ai-agent/requirements.txt
# No external dependencies needed!
# The agent uses only Python stdlib (urllib, json, pathlib)
# stdlib is always available in Jenkins agents
```

---

## 7. Lab 5 — Connect Ollama to Jenkins

### Step 5.1 — Add Jenkins Credentials

Navigate to: **Manage Jenkins → Credentials → System → Global credentials → Add Credentials**

Add this one credential:

| Field | Value |
|-------|-------|
| Kind | Secret text |
| ID | `ollama-host` |
| Secret | `http://localhost:11434` |
| Description | Ollama local server URL |

> 💡 If Jenkins and Ollama are on **different machines**, replace `localhost` with the Ollama server IP.

---

### Step 5.2 — Give Jenkins Python Access

```bash
# On Jenkins server — ensure python3 is available
which python3
python3 --version    # must be 3.8+

# If not installed:
sudo apt install -y python3 python3-pip

# Verify Jenkins user can run python3
sudo -u jenkins python3 --version
```

---

### Step 5.3 — Place Agent Script in Your Repo

```bash
# In your git repository:
mkdir -p ai-agent
# Copy ollama_agent.py into ai-agent/
# Add and commit:
git add ai-agent/ollama_agent.py
git commit -m "Add Ollama AI agent for Jenkins pipeline"
git push
```

---

### Step 5.4 — Test the Agent Manually (Before Pipeline)

```bash
# On Jenkins server, in your repo directory:
export OLLAMA_HOST="http://localhost:11434"
export OLLAMA_MODEL="llama3.2:3b"

python3 ai-agent/ollama_agent.py \
  --path ./app \
  --environment dev \
  --output ai_report.json

# Check the output:
cat ai_report.json | python3 -m json.tool
```

---

## 8. Lab 6 — Full Jenkinsfile with Ollama AI Stages

Create `Jenkinsfile` in your repository root:

```groovy
// ═══════════════════════════════════════════════════════════════════
//  Jenkinsfile — CI/CD with Local Ollama AI Agent
//  Stack  : Jenkins + Docker + Kind (Kubernetes)
//  AI     : Ollama (100% local, FREE, no API key needed)
//  Stages : Checkout → Test → Docker → AI Review →
//           AI Security → AI Decision → K8s Deploy → Smoke Test
// ═══════════════════════════════════════════════════════════════════

pipeline {
    agent any

    // ── Parameters ────────────────────────────────────────────────
    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['dev', 'staging', 'prod'],
            description: 'Target Kubernetes namespace: workshop-<env>'
        )
        choice(
            name: 'OLLAMA_MODEL',
            choices: ['llama3.2:3b', 'llama3.2:1b', 'codellama:7b', 'mistral:7b'],
            description: 'Ollama model to use for AI review'
        )
        booleanParam(
            name: 'SKIP_AI',
            defaultValue: false,
            description: 'Skip all AI stages (emergency deploy)'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip unit tests (emergency only)'
        )
        booleanParam(
            name: 'AI_BLOCK_ON_REJECT',
            defaultValue: true,
            description: 'Block pipeline if AI rejects? Uncheck for demo passthrough'
        )
    }

    // ── Environment Variables ──────────────────────────────────────
    environment {
        // 🔧 Change these to match your setup
        DOCKER_IMAGE  = "your-dockerhub-user/workshop-app"
        K8S_NAMESPACE = "workshop-${params.DEPLOY_ENV}"
        DOCKER_TAG    = "${env.BUILD_NUMBER}-${GIT_COMMIT.take(7)}"

        // Ollama settings — pulled from Jenkins credential
        OLLAMA_MODEL  = "${params.OLLAMA_MODEL}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '15'))
        timeout(time: 60, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
        skipDefaultCheckout(false)
    }

    // ══════════════════════════════════════════════════════════════
    stages {

        // ─────────────────────────────────────────────────────────
        stage('🔍 Checkout') {
            steps {
                checkout scm
                sh '''
                    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                    echo " Branch : $(git branch --show-current)"
                    echo " Commit : $(git log -1 --oneline)"
                    echo " Author : $(git log -1 --format='%an <%ae>')"
                    echo " Files  : $(git show --stat HEAD | tail -1)"
                    echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                '''
            }
        }

        // ─────────────────────────────────────────────────────────
        stage('🧪 Unit Tests') {
            when { expression { !params.SKIP_TESTS } }
            steps {
                sh '''
                    echo "Running unit tests..."
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r app/requirements.txt -q
                    mkdir -p test-results
                    pytest app/tests/ -v \
                        --junitxml=test-results/results.xml \
                        --tb=short
                    echo "✅ Tests complete"
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: 'test-results/results.xml'
                }
                failure {
                    echo "❌ Unit tests failed — fix tests before deploying!"
                }
            }
        }

        // ─────────────────────────────────────────────────────────
        stage('🐳 Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                        echo " Building: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

                        docker build \
                            --build-arg BUILD_NUMBER=${BUILD_NUMBER} \
                            --label "git.commit=${GIT_COMMIT}" \
                            --label "jenkins.build=${BUILD_NUMBER}" \
                            -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                            -t ${DOCKER_IMAGE}:${DEPLOY_ENV}-latest \
                            .

                        echo "${DOCKER_PASS}" | docker login \
                            -u "${DOCKER_USER}" --password-stdin

                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:${DEPLOY_ENV}-latest

                        docker logout
                        echo "✅ Image pushed: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────────────────
        // AI STAGES — all run the ollama_agent.py script once,
        // parse the single ai_report.json for each check
        // ─────────────────────────────────────────────────────────

        stage('🦙 Ollama AI Analysis') {
            when { expression { !params.SKIP_AI } }
            steps {
                withCredentials([string(
                    credentialsId: 'ollama-host',
                    variable: 'OLLAMA_HOST'
                )]) {
                    script {
                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                        echo " 🦙 Running Ollama AI Agent"
                        echo " Host  : ${env.OLLAMA_HOST}"
                        echo " Model : ${params.OLLAMA_MODEL}"
                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

                        // Run the full AI agent (all 3 stages in one call)
                        def exitCode = sh(
                            script: """
                                OLLAMA_HOST=${env.OLLAMA_HOST} \
                                OLLAMA_MODEL=${params.OLLAMA_MODEL} \
                                python3 ai-agent/ollama_agent.py \
                                    --path app/ \
                                    --environment ${params.DEPLOY_ENV} \
                                    --output ai_report.json
                            """,
                            returnStatus: true
                        )

                        // Parse the full report
                        def report = readJSON file: 'ai_report.json'

                        // ── Code Review Results ──────────────────
                        def score = report.score ?: report.code_review?.score ?: 5
                        def grade = report.code_review?.grade ?: 'N/A'
                        echo """
╔══════════════════════════════════════════╗
║   📋 CODE REVIEW                         ║
╠══════════════════════════════════════════╣
║  Score   : ${score}/10  (Grade: ${grade})
║  Summary : ${report.summary ?: 'N/A'}
╚══════════════════════════════════════════╝"""

                        // ── Security Scan Results ────────────────
                        def risk    = report.risk_level ?: report.security_scan?.risk_level ?: 'UNKNOWN'
                        def blocked = report.security_scan?.block_deployment ?: false
                        echo """
╔══════════════════════════════════════════╗
║   🔒 SECURITY SCAN                       ║
╠══════════════════════════════════════════╣
║  Risk Level : ${risk}
║  Blocked    : ${blocked}
║  Secrets    : ${report.security_scan?.secrets_found ?: false}
╚══════════════════════════════════════════╝"""

                        // ── Deploy Decision Results ──────────────
                        def decision   = report.decision ?: report.deploy_decision?.decision ?: 'APPROVE'
                        def confidence = report.deploy_decision?.confidence ?: 'N/A'
                        def reason     = report.deploy_decision?.reason ?: 'N/A'
                        echo """
╔══════════════════════════════════════════╗
║   🚦 DEPLOY DECISION                     ║
╠══════════════════════════════════════════╣
║  Decision   : ${decision}
║  Confidence : ${confidence}%
║  Reason     : ${reason?.take(50)}
╚══════════════════════════════════════════╝"""

                        // ── Gate Logic ───────────────────────────
                        if (params.AI_BLOCK_ON_REJECT) {

                            // Hard block: CRITICAL security or secrets
                            if (risk == 'CRITICAL') {
                                error("🚨 CRITICAL security risk! Deployment blocked by AI.")
                            }
                            if (report.security_scan?.secrets_found == true) {
                                error("🚨 Secrets found in code! Deployment blocked.")
                            }

                            // Prod-only blocks
                            if (params.DEPLOY_ENV == 'prod') {
                                if (risk == 'HIGH') {
                                    error("🚨 HIGH security risk — production deploy blocked!")
                                }
                                if (score < 5) {
                                    error("❌ Code quality score ${score}/10 too low for production!")
                                }
                                if (decision == 'REJECT') {
                                    error("🚫 AI Agent rejected production deployment!\n${reason}")
                                }
                            }

                            // Warnings for non-prod
                            if (decision == 'REJECT' && params.DEPLOY_ENV != 'prod') {
                                unstable("⚠️  AI recommends not deploying — proceeding to ${params.DEPLOY_ENV} anyway")
                            }
                            if (score < 6) {
                                unstable("⚠️  Code quality score ${score}/10 — consider fixing before prod")
                            }

                        } else {
                            echo "ℹ️  AI_BLOCK_ON_REJECT=false — AI result is advisory only"
                        }
                    }
                }
            }
        }

        // ─────────────────────────────────────────────────────────
        stage('🚀 Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                        echo " Deploying to: ${K8S_NAMESPACE}"
                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"

                        # Create namespace if it does not exist
                        kubectl get namespace ${K8S_NAMESPACE} 2>/dev/null \
                            || kubectl create namespace ${K8S_NAMESPACE}

                        # Apply all manifests in k8s/
                        kubectl apply -f k8s/ -n ${K8S_NAMESPACE}

                        # Update the running image to the exact tag we built
                        kubectl set image deployment/workshop-app \
                            workshop-app=${DOCKER_IMAGE}:${DOCKER_TAG} \
                            -n ${K8S_NAMESPACE}

                        # Stamp metadata onto the deployment
                        kubectl annotate deployment/workshop-app \
                            jenkins.build="${BUILD_NUMBER}" \
                            git.commit="${GIT_COMMIT}" \
                            deployed.at="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
                            -n ${K8S_NAMESPACE} --overwrite

                        # Block until rollout completes (or 5m timeout)
                        kubectl rollout status deployment/workshop-app \
                            -n ${K8S_NAMESPACE} \
                            --timeout=5m

                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                        echo " Pods after deploy:"
                        kubectl get pods -n ${K8S_NAMESPACE} \
                            -l app=workshop-app \
                            -o wide
                        echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────────────────
        stage('✅ Smoke Tests') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    sh '''
                        echo "Running smoke tests against ${K8S_NAMESPACE}..."

                        # Start port-forward in background
                        kubectl port-forward svc/workshop-app-svc 9977:80 \
                            -n ${K8S_NAMESPACE} &
                        PF_PID=$!

                        # Wait for port-forward to be ready
                        sleep 6

                        FAIL=0

                        # Test 1: Health endpoint
                        curl -sf http://localhost:9977/health \
                            && echo "✅ /health → 200 OK" \
                            || { echo "❌ /health → FAILED"; FAIL=1; }

                        # Test 2: Readiness endpoint
                        curl -sf http://localhost:9977/ready \
                            && echo "✅ /ready  → 200 OK" \
                            || { echo "❌ /ready  → FAILED"; FAIL=1; }

                        # Test 3: Root endpoint
                        curl -sf http://localhost:9977/ \
                            && echo "✅ /       → 200 OK" \
                            || { echo "❌ /       → FAILED"; FAIL=1; }

                        # Cleanup port-forward
                        kill $PF_PID 2>/dev/null || true
                        wait  $PF_PID 2>/dev/null || true

                        if [ $FAIL -ne 0 ]; then
                            echo "❌ Smoke tests failed!"
                            exit 1
                        fi
                        echo "✅ All smoke tests passed!"
                    '''
                }
            }
        }

    }
    // ══════════════════════════════════════════════════════════════

    // ── Post Actions ──────────────────────────────────────────────
    post {
        success {
            echo """
╔══════════════════════════════════════════════════════╗
║  ✅  PIPELINE SUCCEEDED                              ║
╠══════════════════════════════════════════════════════╣
║  Image     : ${DOCKER_IMAGE}:${DOCKER_TAG}
║  Namespace : ${K8S_NAMESPACE}
║  Build     : #${BUILD_NUMBER}
║  AI Model  : ${params.OLLAMA_MODEL}
╚══════════════════════════════════════════════════════╝"""
        }

        failure {
            echo "❌ Pipeline failed — triggering auto-rollback..."
            withCredentials([file(
                credentialsId: 'kubeconfig',
                variable: 'KUBECONFIG'
            )]) {
                sh '''
                    kubectl rollout undo deployment/workshop-app \
                        -n ${K8S_NAMESPACE} 2>/dev/null \
                        && echo "🔄 Rollback complete" \
                        || echo "⚠️  Rollback skipped (deploy may not have started)"
                '''
            }
        }

        always {
            // Archive the AI report as a Jenkins build artifact
            archiveArtifacts artifacts: 'ai_report.json',
                             allowEmptyArchive: true
            cleanWs()
        }
    }
}
```

---

## 9. Lab 7 — Deploy to Kind Kubernetes

Since you already have Kind set up, here are the minimal manifests needed:

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
# Check your Kind cluster
kind get clusters
kubectl cluster-info
kubectl get nodes

# Check existing namespaces
kubectl get namespaces

# Create namespaces manually if not done by pipeline
kubectl create namespace workshop-dev     2>/dev/null || true
kubectl create namespace workshop-staging 2>/dev/null || true
kubectl create namespace workshop-prod    2>/dev/null || true
```

---

## 10. Lab 8 — Demo Run & Verification

### Step 8.1 — Pre-Demo Checklist

```bash
# 1. Ollama running?
curl http://localhost:11434/
# Expected: "Ollama is running"

# 2. Model pulled?
ollama list
# Expected: llama3.2:3b listed

# 3. Jenkins running?
curl http://localhost:8080/
# Expected: Jenkins login page

# 4. Kind cluster up?
kubectl get nodes
# Expected: control-plane + worker nodes Ready

# 5. Docker running?
docker ps
# Expected: docker daemon responding

# 6. Test AI agent manually
cd /path/to/your/repo
python3 ai-agent/ollama_agent.py \
  --path ./app \
  --environment dev \
  --output test_report.json

cat test_report.json | python3 -m json.tool
```

### Step 8.2 — Run the Pipeline

1. Open Jenkins: `http://localhost:8080`
2. Open your pipeline job
3. Click **"Build with Parameters"**
4. Select:
   - `DEPLOY_ENV`: `dev`
   - `OLLAMA_MODEL`: `llama3.2:3b`
   - `AI_BLOCK_ON_REJECT`: ✅ checked
5. Click **Build**
6. Click the build number → **Console Output**
7. Watch the AI stages in real time!

### Step 8.3 — Verify Deployment

```bash
# Watch pods come up
kubectl get pods -n workshop-dev -w

# Check deployment status
kubectl describe deployment workshop-app -n workshop-dev

# Test the running app
kubectl port-forward svc/workshop-app-svc 8888:80 -n workshop-dev &
curl http://localhost:8888/health
curl http://localhost:8888/

# View AI report artifact in Jenkins
# Jenkins → Build → Artifacts → ai_report.json
```

### Step 8.4 — Demo: Force AI Rejection

To demonstrate the AI blocking a bad deployment for your audience:

```bash
# Add a "bad" file to trigger AI security concern
cat >> app/app.py << 'EOF'

# BAD PRACTICE — for demo purposes
DB_PASSWORD = "admin123"
SECRET_KEY  = "hardcoded-secret-do-not-do-this"
EOF

git add app/app.py
git commit -m "demo: introduce security issues"
git push
```

Then run the pipeline again with:
- `DEPLOY_ENV`: `prod`
- `AI_BLOCK_ON_REJECT`: ✅ checked

The AI should detect the hardcoded secrets and block the production deployment! 🎯

---

## 11. Troubleshooting

### ❌ "Cannot reach Ollama" in Jenkins

```bash
# Check if Ollama is running
sudo systemctl status ollama

# Start it
sudo systemctl start ollama

# If Jenkins is in Docker, use host IP not localhost
# Find host IP:
ip route | grep docker | awk '{print $9}'
# Use that IP in the Jenkins credential, e.g.: http://172.17.0.1:11434

# Or allow Ollama on all interfaces:
sudo OLLAMA_HOST=0.0.0.0 ollama serve
```

### ❌ "Model not found" error

```bash
# Pull the model
ollama pull llama3.2:3b

# Verify it downloaded
ollama list

# Check disk space (models need ~2-4 GB)
df -h
```

### ❌ AI response not valid JSON

The model sometimes adds conversational text around JSON. The agent handles this with `safe_parse_json()`. If you still see issues:

```bash
# Switch to a more instruction-following model
ollama pull mistral:7b

# Then set in Jenkins parameter or:
export OLLAMA_MODEL=mistral:7b
python3 ai-agent/ollama_agent.py --path ./app
```

### ❌ Ollama is slow (taking 60+ seconds)

```bash
# Check if GPU is being used
ollama ps   # shows GPU/CPU usage

# Use a smaller model for faster demo
ollama pull llama3.2:1b   # ~1 GB, much faster on CPU

# Or increase timeout in ollama_agent.py:
# Change: timeout=120 → timeout=300
```

### ❌ kubectl rollout timeout

```bash
# Check why pods are not starting
kubectl describe pods -n workshop-dev
kubectl get events -n workshop-dev --sort-by='.lastTimestamp'

# Common: image pull failed — check Docker Hub credentials
kubectl get pods -n workshop-dev
kubectl logs <pod-name> -n workshop-dev
```

### ❌ Port-forward fails in smoke tests

```bash
# Kill existing port-forwards
pkill -f "kubectl port-forward" || true

# Check if service exists
kubectl get svc -n workshop-dev

# Try a different port
kubectl port-forward svc/workshop-app-svc 9876:80 -n workshop-dev
curl http://localhost:9876/health
```

---

## 📊 Quick Reference Card

```
┌──────────────────────────────────────────────────────────────────┐
│                    OLLAMA COMMANDS                                │
├──────────────────────────────────────────────────────────────────┤
│  ollama serve              → Start Ollama server                 │
│  ollama pull llama3.2:3b   → Download model                      │
│  ollama list               → Show downloaded models              │
│  ollama run llama3.2:3b    → Interactive chat                    │
│  ollama ps                 → Show running models + GPU/CPU       │
│  ollama rm llama3.2:3b     → Delete a model                      │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    MODEL RECOMMENDATIONS                          │
├────────────────┬──────────┬──────────┬───────────────────────────┤
│  Model         │  Size    │  RAM     │  Best For                 │
├────────────────┼──────────┼──────────┼───────────────────────────┤
│  llama3.2:1b   │  1.3 GB  │  4 GB+   │  Fast demo, low RAM       │
│  llama3.2:3b   │  2.0 GB  │  8 GB+   │  ✅ Workshop default      │
│  mistral:7b    │  4.1 GB  │  8 GB+   │  Better JSON quality      │
│  codellama:7b  │  3.8 GB  │  8 GB+   │  Best for code review     │
└────────────────┴──────────┴──────────┴───────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    AI AGENT EXIT CODES                            │
├──────────────────────────────────────────────────────────────────┤
│  0 → Success — pipeline proceeds                                 │
│  2 → Security scan blocked deployment                            │
│  3 → AI deploy decision = REJECT                                 │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│                    JENKINS CREDENTIALS NEEDED                     │
├────────────────────┬─────────────────┬────────────────────────────┤
│  ID                │  Type           │  Value                     │
├────────────────────┼─────────────────┼────────────────────────────┤
│  ollama-host       │  Secret text    │  http://localhost:11434    │
│  docker-hub-creds  │  User/Password  │  Docker Hub login          │
│  kubeconfig        │  Secret file    │  ~/.kube/config            │
└────────────────────┴─────────────────┴────────────────────────────┘
```

---

*Workshop Guide — Jenkins + Ollama Local AI Agent*
*100% Free | No API Keys | Runs Fully Offline*
*Estimated Duration: 3–4 hours*
