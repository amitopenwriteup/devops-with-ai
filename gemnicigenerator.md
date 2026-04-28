# Workshop: Build a Git-Aware AI Tool with Gemini API
### Scan Repos · Detect Changesets · Auto-Generate `.gitlab-ci.yml`

No Claude Desktop required. Everything runs from the terminal or browser.

---

## Table of Contents

1. [Prerequisites](#prerequisites-checklist)
2. [Getting Your Gemini API Key](#getting-your-gemini-api-key)
3. [What We're Building](#what-were-building)
4. [Project Structure](#project-structure)
5. [Part 1 — Setup and Installation](#part-1--setup-and-installation)
6. [Part 2 — Build the Core Tools](#part-2--build-the-core-tools)
7. [Part 3 — Option A: Terminal CLI](#part-3--option-a-terminal-cli)
8. [Part 4 — Option B: Browser UI](#part-4--option-b-browser-ui)
9. [Gemini Model Reference](#gemini-model-reference)
10. [Part 5 — Hands-On Exercises](#part-5--hands-on-exercises)
11. [Part 6 — Troubleshooting](#part-6--troubleshooting)
12. [Part 7 — Extend the Tool](#part-7--extend-the-tool-bonus)
13. [Ollama → Gemini Migration Reference](#ollama--gemini-migration-reference)
14. [Reference Links](#reference-links)

---

## Prerequisites Checklist

| Tool | Version | Check Command |
|------|---------|---------------|
| Python | >= 3.11 | `python --version` |
| Git | >= 2.x | `git --version` |
| Google Gemini API Key | — | https://aistudio.google.com/apikey |
| pip | Latest | `pip --version` |

> **Get your API key free at** https://aistudio.google.com/apikey — the free tier includes generous rate limits suitable for this workshop.

---

## Getting Your Gemini API Key

### Step 0.1 — Sign in to Google AI Studio

1. Open https://aistudio.google.com in your browser
2. Click **Sign in** in the top-right corner
3. Sign in with any Google account (Gmail, Workspace, etc.)
4. If prompted, accept the Google AI Studio terms of service

> You do not need a Google Cloud account or billing setup for the free tier.

---

### Step 0.2 — Create an API Key

1. Once signed in, click **Get API key** in the left sidebar

   ![Google AI Studio sidebar showing Get API key link]

2. Click the blue **Create API key** button

3. A dialog appears asking where to create the key. You have two options:

   | Option | When to use |
   |--------|-------------|
   | **Create API key in new project** | First time — Google creates a project for you automatically |
   | **Create API key in existing project** | If you already have a Google Cloud project |

   For this workshop, choose **Create API key in new project**.

4. Your API key is generated and displayed. It looks like:

   ```
   AIzaSyD-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
   ```

5. Click **Copy** to copy it to your clipboard

> **This is the only time the full key is shown.** Copy it now. If you lose it, you can always generate a new one from the same page — old keys can be deleted.

---

### Step 0.3 — Understand Key Permissions and Limits

The key you just created has access to all Gemini models under your Google account's free quota.

**Free tier limits (no billing required):**

| Model | Requests / min | Tokens / min | Requests / day |
|-------|---------------|--------------|----------------|
| gemini-2.0-flash | 15 | 1,000,000 | 1,500 |
| gemini-1.5-flash | 15 | 1,000,000 | 1,500 |
| gemini-1.5-pro | 2 | 32,000 | 50 |

This is more than sufficient for this workshop.

---

### Step 0.4 — Secure the Key

API keys are credentials. Treat them like passwords.

**Never do this:**
```bash
# Do NOT hardcode the key in your source code
GEMINI_API_KEY = "AIzaSyD-xxxx..."   # BAD — will leak if you push to Git
```

**Always do this instead:**
```bash
# Store in .env (which is listed in .gitignore)
echo 'GEMINI_API_KEY=AIzaSyD-xxxx...' > .env
echo '.env' >> .gitignore
```

Verify `.gitignore` is protecting it:
```bash
git status
# .env should NOT appear in the output
```

---

### Step 0.5 — Verify the Key Works

Run this one-liner before starting the workshop to confirm everything is connected:

```bash
python - <<'EOF'
import google.generativeai as genai
import os
from dotenv import load_dotenv

load_dotenv()
key = os.environ.get("GEMINI_API_KEY", "")
if not key:
    print("ERROR: GEMINI_API_KEY not found in .env")
else:
    genai.configure(api_key=key)
    model = genai.GenerativeModel("gemini-2.0-flash")
    response = model.generate_content("Reply with: API key works.")
    print(response.text)
EOF
```

Expected output:
```
API key works.
```

If you see an error, check the table below:

| Error message | Fix |
|---------------|-----|
| `GEMINI_API_KEY not found` | Check `.env` file exists and key name is spelled correctly |
| `API_KEY_INVALID` | Key was copied incorrectly — regenerate from AI Studio |
| `PERMISSION_DENIED` | Key may be restricted; create a new unrestricted key |
| `ModuleNotFoundError: google` | Run `pip install google-generativeai python-dotenv` |

---

### Step 0.6 — (Optional) Restrict or Rotate the Key

If you are sharing this project with others or deploying it, restrict the key:

1. Go to https://aistudio.google.com/apikey
2. Find your key and click the three-dot menu → **Edit**
3. Under **API restrictions**, select **Restrict key** → choose **Generative Language API**
4. Click **Save**

To rotate (replace) a compromised key:
1. Click **Delete** on the old key
2. Click **Create API key** to generate a fresh one
3. Update your `.env` file with the new value

---

## What We're Building

```
+---------------------+       +---------------------+
|   cli.py (terminal) |       |  app.py (browser UI) |
|   or                |       |  http://localhost:5000|
+----------+----------+       +---------+-----------+
           |                            |
           +------------+---------------+
                        |
           +------------v------------+
           |       tools.py          |
           |  - scan_repo            |
           |  - get_changeset        |
           |  - analyze_issues       |
           |  - generate_ci          |
           +------------+------------+
                        |
           +------------v------------+
           |   Google Gemini API     |
           |   gemini-2.0-flash      |
           |   generativelanguage    |
           |   .googleapis.com       |
           +-------------------------+
```

Two ways to run:

- **Option A** — `cli.py` : interactive terminal menu
- **Option B** — `app.py` : browser UI using Flask

---

## Project Structure

```
git-ai-tool/
├── tools.py           <- Core logic (git + Gemini API)
├── cli.py             <- Terminal interface (Option A)
├── app.py             <- Browser UI with Flask (Option B)
├── requirements.txt   <- Dependencies
├── .env               <- Your Gemini API key (never commit this)
└── README.md
```

---

## Part 1 — Setup and Installation

### Step 1.1 — Create the Project

```bash
mkdir git-ai-tool
cd git-ai-tool
python3 -m venv venv

# Activate
# macOS / Linux:
source venv/bin/activate

# Windows:
# venv\Scripts\activate
```

### Step 1.2 — Install Dependencies

Create `requirements.txt`:

```txt
httpx>=0.27.0
pyyaml>=6.0
flask>=3.0.0
google-generativeai>=0.7.0
python-dotenv>=1.0.0
```

Install:

```bash
pip install -r requirements.txt
```

### Step 1.3 — Store Your Gemini API Key

Create a `.env` file in the project root:

```bash
# .env
GEMINI_API_KEY=your_api_key_here
```

> **Important:** Add `.env` to your `.gitignore` immediately so the key is never committed:
> ```bash
> echo ".env" >> .gitignore
> ```

### Step 1.4 — Verify the API Key Works

```bash
python - <<'EOF'
import google.generativeai as genai
import os
from dotenv import load_dotenv

load_dotenv()
genai.configure(api_key=os.environ["GEMINI_API_KEY"])
model = genai.GenerativeModel("gemini-2.0-flash")
print(model.generate_content("Say hello in one sentence.").text)
EOF
```

Expected output: a short greeting from Gemini. If you see an error, double-check your API key.

---

## Part 2 — Build the Core Tools

All shared logic lives in `tools.py`. Both `cli.py` and `app.py` import from it.

Create `tools.py`:

```python
# tools.py
import subprocess
import os
import yaml
from pathlib import Path
from dotenv import load_dotenv
import google.generativeai as genai

# Load API key from .env
load_dotenv()
_api_key = os.environ.get("GEMINI_API_KEY", "")
if _api_key:
    genai.configure(api_key=_api_key)

DEFAULT_MODEL = "gemini-2.0-flash"


# --------------------------------------------------
# Git helpers
# --------------------------------------------------

def validate_repo(repo_path: str) -> dict:
    path = Path(repo_path).expanduser().resolve()
    if not path.exists():
        return {"valid": False, "error": f"Path does not exist: {path}"}
    if not (path / ".git").exists():
        return {"valid": False, "error": f"Not a git repository: {path}"}
    return {"valid": True, "path": str(path)}


def scan_repo(repo_path: str) -> dict:
    v = validate_repo(repo_path)
    if not v["valid"]:
        return {"error": v["error"]}

    path = Path(repo_path).expanduser().resolve()

    tree = []
    for root, dirs, files in os.walk(path):
        dirs[:] = [d for d in dirs if d not in {".git", "node_modules", "__pycache__", ".venv", "venv"}]
        level = len(Path(root).relative_to(path).parts)
        if level > 3:
            continue
        indent = "  " * level
        rel = Path(root).relative_to(path)
        if str(rel) != ".":
            tree.append(f"{indent}[dir]  {rel.name}/")
        for f in files[:10]:
            tree.append(f"{indent}  [file] {f}")

    branch  = _git(repo_path, ["rev-parse", "--abbrev-ref", "HEAD"]).strip()
    commits = _git(repo_path, ["log", "--oneline", "-5"]).strip().split("\n")
    stack   = _detect_stack(path)

    return {
        "repo_path":      str(path),
        "branch":         branch,
        "recent_commits": commits,
        "tech_stack":     stack,
        "file_tree":      "\n".join(tree),
    }


def get_changeset(repo_path: str) -> dict:
    v = validate_repo(repo_path)
    if not v["valid"]:
        return {"error": v["error"]}

    status        = _git(repo_path, ["status", "--short"])
    diff_staged   = _git(repo_path, ["diff", "--cached"])
    diff_unstaged = _git(repo_path, ["diff"])

    staged    = [l[3:] for l in status.splitlines() if l.startswith(("A ", "M ", "D ", "R "))]
    unstaged  = [l[3:] for l in status.splitlines() if l.startswith((" M", " D", " A"))]
    untracked = [l[3:] for l in status.splitlines() if l.startswith("??")]
    full_diff = (diff_staged + "\n" + diff_unstaged).strip()

    return {
        "staged_files":    staged,
        "unstaged_files":  unstaged,
        "untracked_files": untracked,
        "diff":            full_diff[:8000],
        "has_changes":     bool(staged or unstaged or untracked),
    }


def _detect_stack(path: Path) -> list:
    markers = {
        "Python":     ["requirements.txt", "pyproject.toml", "setup.py", "Pipfile"],
        "Node.js":    ["package.json"],
        "Docker":     ["Dockerfile", "docker-compose.yml", "docker-compose.yaml"],
        "Java":       ["pom.xml", "build.gradle"],
        "Go":         ["go.mod"],
        "Rust":       ["Cargo.toml"],
        "Ruby":       ["Gemfile"],
        ".NET":       ["*.csproj", "*.sln"],
        "Terraform":  ["*.tf"],
        "Kubernetes": ["*.yaml"],
    }
    all_files = {f.name for f in path.rglob("*") if f.is_file()}
    return [tech for tech, files in markers.items() if any(f in all_files for f in files)]


def _git(repo_path: str, args: list) -> str:
    try:
        r = subprocess.run(
            ["git"] + args,
            cwd=repo_path,
            capture_output=True,
            text=True,
            timeout=15,
        )
        return r.stdout
    except Exception as e:
        return f"[git error: {e}]"


# --------------------------------------------------
# Gemini helpers
# --------------------------------------------------

def gemini_chat(prompt: str, system: str = "", model: str = DEFAULT_MODEL) -> str:
    """
    Send a prompt to Gemini and return the response text.

    Parameters
    ----------
    prompt  : The user message / task.
    system  : Optional system instruction (sets the model's persona/constraints).
    model   : Gemini model name. Defaults to gemini-2.0-flash.
    """
    if not _api_key:
        return "[ERROR] GEMINI_API_KEY not set. Add it to your .env file."

    try:
        gemini_model = genai.GenerativeModel(
            model_name=model,
            system_instruction=system if system else None,
        )
        response = gemini_model.generate_content(prompt)
        return response.text.strip()
    except Exception as e:
        return f"[ERROR] Gemini API call failed: {e}"


def list_models() -> list:
    """Return all Gemini models available to your API key."""
    if not _api_key:
        return ["[ERROR] GEMINI_API_KEY not set"]
    try:
        return [m.name for m in genai.list_models()
                if "generateContent" in m.supported_generation_methods]
    except Exception as e:
        return [f"[ERROR] {e}"]


# --------------------------------------------------
# High-level actions
# --------------------------------------------------

def analyze_issues(repo_path: str, model: str = DEFAULT_MODEL) -> str:
    changeset = get_changeset(repo_path)
    if "error" in changeset:
        return f"Error: {changeset['error']}"
    if not changeset["has_changes"]:
        return "No changes detected in the repository."

    diff_text = changeset["diff"][:6000]

    system = (
        "You are a senior software engineer doing a code review. "
        "Be concise. Focus on bugs, security issues, and anti-patterns. "
        "Format your response as a short bullet list."
    )
    prompt = (
        "Review this git diff for issues:\n\n"
        + diff_text
        + "\n\nList bugs, security problems, or code quality issues. "
        "If none found, say 'No issues found.'"
    )

    return gemini_chat(prompt, system=system, model=model)


def generate_ci(repo_path: str, model: str = DEFAULT_MODEL, save: bool = False) -> str:
    scan = scan_repo(repo_path)
    if "error" in scan:
        return f"Error: {scan['error']}"

    stack     = scan.get("tech_stack", [])
    branch    = scan.get("branch", "main")
    tree      = scan.get("file_tree", "")[:2000]
    stack_str = ", ".join(stack) if stack else "Unknown"

    system = (
        "You are a DevOps engineer. Output valid GitLab CI YAML only. "
        "No explanations. No markdown fences. Just raw YAML."
    )
    prompt = (
        "Generate a .gitlab-ci.yml for:\n"
        f"- Tech stack: {stack_str}\n"
        f"- Default branch: {branch}\n"
        "- File structure:\n"
        + tree
        + "\n\nInclude stages: build, test, deploy. "
        "Use caching where applicable. "
        f"Add only:deploy rule for branch '{branch}'. "
        "Return raw YAML only."
    )

    ci_yaml = gemini_chat(prompt, system=system, model=model)

    # Strip any accidental markdown fences Gemini may add
    if ci_yaml.startswith("```"):
        ci_yaml = "\n".join(
            line for line in ci_yaml.splitlines()
            if not line.startswith("```")
        ).strip()

    # Validate; fall back to built-in template if YAML is broken
    try:
        yaml.safe_load(ci_yaml)
    except yaml.YAMLError:
        ci_yaml = _fallback_ci(stack, branch)

    if save:
        out = Path(repo_path) / ".gitlab-ci.yml"
        out.write_text(ci_yaml)
        return f"Saved to {out}\n\n{ci_yaml}"

    return ci_yaml


def summarize_branch(repo_path: str, model: str = DEFAULT_MODEL) -> str:
    log = _git(repo_path, ["log", "main..HEAD", "--oneline"])
    if not log.strip():
        return "No commits ahead of main."
    prompt = "Summarize these git commits in plain English:\n\n" + log
    return gemini_chat(prompt, model=model)


def generate_mr_description(repo_path: str, model: str = DEFAULT_MODEL) -> str:
    changeset = get_changeset(repo_path)
    diff = changeset.get("diff", "")[:4000]
    system = "You write concise, professional merge request descriptions."
    prompt = "Write a clear merge request description for this diff:\n\n" + diff
    return gemini_chat(prompt, system=system, model=model)


# --------------------------------------------------
# Fallback CI template
# --------------------------------------------------

def _fallback_ci(stack: list, branch: str) -> str:
    jobs = {}

    if "Python" in stack:
        jobs["build"] = {
            "stage": "build",
            "image": "python:3.11-slim",
            "script": ["pip install -r requirements.txt"],
            "cache": {"paths": [".cache/pip"], "key": "$CI_COMMIT_REF_SLUG"},
        }
        jobs["test"] = {
            "stage": "test",
            "image": "python:3.11-slim",
            "script": ["pip install -r requirements.txt", "pytest --tb=short"],
        }
    elif "Node.js" in stack:
        jobs["build"] = {
            "stage": "build",
            "image": "node:20-alpine",
            "script": ["npm ci"],
            "cache": {"paths": ["node_modules/"], "key": "$CI_COMMIT_REF_SLUG"},
        }
        jobs["test"] = {
            "stage": "test",
            "image": "node:20-alpine",
            "script": ["npm ci", "npm test"],
        }
    else:
        jobs["build"] = {"stage": "build", "script": ["make build || true"]}
        jobs["test"]  = {"stage": "test",  "script": ["make test || true"]}

    if "Docker" in stack:
        jobs["docker_build"] = {
            "stage": "build",
            "image": "docker:24",
            "services": ["docker:24-dind"],
            "script": ["docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA ."],
        }

    jobs["deploy"] = {
        "stage": "deploy",
        "script": ["echo Deploying"],
        "only": [branch],
        "when": "manual",
    }

    ci = {"stages": ["build", "test", "deploy"]}
    ci.update(jobs)
    return yaml.dump(ci, default_flow_style=False, sort_keys=False)
```

---

## Part 3 — Option A: Terminal CLI

Create `cli.py`:

```python
# cli.py
import sys
import tools

DIVIDER = "-" * 60

def prompt(msg: str) -> str:
    return input(f"\n{msg}: ").strip()

def show(title: str, content):
    print(f"\n{DIVIDER}")
    print(f"  {title}")
    print(DIVIDER)
    if isinstance(content, dict):
        for k, v in content.items():
            print(f"  {k}: {v}")
    else:
        print(content)

def menu():
    print("\n" + DIVIDER)
    print("  Git AI Tool — powered by Google Gemini API")
    print(DIVIDER)
    print("  1. Scan repository")
    print("  2. Show current changeset")
    print("  3. Analyze issues in changeset")
    print("  4. Generate .gitlab-ci.yml")
    print("  5. Summarize branch commits")
    print("  6. Generate MR description")
    print("  7. List available Gemini models")
    print("  0. Exit")
    return input("\n  Choose: ").strip()

def main():
    while True:
        choice = menu()

        if choice == "0":
            print("Goodbye.")
            sys.exit(0)

        elif choice == "1":
            repo = prompt("Repo path")
            result = tools.scan_repo(repo)
            if "error" in result:
                print(f"\n  Error: {result['error']}")
            else:
                show("Scan Result", {
                    "Branch":  result["branch"],
                    "Stack":   ", ".join(result["tech_stack"]) or "Unknown",
                    "Commits": "\n           ".join(result["recent_commits"]),
                })
                print(f"\n  File Tree:\n{result['file_tree']}")

        elif choice == "2":
            repo = prompt("Repo path")
            result = tools.get_changeset(repo)
            if "error" in result:
                print(f"\n  Error: {result['error']}")
            else:
                show("Changeset", {
                    "Staged":    result["staged_files"] or "none",
                    "Unstaged":  result["unstaged_files"] or "none",
                    "Untracked": result["untracked_files"] or "none",
                })
                if result["diff"]:
                    print(f"\n  Diff preview (first 1000 chars):\n{result['diff'][:1000]}")

        elif choice == "3":
            repo = prompt("Repo path")
            print("\n  Sending diff to Gemini...")
            result = tools.analyze_issues(repo)
            show("Issue Analysis", result)

        elif choice == "4":
            repo = prompt("Repo path")
            save = prompt("Save .gitlab-ci.yml to repo? (y/n)").lower() == "y"
            print("\n  Generating CI config via Gemini...")
            result = tools.generate_ci(repo, save=save)
            show("Generated .gitlab-ci.yml", result)

        elif choice == "5":
            repo = prompt("Repo path")
            print("\n  Summarizing branch commits...")
            result = tools.summarize_branch(repo)
            show("Branch Summary", result)

        elif choice == "6":
            repo = prompt("Repo path")
            print("\n  Generating MR description...")
            result = tools.generate_mr_description(repo)
            show("MR Description", result)

        elif choice == "7":
            models = tools.list_models()
            show("Available Gemini Models",
                 "\n  ".join(models) if models else "No models found")

        else:
            print("  Invalid choice. Try again.")

if __name__ == "__main__":
    main()
```

Run it:

```bash
python cli.py
```

---

## Part 4 — Option B: Browser UI

Create `app.py`:

```python
# app.py
from flask import Flask, request, jsonify, render_template_string
import tools

app = Flask(__name__)

HTML = """
<!DOCTYPE html>
<html>
<head>
  <title>Git AI Tool — Gemini</title>
  <style>
    body { font-family: monospace; max-width: 860px; margin: 40px auto; padding: 0 20px; background: #f5f5f5; }
    h1   { font-size: 1.4em; border-bottom: 2px solid #333; padding-bottom: 8px; }
    .row { display: flex; gap: 10px; margin-bottom: 12px; flex-wrap: wrap; }
    input[type=text] { flex: 1; padding: 8px; font-family: monospace; font-size: 0.95em; min-width: 300px; }
    button { padding: 8px 16px; background: #1a73e8; color: #fff; border: none;
             cursor: pointer; font-family: monospace; border-radius: 4px; }
    button:hover { background: #1557b0; }
    button.sec { background: #333; }
    button.sec:hover { background: #555; }
    pre  { background: #fff; border: 1px solid #ccc; padding: 16px; white-space: pre-wrap;
           word-break: break-word; max-height: 480px; overflow-y: auto; font-size: 0.9em; }
    .label { font-weight: bold; margin-top: 20px; margin-bottom: 4px; }
    .badge { display: inline-block; background: #e8f0fe; color: #1a73e8;
             border-radius: 3px; padding: 2px 8px; font-size: 0.8em; margin-left: 8px; }
    select { padding: 8px; font-family: monospace; border: 1px solid #ccc; }
  </style>
</head>
<body>
  <h1>Git AI Tool <span class="badge">Gemini API</span></h1>

  <div class="label">Repository path</div>
  <div class="row">
    <input type="text" id="repo" placeholder="/absolute/path/to/your/repo" />
  </div>

  <div class="label">Model</div>
  <div class="row">
    <select id="model">
      <option value="gemini-2.0-flash" selected>gemini-2.0-flash (recommended)</option>
      <option value="gemini-1.5-flash">gemini-1.5-flash</option>
      <option value="gemini-1.5-pro">gemini-1.5-pro</option>
    </select>
  </div>

  <div class="row" style="margin-top:12px">
    <button onclick="call('scan')" class="sec">Scan Repo</button>
    <button onclick="call('changeset')" class="sec">Show Changeset</button>
    <button onclick="call('analyze')">Analyze Issues</button>
    <button onclick="call('generate')">Generate CI</button>
    <button onclick="call('generate_save')">Generate CI + Save</button>
    <button onclick="call('mr_description')">MR Description</button>
  </div>

  <div class="label">Output</div>
  <pre id="output">Results will appear here...</pre>

  <script>
    async function call(action) {
      const repo  = document.getElementById('repo').value.trim();
      const model = document.getElementById('model').value;
      document.getElementById('output').textContent = 'Working... (Gemini is thinking)';
      try {
        const r = await fetch('/api/' + action, {
          method:  'POST',
          headers: {'Content-Type': 'application/json'},
          body:    JSON.stringify({ repo_path: repo, model: model })
        });
        const data = await r.json();
        document.getElementById('output').textContent =
          typeof data.result === 'object'
            ? JSON.stringify(data.result, null, 2)
            : data.result;
      } catch(e) {
        document.getElementById('output').textContent = 'Request failed: ' + e;
      }
    }
  </script>
</body>
</html>
"""

def get_params():
    body = request.json or {}
    return body.get("repo_path", ""), body.get("model", tools.DEFAULT_MODEL)

@app.route("/")
def index():
    return render_template_string(HTML)

@app.route("/api/scan", methods=["POST"])
def api_scan():
    repo, _ = get_params()
    return jsonify(result=tools.scan_repo(repo))

@app.route("/api/changeset", methods=["POST"])
def api_changeset():
    repo, _ = get_params()
    return jsonify(result=tools.get_changeset(repo))

@app.route("/api/analyze", methods=["POST"])
def api_analyze():
    repo, model = get_params()
    return jsonify(result=tools.analyze_issues(repo, model=model))

@app.route("/api/generate", methods=["POST"])
def api_generate():
    repo, model = get_params()
    return jsonify(result=tools.generate_ci(repo, model=model, save=False))

@app.route("/api/generate_save", methods=["POST"])
def api_generate_save():
    repo, model = get_params()
    return jsonify(result=tools.generate_ci(repo, model=model, save=True))

@app.route("/api/mr_description", methods=["POST"])
def api_mr_description():
    repo, model = get_params()
    return jsonify(result=tools.generate_mr_description(repo, model=model))

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

Run it:

```bash
python app.py
```

Then open: http://localhost:5000

---

## Gemini Model Reference

| Model | Best For | Notes |
|-------|----------|-------|
| `gemini-2.0-flash` | General use — **recommended** | Fast, free tier, strong at code |
| `gemini-1.5-flash` | Lighter workloads | Very fast, lower cost |
| `gemini-1.5-pro` | Complex reasoning | Larger context window, slower |

> Switch the model in the browser UI dropdown or pass `model=` to any `tools.py` function from the CLI.

### Free Tier Rate Limits (as of April 2026)

| Model | Requests/min | Tokens/min |
|-------|-------------|------------|
| gemini-2.0-flash | 15 | 1,000,000 |
| gemini-1.5-flash | 15 | 1,000,000 |

This is more than enough for workshop use. No billing setup required.

---

## Part 5 — Hands-On Exercises

Work through these in order using whichever interface you chose.

### Exercise 1 — Scan a Repository

CLI:

```
Choose: 1
Repo path: /path/to/any/local/git/repo
```

Browser: paste the path, click **Scan Repo**.

Expected: branch name, tech stack, and recent commits appear.

---

### Exercise 2 — Inspect the Changeset

Make a small edit to any tracked file, then:

CLI:

```
Choose: 2
Repo path: /path/to/repo
```

Browser: click **Show Changeset**.

Expected: staged/unstaged/untracked files listed, diff shown.

---

### Exercise 3 — AI Code Review

Stage your changes first:

```bash
git add .
```

Then:

CLI:

```
Choose: 3
Repo path: /path/to/repo
```

Browser: click **Analyze Issues**.

Expected: Gemini returns a bullet list of any code problems it found.

---

### Exercise 4 — Generate GitLab CI

CLI:

```
Choose: 4
Repo path: /path/to/repo
Save .gitlab-ci.yml to repo? (y/n): y
```

Browser: click **Generate CI + Save**.

Expected: a `.gitlab-ci.yml` file appears in the repo root.

---

### Exercise 5 — Full Workflow (Challenge)

Run all four steps back-to-back on the same repo: Scan → Changeset → Analyze → Generate CI. Compare Gemini's output using `gemini-2.0-flash` vs `gemini-1.5-pro` on the same prompt.

---

## Part 6 — Troubleshooting

### API key not found

```bash
# Confirm .env is in the project root and contains the key
cat .env

# Test loading manually
python -c "from dotenv import load_dotenv; import os; load_dotenv(); print(os.environ.get('GEMINI_API_KEY', 'NOT SET'))"
```

---

### Gemini returns a 403 or quota error

```bash
# Verify the key is valid and has generateContent permission
python -c "
import google.generativeai as genai, os
from dotenv import load_dotenv
load_dotenv()
genai.configure(api_key=os.environ['GEMINI_API_KEY'])
print([m.name for m in genai.list_models()][:5])
"
```

If you hit rate limits, wait 60 seconds and retry. The free tier resets per minute.

---

### Gemini returns YAML with markdown fences

`tools.py` strips ` ``` ` fences automatically. If the output is still invalid, the tool falls back to the built-in `_fallback_ci` template. You can also validate manually:

```bash
pip install yamllint
yamllint .gitlab-ci.yml
```

---

### Git errors

```bash
# Confirm the path is a valid repo
git -C /your/repo/path status

# Test tools.py directly
python -c "import tools; print(tools.scan_repo('/your/repo/path'))"
```

---

### Flask not found

```bash
pip install flask
```

---

## Part 7 — Extend the Tool (Bonus)

Add these to `tools.py` and wire them into `cli.py` or `app.py`:

```python
def explain_commit(repo_path: str, sha: str, model: str = DEFAULT_MODEL) -> str:
    """Explain what a specific commit changed, in plain English."""
    diff = _git(repo_path, ["show", sha, "--stat", "--patch"])
    return gemini_chat(
        f"Explain this git commit in plain English:\n\n{diff[:5000]}",
        system="You explain code changes clearly for a non-technical audience.",
        model=model,
    )


def suggest_commit_message(repo_path: str, model: str = DEFAULT_MODEL) -> str:
    """Suggest a Conventional Commits message for staged changes."""
    changeset = get_changeset(repo_path)
    diff = changeset.get("diff", "")[:4000]
    return gemini_chat(
        f"Write a Conventional Commits message for this diff:\n\n{diff}",
        system=(
            "Output a single commit message in Conventional Commits format. "
            "Example: 'feat(auth): add OAuth2 login support'. "
            "No explanation. Just the message."
        ),
        model=model,
    )
```

---

## Ollama → Gemini Migration Reference

If you are converting an existing project that used Ollama, here is the change summary:

| Before (Ollama) | After (Gemini) |
|-----------------|----------------|
| `import httpx` + manual POST | `import google.generativeai as genai` |
| `ollama serve &` (local process) | API key in `.env` (cloud) |
| `http://localhost:11434/api/generate` | `generativelanguage.googleapis.com` (handled by SDK) |
| `"model": "llama3.2:1b"` | `model_name="gemini-2.0-flash"` |
| `ollama pull llama3.2:1b` (1.3 GB download) | No download — API call |
| Runs offline | Requires internet connection |
| Free, unlimited | Free tier: 15 req/min |

The `system` parameter maps directly — both accept a system instruction string. The `prompt` parameter is identical. Only the transport layer changes.

---

## Reference Links

| Resource | URL |
|----------|-----|
| Gemini API docs | https://ai.google.dev/gemini-api/docs |
| Get a free API key | https://aistudio.google.com/apikey |
| google-generativeai SDK | https://pypi.org/project/google-generativeai |
| Gemini model list | https://ai.google.dev/gemini-api/docs/models/gemini |
| GitLab CI YAML reference | https://docs.gitlab.com/ee/ci/yaml |

---

## Workshop Summary

By the end of this workshop you have:

- [x] Stored a Gemini API key securely in `.env`
- [x] Built shared git + Gemini logic in `tools.py`
- [x] Built an interactive terminal CLI in `cli.py`
- [x] Built a browser UI in `app.py` using Flask with a model selector
- [x] Used `gemini-2.0-flash` to review code and generate CI config
- [x] Run a full scan → changeset → analysis → CI generation pipeline

---

*Workshop — Git AI Tool + Google Gemini API · April 2026*
