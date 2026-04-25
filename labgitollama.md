# GitLab CI Pipeline Generator Workshop
### GenAI-Powered .gitlab-ci.yml Generation with Ollama + Llama3 on Ubuntu VM

---

## Table of Contents

1. [Workshop Overview](#1-workshop-overview)
2. [VM Prerequisites](#2-vm-prerequisites)
3. [Environment Setup](#3-environment-setup)
4. [Installing Ollama](#4-installing-ollama)
5. [Project Setup](#5-project-setup)
6. [Application Code](#6-application-code)
7. [Running the Application](#7-running-the-application)
8. [Hands-On Exercises](#8-hands-on-exercises)
9. [Troubleshooting](#9-troubleshooting)
10. [Best Practices Reference](#10-best-practices-reference)

---

## 1. Workshop Overview

**Duration:** 3-4 hours
**Level:** Beginner to Intermediate
**Goal:** Build a GenAI-powered CLI tool that generates optimized `.gitlab-ci.yml` pipelines for Java 11 projects using a locally hosted LLM — no cloud account or API key required.

### What You Will Learn

- Setting up Ollama (self-hosted LLM runtime) on Ubuntu
- Running Llama3 locally
- Applying zero-shot, few-shot, and chain-of-thought prompting techniques
- Building a Python CLI that calls a local LLM API
- Generating production-ready GitLab CI pipelines with AI
- Understanding how prompt style changes the quality of CI output

### Architecture

```
User Input (pipeline type / language / stages)
        |
        v
Python Script (generate_pipeline.py)
        |
        v
Prompt Engine (zero-shot / few-shot / chain-of-thought)
        |
        v
Ollama API (localhost:11434)
        |
        v
Llama3.2:1b Model (local inference)
        |
        v
Generated .gitlab-ci.yml (stdout + saved file)
```

### How Prompting Connects to Pipeline Design

Before writing a single line of code today, understand this mapping. You will see it appear again in every exercise.

| Prompting Style | What You Do | Pipeline Equivalent |
|----------------|-------------|---------------------|
| Zero-shot | One instruction, no examples | "Write a compile job for Maven" |
| Few-shot | Show 2-3 examples, then ask | Define compile + test, then ask for build using the same shape |
| Chain-of-thought | Reason step by step before generating | "First compile, then test only if compile passes, then..." |

---

## 2. VM Prerequisites

### Recommended Ubuntu VM Specs

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 20 GB | 40 GB |
| GPU | Not required | NVIDIA (optional, for speed) |

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

### Step 1 - Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2 - Install Essential Tools

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

### Step 3 - Verify Python Installation

```bash
python3 --version
# Expected: Python 3.10 or higher

pip3 --version
# Expected: pip 22.x or higher
```

---

## 4. Installing Ollama

### Step 1 - Download and Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

This script will:
- Download the Ollama binary
- Install it to `/usr/local/bin/ollama`
- Create a systemd service

### Step 2 - Verify Installation

```bash
ollama --version
# Expected: ollama version 0.x.x
```

### Step 3 - Start the Ollama Service

Option A — Run as a background service (recommended):

```bash
sudo systemctl enable ollama
sudo systemctl start ollama
sudo systemctl status ollama
```

Option B — Run manually in a terminal:

```bash
ollama serve
# Keep this terminal open. Open a new tab for next steps.
```

### Step 4 - Pull the Llama3.2 Model

```bash
ollama pull llama3.2:1b
```

This downloads approximately 1.3 GB. Time depends on your connection speed.

### Step 5 - Test the Model

```bash
ollama run llama3.2:1b "Say hello in one sentence"
```

Expected output: a short greeting from the model.

### Step 6 - Verify the API is Running

```bash
curl http://localhost:11434/api/tags
```

Expected: a JSON response listing available models.

---

## 5. Project Setup

### Step 1 - Create Project Directory

```bash
mkdir ~/gitlab-ci-generator
cd ~/gitlab-ci-generator
```

### Step 2 - Create Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

You should see `(venv)` in your terminal prompt.

### Step 3 - Create requirements.txt

```bash
cat > requirements.txt << 'EOF'
requests==2.31.0
EOF
```

### Step 4 - Install Dependencies

```bash
pip3 install -r requirements.txt
```

---

## 6. Application Code

### Main Script - generate_pipeline.py

This script has three prompting modes. Each one produces a different quality of output. That difference is the lesson.

```bash
cat > generate_pipeline.py << 'EOF'
import requests
import json
import sys

OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL = "llama3.2:1b"

# ─────────────────────────────────────────────────────────────
# PROMPT BUILDERS
# Three strategies — same task, different instruction quality
# ─────────────────────────────────────────────────────────────

def build_zero_shot_prompt(language: str, stages: list) -> str:
    """
    Zero-shot: one instruction, no examples.
    We trust the model to know GitLab CI syntax and Maven conventions.
    Best for: simple, well-known tasks.
    """
    return f"""You are a GitLab CI expert.
Write a production-ready .gitlab-ci.yml for a {language} project with these stages: {', '.join(stages)}.
Output only the YAML. No explanation."""


def build_few_shot_prompt(language: str, stages: list) -> str:
    """
    Few-shot: we show the model two example jobs so it learns our preferred structure,
    then ask it to continue the pattern for the remaining stages.
    Best for: enforcing a consistent job shape across a large pipeline.
    """
    return f"""You are a GitLab CI expert writing a pipeline for a {language} project.

Here are two example jobs that follow the structure I want:

compile:
  stage: compile
  image: maven:3.8-openjdk-11
  script:
    - mvn compile -B
  tags:
    - docker

test:
  stage: test
  image: maven:3.8-openjdk-11
  script:
    - mvn test -B
  artifacts:
    when: always
    reports:
      junit:
        - target/surefire-reports/*.xml
    expire_in: 1 week
  tags:
    - docker

Following exactly the same structure and style, write the remaining jobs for these stages: {', '.join(stages)}.
Also write the stages block and any global variables needed.
Output only the complete .gitlab-ci.yml. No explanation."""


def build_chain_of_thought_prompt(language: str, stages: list) -> str:
    """
    Chain-of-thought: we reason through the pipeline step by step before asking for code.
    Each stage is explained with its dependency on the previous one.
    Best for: complex pipelines where order and failure conditions matter.
    """
    stage_reasoning = "\n".join([
        f"  Step {i+1} ({stage}): runs only if step {i} passed. "
        f"If this fails, nothing after it should execute."
        for i, stage in enumerate(stages)
    ])

    return f"""You are a GitLab CI expert. Think step by step before generating the pipeline.

Project: {language} application using Maven and Docker.

Reasoning through the pipeline:
{stage_reasoning}

Key constraints to apply at each step:
- compile: use maven:3.8-openjdk-11, run mvn compile -B, tag runner as docker
- test: save JUnit XML artifacts from target/surefire-reports/, expire in 1 week
- build: use docker:24 image with docker:24-dind service, authenticate to Docker Hub using $DOCKER_USERNAME and $DOCKER_PASSWORD, tag image as $DOCKER_USERNAME/java11-demo:$CI_COMMIT_SHORT_SHA, only run on main branch
- scan: use aquasec/trivy:latest with empty entrypoint, fail on CRITICAL or HIGH severity, save HTML report as artifact, only run on main branch
- publish: re-tag image with git tag if $CI_COMMIT_TAG exists, only run on main branch and tags

Now generate the complete .gitlab-ci.yml applying this reasoning. Include:
- stages block
- global variables block (MAVEN_OPTS, IMAGE_NAME, IMAGE_TAG)
- cache block for .m2/repository
- one job per stage

Output only the YAML. No explanation outside comments inside the YAML itself."""


# ─────────────────────────────────────────────────────────────
# GENERATOR
# ─────────────────────────────────────────────────────────────

def generate_pipeline(prompt: str, mode: str) -> str:
    payload = {
        "model": MODEL,
        "prompt": prompt,
        "stream": True
    }

    print(f"\nGenerating pipeline using {mode} prompting...\n")
    print("=" * 60)

    full_response = ""
    try:
        with requests.post(OLLAMA_URL, json=payload, stream=True, timeout=180) as response:
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
        print("Error: Cannot connect to Ollama. Make sure it is running:")
        print("  ollama serve")
        sys.exit(1)
    except requests.exceptions.Timeout:
        print("Error: Request timed out. The model may be loading. Wait 30 seconds and retry.")
        sys.exit(1)

    print("\n" + "=" * 60)
    return full_response


def save_pipeline(content: str, mode: str):
    filename = f"gitlab-ci-{mode}.yml"
    with open(filename, "w") as f:
        f.write(content)
    print(f"\nSaved to: {filename}")


# ─────────────────────────────────────────────────────────────
# MAIN
# ─────────────────────────────────────────────────────────────

def main():
    print("GitLab CI Pipeline Generator — Powered by Ollama + Llama3")
    print("Prompt Engineering Modes: zero-shot | few-shot | chain-of-thought")
    print("-" * 60)

    language = input("Project language/stack (e.g., Java Maven, Python, Node.js) [default: Java Maven]: ").strip()
    if not language:
        language = "Java Maven"

    print("\nAvailable stages: compile, test, build, scan, publish")
    stages_input = input("Stages to include (comma-separated) [default: all five]: ").strip()
    if not stages_input:
        stages = ["compile", "test", "build", "scan", "publish"]
    else:
        stages = [s.strip() for s in stages_input.split(",")]

    print("\nSelect prompting mode:")
    print("  1 - Zero-shot   (one instruction, model figures it out)")
    print("  2 - Few-shot    (examples shown, model continues the pattern)")
    print("  3 - Chain-of-thought (step-by-step reasoning, then generate)")
    print("  4 - All three   (generate all and compare)")

    choice = input("\nEnter choice [1-4]: ").strip()

    mode_map = {
        "1": [("zero-shot", build_zero_shot_prompt)],
        "2": [("few-shot", build_few_shot_prompt)],
        "3": [("chain-of-thought", build_chain_of_thought_prompt)],
        "4": [
            ("zero-shot", build_zero_shot_prompt),
            ("few-shot", build_few_shot_prompt),
            ("chain-of-thought", build_chain_of_thought_prompt),
        ],
    }

    modes = mode_map.get(choice, mode_map["3"])

    for mode_name, prompt_fn in modes:
        prompt = prompt_fn(language, stages)
        content = generate_pipeline(prompt, mode_name)

        save = input(f"\nSave {mode_name} pipeline to file? (y/n): ").strip().lower()
        if save == "y":
            save_pipeline(content, mode_name)

    print("\nDone. Compare the outputs to see how prompting style affects pipeline quality.")


if __name__ == "__main__":
    main()
EOF
```

### Optional - Batch Compare Script

Run all three modes back to back and save all outputs for side-by-side comparison.

```bash
cat > batch_compare.py << 'EOF'
import subprocess
import sys

modes = ["1", "2", "3"]
mode_names = ["zero-shot", "few-shot", "chain-of-thought"]
language = "Java Maven"
stages = "compile,test,build,scan,publish"

print("GitLab CI Generator — Batch Mode Comparison")
print(f"Language: {language}")
print(f"Stages: {stages}")
print(f"Running all three prompting modes...\n")

for mode, name in zip(modes, mode_names):
    print(f"\n{'='*60}")
    print(f"Mode: {name}")
    print("=" * 60)
    subprocess.run(
        [sys.executable, "generate_pipeline.py"],
        input=f"{language}\n{stages}\n{mode}\ny\n",
        text=True
    )

print("\nAll three pipelines generated.")
print("Files saved: gitlab-ci-zero-shot.yml, gitlab-ci-few-shot.yml, gitlab-ci-chain-of-thought.yml")
print("\nOpen them side by side and compare:")
print("  - Which one is most complete?")
print("  - Which one has correct YAML indentation?")
print("  - Which one includes security best practices?")
print("  - Which one would you trust to run in production?")
EOF
```

---

## 7. Running the Application

### Interactive Mode

```bash
# Make sure the virtual environment is active
source venv/bin/activate

python3 generate_pipeline.py
```

Sample session:

```
GitLab CI Pipeline Generator — Powered by Ollama + Llama3
Prompt Engineering Modes: zero-shot | few-shot | chain-of-thought
------------------------------------------------------------
Project language/stack [default: Java Maven]:

Available stages: compile, test, build, scan, publish
Stages to include [default: all five]:

Select prompting mode:
  1 - Zero-shot
  2 - Few-shot
  3 - Chain-of-thought
  4 - All three (generate all and compare)

Enter choice [1-4]: 3

Generating pipeline using chain-of-thought prompting...

============================================================
stages:
  - compile
  - test
  - build
  - scan
  - publish
...
============================================================

Save chain-of-thought pipeline to file? (y/n): y
Saved to: gitlab-ci-chain-of-thought.yml
```

### Command-Line Shortcut for Each Mode

```bash
# Zero-shot only
echo -e "Java Maven\n\n1\ny" | python3 generate_pipeline.py

# Few-shot only
echo -e "Java Maven\n\n2\ny" | python3 generate_pipeline.py

# Chain-of-thought only
echo -e "Java Maven\n\n3\ny" | python3 generate_pipeline.py

# All three for comparison
python3 batch_compare.py
```

---

## 8. Hands-On Exercises

### Exercise 1 - Zero-Shot Baseline (15 min)

Run the generator in zero-shot mode for a Java Maven project with all five stages.

```bash
python3 generate_pipeline.py
# Choose: Java Maven / all stages / mode 1
```

Open the saved file and answer these questions before moving on:

- Does the output include a `stages:` block?
- Does it use the correct Maven image?
- Does it include runner tags?
- Does it handle Docker Hub credentials securely?
- Would you commit this to a real project as-is?

Write your observations down. You will compare them after running the other modes.

---

### Exercise 2 - Few-Shot Pattern Enforcement (15 min)

Run the generator in few-shot mode. The script provides two example jobs (compile and test) and asks the model to continue the pattern.

```bash
python3 generate_pipeline.py
# Choose: Java Maven / all stages / mode 2
```

Compare the output with your zero-shot result:

- Did the model follow the job structure from the examples?
- Are the build, scan, and publish jobs shaped like the compile and test examples?
- Is the YAML indentation consistent throughout?

Discussion: What changed when you gave examples versus giving no examples?

---

### Exercise 3 - Chain-of-Thought Full Pipeline (20 min)

Run the generator in chain-of-thought mode. The prompt walks through each stage's dependencies and constraints before asking for code.

```bash
python3 generate_pipeline.py
# Choose: Java Maven / all stages / mode 3
```

After it completes:

1. Open `gitlab-ci-chain-of-thought.yml`
2. Check each of these manually without using Copilot or the model:

```
compile job:
  [ ] Uses maven:3.8-openjdk-11
  [ ] Runs mvn compile -B
  [ ] Has tags: [docker]

test job:
  [ ] Saves JUnit artifacts from target/surefire-reports/
  [ ] artifact expires_in is set

build job:
  [ ] Uses docker:24 image
  [ ] Has docker:24-dind as a service
  [ ] Sets DOCKER_TLS_CERTDIR: "/certs"
  [ ] Authenticates to Docker Hub in before_script
  [ ] Only runs on main branch

scan job:
  [ ] Uses aquasec/trivy:latest
  [ ] Has entrypoint: [""]
  [ ] Fails on CRITICAL,HIGH
  [ ] Saves HTML report as artifact
  [ ] Only runs on main branch

publish job:
  [ ] Handles git tag logic
  [ ] Only runs on main and tags
```

For any box you cannot tick, ask the model to fix just that item:

> "Here is my GitLab CI scan job: [paste job]. Add the entrypoint override and make the artifact save even on failure."

---

### Exercise 4 - Side-by-Side Comparison (20 min)

Run all three modes at once and compare the outputs.

```bash
python3 batch_compare.py
```

Open three terminals or editor tabs and load all three files:

```bash
# In three separate terminals:
cat gitlab-ci-zero-shot.yml
cat gitlab-ci-few-shot.yml
cat gitlab-ci-chain-of-thought.yml
```

Fill in this comparison table:

| Feature | Zero-shot | Few-shot | Chain-of-thought |
|---------|-----------|----------|-----------------|
| stages block present | | | |
| global variables defined | | | |
| Maven cache configured | | | |
| Docker Hub auth present | | | |
| Trivy scan included | | | |
| only: main on build/scan/publish | | | |
| non-root user in Dockerfile reference | | | |
| Would pass a code review | | | |

Discussion question: Which mode produced the most complete output? Does that match your expectation?

---

### Exercise 5 - Customize the Chain-of-Thought Prompt (25 min)

Edit `generate_pipeline.py` and find the `build_chain_of_thought_prompt` function. Add one of these requirements to the reasoning section and regenerate:

Option A — Add SAST scanning:
```
- After the test stage, add a sast stage that runs the GitLab SAST template
```

Option B — Add environment-specific deployments:
```
- After publish, add a deploy-staging stage that only runs on the staging branch
- Add a deploy-prod stage that only runs on version tags (v*)
```

Option C — Add Slack notification:
```
- After publish, add a notify stage using alpine with curl to POST to $SLACK_WEBHOOK_URL
```

Regenerate and check: did the model incorporate your addition correctly? If not, what would you change in the prompt to get a better result?

---

### Exercise 6 - Try a Larger Model (15 min)

```bash
# Pull the 3b parameter model (higher quality, slower)
ollama pull llama3.2:3b

# Edit generate_pipeline.py — change the MODEL variable:
# MODEL = "llama3.2:3b"

# Run chain-of-thought again
python3 generate_pipeline.py
# Choose: Java Maven / all stages / mode 3
```

Compare the output to what the 1b model produced. Specifically check:

- Is the YAML indentation more consistent?
- Does it handle the Docker Hub auth block correctly?
- Does it include the `only: main` restriction without being told explicitly?

Discussion: When does model size matter more — zero-shot or chain-of-thought? Why?

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

### "Model not found"

```bash
# List available models
ollama list

# Re-pull the model
ollama pull llama3.2:1b
```

### "Request timed out"

The model may still be loading. Wait 30-60 seconds then retry. To pre-warm it:

```bash
ollama run llama3.2:1b "hello"
```

### "pip: command not found"

```bash
sudo apt install python3-pip -y
```

### "(venv) not appearing after activation"

```bash
python3 -m venv venv --clear
source venv/bin/activate
```

### "Generated YAML has wrong indentation"

This is a known limitation of smaller models. Try these in order:

1. Switch to `llama3.2:3b` for better YAML accuracy
2. Add this line to the end of your prompt: `Ensure all YAML uses 2-space indentation throughout.`
3. Use the chain-of-thought prompt — it produces more structurally consistent output than zero-shot

### "Ollama using too much RAM"

```bash
# Check memory usage
free -h

# Use the 1b model if RAM is limited
ollama pull llama3.2:1b

# Remove the larger model if needed
ollama rm llama3.2:3b
```

---

## 10. Best Practices Reference

### GitLab CI Pipeline Checklist

| Practice | Why It Matters |
|----------|----------------|
| Pin Docker image versions (`maven:3.8-openjdk-11`) | Reproducible jobs; avoids surprise breakages from image updates |
| Use runner tags on every job | Jobs without tags run on any available runner — unpredictable in shared environments |
| Cache `.m2/repository` | Dramatically reduces compile and test job duration |
| Store secrets as masked CI variables | Secrets in code are a security incident waiting to happen |
| Use `$CI_COMMIT_SHORT_SHA` for image tags | Traceability — every image maps to a specific commit |
| Scan before publish | Stops vulnerable images from reaching Docker Hub |
| `allow_failure: false` on scan job | Ensures the pipeline actually stops on findings |
| Save artifacts with `when: always` | Reports are available even when the job fails |
| Restrict build/scan/publish to `only: main` | Avoids pushing images from feature branches |
| Use `before_script` for auth | Separates credential setup from business logic |

### Prompt Engineering Checklist for CI Generation

| Technique | When to Use | Expected Benefit |
|-----------|------------|-----------------|
| Zero-shot | Simple, well-known job types | Fast, good enough for standard jobs |
| Few-shot | You need consistent job structure | Model matches your pattern exactly |
| Chain-of-thought | Complex pipelines with dependencies | More complete, more accurate constraints |
| Iterative fixing | Model missed one specific thing | Cheaper than regenerating everything |
| Paste error into prompt | Job fails in GitLab | Model explains cause and fix |

### Useful Ollama Commands

```bash
ollama list                  # List downloaded models
ollama ps                    # Show running models
ollama rm llama3.2:1b        # Remove a model
ollama pull codellama        # Pull a code-specialized model
ollama show llama3.2:1b      # Show model details and parameters
```

### GitLab CI Built-in Variables Reference

| Variable | Example | Use in pipeline |
|----------|---------|----------------|
| `$CI_COMMIT_SHORT_SHA` | `a1b2c3d4` | Docker image tag |
| `$CI_COMMIT_REF_NAME` | `main` | Branch-conditional logic |
| `$CI_COMMIT_TAG` | `v1.2.0` | Version tagging on publish |
| `$CI_PROJECT_NAME` | `java11` | Notification messages |
| `$CI_PIPELINE_URL` | `https://gitlab.com/...` | Slack/Teams notifications |
| `$GITLAB_USER_LOGIN` | `alice` | Audit trail in notifications |

---

## Final Project Structure

```
~/gitlab-ci-generator/
├── venv/                              # Python virtual environment
├── generate_pipeline.py               # Main application (3 prompt modes)
├── batch_compare.py                   # Run all 3 modes and compare
├── requirements.txt                   # Python dependencies
├── gitlab-ci-zero-shot.yml            # Generated output — zero-shot
├── gitlab-ci-few-shot.yml             # Generated output — few-shot
└── gitlab-ci-chain-of-thought.yml     # Generated output — chain-of-thought
```

---

## Workshop Complete

You have successfully:

- Set up a self-hosted LLM environment with Ollama on Ubuntu
- Built a GenAI-powered CLI tool for GitLab CI pipeline generation
- Applied zero-shot, few-shot, and chain-of-thought prompting to the same task
- Compared output quality across prompting styles
- Generated a complete compile, test, build, scan, and publish pipeline using local AI inference

### Next Steps

- Try `ollama pull codellama` for a code-specialized model and compare YAML accuracy
- Extend the tool to generate `docker-compose.yml` for local development alongside the pipeline
- Add a validation step using `yamllint` to automatically score YAML correctness per mode
- Integrate this generator into a pre-commit hook that drafts a pipeline whenever a new `pom.xml` is detected

---

*Workshop guide — GitLab CI Pipeline Generator with Ollama + Llama3 on Ubuntu*
*Companion to: Dockerfile Generator Workshop (doc2) and GitLab CI Copilot Participant Guide (doc1)*
