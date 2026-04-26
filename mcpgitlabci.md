# Workshop: Build a Git-Aware AI Tool with Ollama
### Scan Repos · Detect Changesets · Auto-Generate `.gitlab-ci.yml`

No Claude Desktop required. Everything runs from the terminal or browser.

---

## Prerequisites Checklist

| Tool | Version | Check Command |
|------|---------|---------------|
| Python | >= 3.11 | `python --version` |
| Git | >= 2.x | `git --version` |
| Ollama | Latest | `ollama --version` |
| pip | Latest | `pip --version` |

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
           |    Ollama (llama3.2:1b) |
           |    localhost:11434      |
           +-------------------------+
```

Two ways to run:
- **Option A** — `cli.py` : interactive terminal menu
- **Option B** — `app.py` : browser UI using Flask

---

## Project Structure

```
git-ai-tool/
├── tools.py           <- Core logic (git + Ollama)
├── cli.py             <- Terminal interface (Option A)
├── app.py             <- Browser UI with Flask (Option B)
├── requirements.txt   <- Dependencies
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
# macOS/Linux:
source venv/bin/activate

```

### Step 1.2 — Install Dependencies

Create `requirements.txt`:

```txt
httpx>=0.27.0
pyyaml>=6.0
flask>=3.0.0
```

Install:

```bash
pip install -r requirements.txt
```

### Step 1.3 — Pull the Ollama Model

```bash
# Start Ollama
ollama serve &

# Pull the model (approx 1.3GB)
ollama pull llama3.2:1b

# Verify
ollama run llama3.2:1b "Say hello"
```

---

## Part 2 — Build the Core Tools

All shared logic lives in `tools.py`. Both `cli.py` and `app.py` import from it.

Create `tools.py`:

```python
# tools.py
import subprocess
import os
import httpx
import yaml
from pathlib import Path

OLLAMA_BASE_URL = "http://localhost:11434"
DEFAULT_MODEL = "llama3.2:1b"


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
        "staged_files":   staged,
        "unstaged_files": unstaged,
        "untracked_files": untracked,
        "diff":           full_diff[:8000],
        "has_changes":    bool(staged or unstaged or untracked),
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
# Ollama helpers
# --------------------------------------------------

def ollama_chat(prompt: str, system: str = "", model: str = DEFAULT_MODEL) -> str:
    payload = {"model": model, "prompt": prompt, "stream": False}
    if system:
        payload["system"] = system
    try:
        with httpx.Client(timeout=120.0) as client:
            r = client.post(f"{OLLAMA_BASE_URL}/api/generate", json=payload)
            r.raise_for_status()
            return r.json().get("response", "").strip()
    except httpx.ConnectError:
        return "[ERROR] Ollama is not running. Run: ollama serve"
    except Exception as e:
        return f"[ERROR] {e}"


def list_models() -> list:
    try:
        with httpx.Client(timeout=10.0) as client:
            r = client.get(f"{OLLAMA_BASE_URL}/api/tags")
            r.raise_for_status()
            return [m["name"] for m in r.json().get("models", [])]
    except Exception:
        return []


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
        "You are a senior software engineer doing code review. "
        "Be concise. Focus on bugs, security issues, and anti-patterns. "
        "Format your response as a short bullet list."
    )
    prompt = (
        "Review this git diff for issues:\n\n"
        + diff_text
        + "\n\nList bugs, security problems, or code quality issues. "
        "If none, say 'No issues found.'"
    )

    return ollama_chat(prompt, system=system, model=model)


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

    ci_yaml = ollama_chat(prompt, system=system, model=model)

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
    return ollama_chat(prompt, model=model)


def generate_mr_description(repo_path: str, model: str = DEFAULT_MODEL) -> str:
    changeset = get_changeset(repo_path)
    diff = changeset.get("diff", "")[:4000]
    system = "You write concise, professional merge request descriptions."
    prompt = "Write a clear merge request description for this diff:\n\n" + diff
    return ollama_chat(prompt, system=system, model=model)


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
    print("  Git AI Tool — powered by Ollama llama3.2:1b")
    print(DIVIDER)
    print("  1. Scan repository")
    print("  2. Show current changeset")
    print("  3. Analyze issues in changeset")
    print("  4. Generate .gitlab-ci.yml")
    print("  5. List Ollama models")
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
                    "Branch":     result["branch"],
                    "Stack":      ", ".join(result["tech_stack"]) or "Unknown",
                    "Commits":    "\n           ".join(result["recent_commits"]),
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
            print("\n  Sending diff to Ollama... (may take a moment)")
            result = tools.analyze_issues(repo)
            show("Issue Analysis", result)

        elif choice == "4":
            repo = prompt("Repo path")
            save = prompt("Save .gitlab-ci.yml to repo? (y/n)").lower() == "y"
            print("\n  Generating CI config via Ollama...")
            result = tools.generate_ci(repo, save=save)
            show("Generated .gitlab-ci.yml", result)

        elif choice == "5":
            models = tools.list_models()
            show("Available Ollama Models",
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
  <title>Git AI Tool</title>
  <style>
    body { font-family: monospace; max-width: 860px; margin: 40px auto; padding: 0 20px; background: #f5f5f5; }
    h1   { font-size: 1.4em; border-bottom: 2px solid #333; padding-bottom: 8px; }
    .row { display: flex; gap: 10px; margin-bottom: 12px; }
    input[type=text] { flex: 1; padding: 8px; font-family: monospace; font-size: 0.95em; }
    button { padding: 8px 18px; background: #333; color: #fff; border: none; cursor: pointer; font-family: monospace; }
    button:hover { background: #555; }
    pre  { background: #fff; border: 1px solid #ccc; padding: 16px; white-space: pre-wrap; word-break: break-word;
           max-height: 480px; overflow-y: auto; font-size: 0.9em; }
    .label { font-weight: bold; margin-top: 20px; margin-bottom: 4px; }
    .err { color: red; }
    select { padding: 8px; font-family: monospace; }
  </style>
</head>
<body>
  <h1>Git AI Tool — llama3.2:1b via Ollama</h1>

  <div class="label">Repository path</div>
  <div class="row">
    <input type="text" id="repo" placeholder="/absolute/path/to/your/repo" />
  </div>

  <div class="row">
    <button onclick="call('scan')">Scan Repo</button>
    <button onclick="call('changeset')">Show Changeset</button>
    <button onclick="call('analyze')">Analyze Issues</button>
    <button onclick="call('generate')">Generate CI</button>
    <button onclick="call('generate_save')">Generate CI + Save</button>
  </div>

  <div class="label">Output</div>
  <pre id="output">Results will appear here...</pre>

  <script>
    async function call(action) {
      const repo = document.getElementById('repo').value.trim();
      document.getElementById('output').textContent = 'Working...';
      try {
        const r = await fetch('/api/' + action, {
          method: 'POST',
          headers: {'Content-Type': 'application/json'},
          body: JSON.stringify({ repo_path: repo })
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

@app.route("/")
def index():
    return render_template_string(HTML)

@app.route("/api/scan", methods=["POST"])
def api_scan():
    repo = request.json.get("repo_path", "")
    return jsonify(result=tools.scan_repo(repo))

@app.route("/api/changeset", methods=["POST"])
def api_changeset():
    repo = request.json.get("repo_path", "")
    return jsonify(result=tools.get_changeset(repo))

@app.route("/api/analyze", methods=["POST"])
def api_analyze():
    repo = request.json.get("repo_path", "")
    return jsonify(result=tools.analyze_issues(repo))

@app.route("/api/generate", methods=["POST"])
def api_generate():
    repo = request.json.get("repo_path", "")
    return jsonify(result=tools.generate_ci(repo, save=False))

@app.route("/api/generate_save", methods=["POST"])
def api_generate_save():
    repo = request.json.get("repo_path", "")
    return jsonify(result=tools.generate_ci(repo, save=True))

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

Run it:

```bash
python app.py
```

Then open: [http://localhost:5000](http://localhost:5000)

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

Expected: Ollama returns a bullet list of any code problems found.

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

Run all four steps back-to-back on the same repo and review the complete output.

---

## Part 6 — Troubleshooting

### Ollama not responding

```bash
# Check it is running
curl http://localhost:11434/api/tags

# Start it if not
ollama serve

# Confirm the model is downloaded
ollama list
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

### YAML invalid in generated CI

```bash
pip install yamllint
yamllint .gitlab-ci.yml
```

If Ollama returns bad YAML the tool falls back to a built-in template automatically.

---

## Part 7 — Extend the Tool (Bonus)

Add these functions to `tools.py` and wire them into `cli.py` or `app.py`:

```python
def summarize_branch(repo_path: str, model: str = DEFAULT_MODEL) -> str:
    """Summarize all commits on the current branch vs main."""
    log = _git(repo_path, ["log", "main..HEAD", "--oneline"])
    if not log.strip():
        return "No commits ahead of main."
    return ollama_chat(f"Summarize these git commits:\n{log}", model=model)


def generate_mr_description(repo_path: str, model: str = DEFAULT_MODEL) -> str:
    """Auto-generate a merge request description from the changeset."""
    changeset = get_changeset(repo_path)
    diff = changeset.get("diff", "")
    return ollama_chat(
        f"Write a clear merge request description for this diff:\n{diff[:4000]}",
        system="You write concise, professional merge request descriptions.",
        model=model
    )
```

---

## Reference

| Resource | URL |
|----------|-----|
| Ollama API docs | https://github.com/ollama/ollama/blob/main/docs/api.md |
| Ollama model library | https://ollama.com/library |
| GitLab CI reference | https://docs.gitlab.com/ee/ci/yaml/ |
| MCP Python SDK | https://github.com/modelcontextprotocol/python-sdk |

---

## Workshop Summary

By the end of this workshop you have:

- [x] Built shared git + Ollama logic in `tools.py`
- [x] Built an interactive terminal CLI in `cli.py`
- [x] Built a browser UI in `app.py` using Flask
- [x] Used `llama3.2:1b` locally to review code and generate CI config
- [x] Run a full scan -> changeset -> analysis -> CI generation pipeline

---

*Workshop — Git AI Tool + Ollama · April 2026*
