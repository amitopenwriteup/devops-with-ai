#  Dockerfile Generator Workshop
### GenAI-Powered Dockerfile Generation with Ollama + Llama3 on Ubuntu VM

---

## Table of Contents
1. [Workshop Overview](#workshop-overview)
2. [VM Prerequisites](#vm-prerequisites)
3. [Environment Setup](#environment-setup)
4. [Installing Ollama](#installing-ollama)
5. [Project Setup](#project-setup)
6. [Application Code](#application-code)
7. [Running the Application](#running-the-application)
8. [Hands-On Exercises](#hands-on-exercises)
9. [Troubleshooting](#troubleshooting)
10. [Best Practices Reference](#best-practices-reference)

---

## 1. Workshop Overview

**Duration:** 3–4 hours  
**Level:** Beginner to Intermediate  
**Goal:** Build a GenAI-powered CLI tool that generates optimized Dockerfiles for any programming language using a locally hosted LLM (no cloud/API key required).

### What You'll Learn
- Setting up Ollama (self-hosted LLM runtime) on Ubuntu
- Running Llama3 model locally
- Building a Python CLI that calls a local LLM API
- Generating production-ready Dockerfiles with AI
- Docker best practices across multiple languages

### Architecture

```
User Input (language)
        │
        ▼
Python Script (generate_dockerfile.py)
        │
        ▼
Ollama API (localhost:11434)
        │
        ▼
Llama3.2:1b Model (local inference)
        │
        ▼
Generated Dockerfile (stdout)
```

---

## 2. VM Prerequisites

### Recommended Ubuntu VM Specs

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS        | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| CPU       | 4 cores | 8 cores |
| RAM       | 8 GB | 16 GB |
| Disk      | 20 GB | 40 GB |
| GPU       | Not required | NVIDIA (optional, for speed) |

### Check Your VM Specs
```bash
# CPU cores
nproc

# RAM
free -h

# Disk space
df -h /

# OS version
lsb_release -a
```

---

## 3. Environment Setup

### Step 1 — Update System Packages
```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 — Install Essential Tools
```bash
sudo apt install -y \
  curl \
  wget \
  git \
  python3 \
  python3-pip \
  python3-venv \
  build-essential
```

### Step 3 — Verify Python Installation
```bash
python3 --version
# Expected: Python 3.10+ or higher

pip3 --version
# Expected: pip 22.x or higher
```

---

## 4. Installing Ollama

### Step 1 — Download and Install Ollama
```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This script will:
- Download the Ollama binary
- Install it to `/usr/local/bin/ollama`
- Create a systemd service

### Step 2 — Verify Installation
```bash
ollama --version
# Expected: ollama version 0.x.x
```

### Step 3 — Start the Ollama Service

**Option A — Run as a background service (recommended):**
```bash
sudo systemctl edit ollama
```
```
###Add this in edito

[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
#save it
```
```bash
sudo systemctl enable ollama
sudo systemctl start ollama
sudo systemctl status ollama
```

**Option B — Run manually in a terminal:**
```bash
ollama serve
# Keep this terminal open; open a new tab for next steps
```

### Step 4 — Pull the Llama3.2 Model
```bash
ollama pull llama3.2:1b
```

> ⏳ This downloads ~1.3 GB. Time depends on your internet speed.

### Step 5 — Test the Model
```bash
ollama run llama3.2:1b "Say hello in one sentence"
```

Expected output: A short greeting from the model.

### Step 6 — Verify the API is Running
```bash
curl http://localhost:11434/api/tags
```

Expected: JSON response listing available models.

---

## 5. Project Setup

### Step 1 — Create Project Directory
```bash
mkdir ~/dockerfile-generator
cd ~/dockerfile-generator
```

### Step 2 — Create Virtual Environment
```bash
python3 -m venv venv
source venv/bin/activate
```

You should now see `(venv)` in your terminal prompt.

### Step 3 — Create requirements.txt
```bash
cat > requirements.txt << 'EOF'
requests==2.31.0
EOF
```

### Step 4 — Install Dependencies
```bash
pip3 install -r requirements.txt
```

---

## 6. Application Code

### Main Script — generate_dockerfile.py
```bash
cat > generate_dockerfile.py << 'EOF'
import requests
import json
import sys

OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL = "llama3.2:1b"

def build_prompt(language: str) -> str:
    return f"""You are a Docker expert. Generate a production-ready, optimized Dockerfile for a {language} application.

Follow these best practices:
1. Use official base images with specific version tags (avoid 'latest')
2. Use multi-stage builds where appropriate to minimize final image size
3. ADD as a non-root user for security
4. Use .dockerignore-friendly layer ordering (copy dependencies before source code)
5. Add HEALTHCHECK instruction
6. Include meaningful LABELS
7. Use ARG for build-time variables
8. Set proper WORKDIR

Output ONLY the Dockerfile content with inline comments explaining each decision.
Do not include any explanation outside the Dockerfile itself."""


def generate_dockerfile(language: str) -> str:
    prompt = build_prompt(language)
    payload = {
        "model": MODEL,
        "prompt": prompt,
        "stream": True
    }

    print(f"\n Generating Dockerfile for {language}...\n")
    print("=" * 60)

    full_response = ""
    try:
        with requests.post(OLLAMA_URL, json=payload, stream=True, timeout=120) as response:
            response.raise_for_status()
            for line in response.iter_lines():
                if line:
                    data = json.loads(line)
                    chunk = data.get("response", "")
                    print(chunk, end="", flush=True)
                    full_response += chunk
                    if data.get("done"):
                        break
    except requests.exceptions.ConnectionError:
        print(" Error: Cannot connect to Ollama. Make sure it is running:")
        print("   ollama serve")
        sys.exit(1)
    except requests.exceptions.Timeout:
        print(" Error: Request timed out. The model may be loading.")
        sys.exit(1)

    print("\n" + "=" * 60)
    return full_response


def save_dockerfile(content: str, language: str):
    filename = f"Dockerfile.{language.lower().replace(' ', '_')}"
    with open(filename, "w") as f:
        f.write(content)
    print(f"\n Saved to: {filename}")


def main():
    print(" Dockerfile Generator — Powered by Ollama + Llama3")
    print("-" * 50)

    if len(sys.argv) > 1:
        language = " ".join(sys.argv[1:])
    else:
        language = input("Enter programming language (e.g., Python, Node.js, Java, Go): ").strip()

    if not language:
        print(" Error: Please provide a programming language.")
        sys.exit(1)

    dockerfile_content = generate_dockerfile(language)

    save = input("\n Save to file? (y/n): ").strip().lower()
    if save == "y":
        save_dockerfile(dockerfile_content, language)


if __name__ == "__main__":
    main()
EOF
```

### Optional — Enhanced Version with Multiple Languages
```bash
cat > batch_generate.py << 'EOF'
import subprocess
import sys

languages = ["Python", "Node.js", "Java", "Go", "Rust"]

print(" Batch Dockerfile Generator")
print(f"Generating Dockerfiles for: {', '.join(languages)}\n")

for lang in languages:
    print(f"\n{'='*60}")
    print(f"Processing: {lang}")
    print('='*60)
    result = subprocess.run(
        [sys.executable, "generate_dockerfile.py", lang],
        input="y\n",
        text=True,
        capture_output=False
    )
EOF
```

---

## 7. Running the Application

### Interactive Mode
```bash
# Make sure virtual environment is active
source venv/bin/activate

python3 generate_dockerfile.py
```

Sample session:
```
 Dockerfile Generator — Powered by Ollama + Llama3
--------------------------------------------------
Enter programming language: Python

 Generating Dockerfile for Python...

============================================================
# Stage 1: Build stage
FROM python:3.11-slim AS builder
...
============================================================

 Save to file? (y/n): y
 Saved to: Dockerfile.python
```

### Command-Line Mode
```bash
python3 generate_dockerfile.py Python
python3 generate_dockerfile.py "Node.js"
python3 generate_dockerfile.py Go
python3 generate_dockerfile.py Java
```

---

## 8. Hands-On Exercises

### Exercise 1 — Basic Generation (10 min)
Generate Dockerfiles for three languages and compare them:
```bash
python3 generate_dockerfile.py Python
python3 generate_dockerfile.py Go
python3 generate_dockerfile.py "Node.js"
```
**Discussion:** What differs between multi-stage and single-stage builds? Which image is smallest?

---

### Exercise 2 — Validate the Dockerfile (15 min)
Install Docker and build the generated image:
```bash
# Install Docker
sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

# Create a sample Python app
mkdir sample-app && cd sample-app
echo "print('Hello from Docker!')" > app.py
cp ../Dockerfile.python Dockerfile

# Build the image
docker build -t my-python-app .

# Run it
docker run --rm my-python-app
```

---

### Exercise 3 — Customize the Prompt (20 min)
Edit `generate_dockerfile.py` and modify the `build_prompt()` function to add extra requirements:

```python
# Try adding these to the prompt:
"5. Install and configure a health check endpoint on port 8080"
"6. Add support for reading environment variables from a .env file"
"7. Optimize for a FastAPI web application specifically"
```

Re-run and observe how the generated Dockerfile changes.

---

### Exercise 4 — Try Different Models (15 min)
```bash
# Pull a larger model for better quality
ollama pull llama3.2:3b

# Edit generate_dockerfile.py — change MODEL variable:
# MODEL = "llama3.2:3b"

python3 generate_dockerfile.py Python
```
**Discussion:** How does output quality differ between 1b and 3b models?

---



---

## 9. Troubleshooting

### "Cannot connect to Ollama"
```bash
# Check if Ollama is running
sudo systemctl status ollama

# Start if stopped
sudo systemctl start ollama

# Or run manually
ollama serve
```

###  "Model not found"
```bash
# List available models
ollama list

# Re-pull the model
ollama pull llama3.2:1b
```

###  "Request timed out"
The model may be loading for the first time. Wait 30–60 seconds and retry. For faster response:
```bash
# Pre-warm the model
ollama run llama3.2:1b "hello"
```

###  "pip: command not found"
```bash
sudo apt install python3-pip -y
```

###  "(venv) not appearing after activation"
```bash
# Re-create the virtual environment
python3 -m venv venv --clear
source venv/bin/activate
```

###  Ollama using too much RAM
```bash
# Use the smaller 1b model (recommended for VMs with 8GB RAM)
ollama pull llama3.2:1b

# Check memory usage
free -h
htop
```

---

## 10. Best Practices Reference

### Dockerfile Best Practices Checklist

| Practice | Why It Matters |
|----------|---------------|
| Pin base image versions (`python:3.11-slim`) | Reproducible builds; avoids surprise breakages |
| Multi-stage builds | Reduces final image size dramatically |
| Non-root user (`USER appuser`) | Reduces attack surface |
| Copy `requirements.txt` before source code | Leverages Docker layer caching |
| Use `.dockerignore` | Prevents secrets/unnecessary files entering build context |
| `HEALTHCHECK` instruction | Enables orchestrators (K8s, Swarm) to detect unhealthy containers |
| Minimal base images (`slim`, `alpine`) | Smaller images = faster pulls, less attack surface |
| `WORKDIR` instead of `RUN mkdir` | Cleaner, idiomatic |

### Quick .dockerignore Template
```
.git
.env
*.pyc
__pycache__/
venv/
node_modules/
*.log
.DS_Store
README.md
```

### Useful Ollama Commands

```bash
ollama list              # List downloaded models
ollama ps                # Show running models
ollama rm llama3.2:1b    # Remove a model
ollama pull codellama    # Pull a code-specialized model
ollama show llama3.2:1b  # Show model details
```

---

## Final Project Structure

```
~/dockerfile-generator/
├── venv/                        # Python virtual environment
├── generate_dockerfile.py       # Main application
├── batch_generate.py            # Batch generation script
├── requirements.txt             # Python dependencies
├── Dockerfile.python            # Generated output
├── Dockerfile.node_js           # Generated output
└── Dockerfile.go                # Generated output
```

---

## 🎓 Workshop Complete!

You have successfully:
- Set up a self-hosted LLM environment with Ollama on Ubuntu
- Built a GenAI-powered CLI tool using Python
- Generated optimized Dockerfiles using local AI inference
- Learned Docker best practices applied by the model

### Next Steps
- Try `ollama pull codellama` for a code-specialized model
- Extend the tool to generate `docker-compose.yml` files
- Add a web UI using Flask or FastAPI
- Integrate with a CI/CD pipeline to auto-generate Dockerfiles on commit

---

*Workshop guide — Dockerfile Generator with Ollama + Llama3 on Ubuntu*
