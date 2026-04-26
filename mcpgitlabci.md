# Workshop: Build a Git-Aware MCP Server with Ollama
### Scan Repos · Detect Changesets · Auto-Generate `.gitlab-ci.yml`

---

## Prerequisites Checklist

Before the workshop, ensure you have the following installed:

| Tool | Version | Check Command |
|------|---------|---------------|
| python3 | >= 3.11 | `python3 --version` |
| Git | >= 2.x | `git --version` |
| Ollama | Latest | `ollama --version` |
| pip | Latest | `pip --version` |
| Claude Desktop | Latest | — |

> **Tip:** All exercises are run from your terminal. Keep it open throughout.

---

## What We're Building

```
+-----------------------------------------------------+
|                   Claude Desktop                     |
|                  (MCP Client / UI)                   |
+----------------------+------------------------------+
                       |  MCP Protocol (stdio)
+----------------------v------------------------------+
|              Git-Aware MCP Server                    |
|  +--------------------------------------------------+|
|  |  Tools:                                          ||
|  |  - scan_repo       -> repo structure             ||
|  |  - get_changeset   -> git diff / status          ||
|  |  - analyze_issues  -> Ollama LLM review          ||
|  |  - generate_ci     -> .gitlab-ci.yml output      ||
|  +--------------------------------------------------+|
+----------------------+------------------------------+
                       |  HTTP (localhost:11434)
+----------------------v------------------------------+
|                  Ollama (Local LLM)                  |
|               Model: llama3.2:1b                     |
+-----------------------------------------------------+
```

---

## Project Structure

```
git-mcp-server/
├── server.py          <- Main MCP server
├── git_tools.py       <- Git operations
├── ollama_client.py   <- Ollama LLM calls
├── ci_generator.py    <- GitLab CI YAML builder
├── requirements.txt   <- Dependencies
└── README.md
```

---

## Part 1 — Setup and Installation

### Step 1.1 — Create the Project

```bash
mkdir git-mcp-server
cd git-mcp-server
python3 -m venv venv

# Activate the virtual environment
# On macOS/Linux:
source venv/bin/activate
# On Windows:
venv\Scripts\activate
```

### Step 1.2 — Install Dependencies

Create `requirements.txt`:

```txt
mcp>=1.0.0
gitpython3>=3.1.40
httpx>=0.27.0
pyyaml>=6.0
```

Install them:

```bash
pip install -r requirements.txt
```

### Step 1.3 — Pull the Ollama Model

```bash
# Start Ollama (if not running as a service)
ollama serve &

# Pull llama3.2:1b (lightweight, fast, ~1.3GB)
ollama pull llama3.2:1b

# Verify it works
ollama run llama3.2:1b "Say hello"
```

---

## Part 2 — Build the Git Tools Module

Create `git_tools.py`:

```python3
# git_tools.py
import subprocess
import os
from pathlib import Path


def validate_repo(repo_path: str) -> dict:
    """Check if a path is a valid git repository."""
    path = Path(repo_path).expanduser().resolve()
    if not path.exists():
        return {"valid": False, "error": f"Path does not exist: {path}"}
    git_dir = path / ".git"
    if not git_dir.exists():
        return {"valid": False, "error": f"Not a git repository: {path}"}
    return {"valid": True, "path": str(path)}


def scan_repo(repo_path: str) -> dict:
    """
    Scan a git repository and return:
    - file tree (top 3 levels)
    - branch name
    - last 5 commit messages
    - detected tech stack
    """
    validation = validate_repo(repo_path)
    if not validation["valid"]:
        return {"error": validation["error"]}

    path = Path(repo_path).expanduser().resolve()

    # File tree (3 levels deep, exclude .git)
    tree = []
    for root, dirs, files in os.walk(path):
        dirs[:] = [d for d in dirs if d not in {".git", "node_modules", "__pycache__", ".venv", "venv"}]
        level = len(Path(root).relative_to(path).parts)
        if level > 3:
            continue
        indent = "  " * level
        rel = Path(root).relative_to(path)
        if str(rel) != ".":
            tree.append(f"{indent}[dir] {rel.name}/")
        for f in files[:10]:  # limit files per dir
            tree.append(f"{indent}  [file] {f}")

    # Current branch
    branch = _run_git(repo_path, ["rev-parse", "--abbrev-ref", "HEAD"])

    # Recent commits
    commits = _run_git(repo_path, ["log", "--oneline", "-5"])

    # Tech stack detection
    stack = detect_stack(path)

    return {
        "repo_path": str(path),
        "branch": branch.strip(),
        "recent_commits": commits.strip().split("\n"),
        "tech_stack": stack,
        "file_tree": "\n".join(tree),
    }


def get_changeset(repo_path: str) -> dict:
    """
    Return the current changeset:
    - staged files
    - unstaged files
    - untracked files
    - full diff (staged + unstaged)
    """
    validation = validate_repo(repo_path)
    if not validation["valid"]:
        return {"error": validation["error"]}

    status = _run_git(repo_path, ["status", "--short"])
    diff_staged = _run_git(repo_path, ["diff", "--cached"])
    diff_unstaged = _run_git(repo_path, ["diff"])

    staged = [l[3:] for l in status.splitlines() if l.startswith(("A ", "M ", "D ", "R "))]
    unstaged = [l[3:] for l in status.splitlines() if l.startswith((" M", " D", " A"))]
    untracked = [l[3:] for l in status.splitlines() if l.startswith("??")]

    full_diff = (diff_staged + "\n" + diff_unstaged).strip()

    return {
        "staged_files": staged,
        "unstaged_files": unstaged,
        "untracked_files": untracked,
        "diff": full_diff[:8000],  # cap to 8k chars for LLM context
        "has_changes": bool(staged or unstaged or untracked),
    }


def detect_stack(path: Path) -> list[str]:
    """Detect the technology stack from project files."""
    stack = []
    markers = {
        "python3": ["requirements.txt", "pyproject.toml", "setup.py", "Pipfile"],
        "Node.js": ["package.json"],
        "Docker": ["Dockerfile", "docker-compose.yml", "docker-compose.yaml"],
        "Java": ["pom.xml", "build.gradle"],
        "Go": ["go.mod"],
        "Rust": ["Cargo.toml"],
        "Ruby": ["Gemfile"],
        ".NET": ["*.csproj", "*.sln"],
        "Terraform": ["*.tf", "main.tf"],
        "Kubernetes": ["*.yaml"],
    }
    all_files = {f.name for f in path.rglob("*") if f.is_file()}
    for tech, files in markers.items():
        if any(f in all_files for f in files):
            stack.append(tech)
    return stack


def _run_git(repo_path: str, args: list[str]) -> str:
    """Run a git command and return stdout."""
    try:
        result = subprocess.run(
            ["git"] + args,
            cwd=repo_path,
            capture_output=True,
            text=True,
            timeout=15,
        )
        return result.stdout
    except Exception as e:
        return f"[git error: {e}]"
```

---

## Part 3 — Build the Ollama Client

Create `ollama_client.py`:

```python3
# ollama_client.py
import httpx

OLLAMA_BASE_URL = "http://localhost:11434"
DEFAULT_MODEL = "llama3.2:1b"


def chat(prompt: str, model: str = DEFAULT_MODEL, system: str = "") -> str:
    """
    Send a prompt to Ollama and return the response text.
    Uses the /api/generate endpoint (single-turn).
    """
    payload = {
        "model": model,
        "prompt": prompt,
        "stream": False,
    }
    if system:
        payload["system"] = system

    try:
        with httpx.Client(timeout=120.0) as client:
            response = client.post(
                f"{OLLAMA_BASE_URL}/api/generate",
                json=payload,
            )
            response.raise_for_status()
            return response.json().get("response", "").strip()
    except httpx.ConnectError:
        return "[ERROR] Ollama is not running. Start it with: ollama serve"
    except Exception as e:
        return f"[ERROR] Ollama request failed: {e}"


def list_models() -> list[str]:
    """Return available Ollama models."""
    try:
        with httpx.Client(timeout=10.0) as client:
            r = client.get(f"{OLLAMA_BASE_URL}/api/tags")
            r.raise_for_status()
            return [m["name"] for m in r.json().get("models", [])]
    except Exception:
        return []


def analyze_diff(diff: str, model: str = DEFAULT_MODEL) -> str:
    """Ask Ollama to review a git diff for issues."""
    if not diff or not diff.strip():
        return "No changes to analyze."

    system = (
        "You are a senior software engineer doing code review. "
        "Be concise. Focus on bugs, security issues, and anti-patterns. "
        "Format your response as a short bullet list."
    )
    prompt = f"""Review the following git diff and identify any issues:

```diff
{diff[:6000]}
```

List issues found (bugs, security, style, logic). If none, say 'No issues found.'"""

    return chat(prompt, model=model, system=system)
```

---

## Part 4 — Build the CI Generator

Create `ci_generator.py`:

```python3
# ci_generator.py
import yaml
from ollama_client import chat

DEFAULT_MODEL = "llama3.2:1b"


def generate_gitlab_ci(scan_result: dict, model: str = DEFAULT_MODEL) -> str:
    """
    Use Ollama to generate a .gitlab-ci.yml based on the scanned repo.
    Falls back to a template if Ollama fails.
    """
    stack = scan_result.get("tech_stack", [])
    branch = scan_result.get("branch", "main")
    tree_snippet = scan_result.get("file_tree", "")[:2000]

    system = (
        "You are a DevOps engineer. Generate valid GitLab CI YAML only. "
        "No explanations. No markdown fences. Just raw YAML."
    )

    prompt = f"""Generate a .gitlab-ci.yml for a project with:
- Tech stack: {', '.join(stack) if stack else 'Unknown'}
- Default branch: {branch}
- File structure (partial):
{tree_snippet}

Requirements:
- Include stages: build, test, deploy
- Add appropriate jobs for the detected stack
- Use caching where applicable
- Include only:deploy for branch '{branch}'
- Add a security scan stage if Docker is detected

Return only valid YAML."""

    llm_output = chat(prompt, model=model, system=system)

    # Validate the YAML is parseable
    try:
        yaml.safe_load(llm_output)
        return llm_output
    except yaml.YAMLError:
        # Fall back to template
        return _fallback_ci(stack, branch)


def _fallback_ci(stack: list[str], branch: str) -> str:
    """Return a basic GitLab CI template based on detected stack."""
    stages = ["build", "test", "deploy"]
    jobs = {}

    if "python3" in stack:
        jobs["build"] = {
            "stage": "build",
            "image": "python3:3.11-slim",
            "script": ["pip install -r requirements.txt"],
            "cache": {"paths": [".cache/pip"], "key": "$CI_COMMIT_REF_SLUG"},
        }
        jobs["test"] = {
            "stage": "test",
            "image": "python3:3.11-slim",
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
        jobs["build"] = {"stage": "build", "script": ["echo 'Build step'", "make build || true"]}
        jobs["test"] = {"stage": "test", "script": ["echo 'Test step'", "make test || true"]}

    if "Docker" in stack:
        jobs["docker_build"] = {
            "stage": "build",
            "image": "docker:24",
            "services": ["docker:24-dind"],
            "script": ["docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA ."],
        }

    jobs["deploy_production"] = {
        "stage": "deploy",
        "script": ["echo 'Deploy to production'"],
        "only": [branch],
        "when": "manual",
    }

    ci = {"stages": stages}
    ci.update(jobs)
    return yaml.dump(ci, default_flow_style=False, sort_keys=False)
```

---

## Part 5 — Build the MCP Server

Create `server.py` (the main entry point):

```python3
# server.py
import asyncio
import json
from mcp.server import Server
from mcp.server.stdio import stdio_server
from mcp.types import Tool, TextContent

import git_tools
import ollama_client
import ci_generator

app = Server("git-mcp-server")

# --------------------------------------------------
#  Tool Definitions
# --------------------------------------------------

@app.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="scan_repo",
            description=(
                "Scan a local git repository. Returns branch, recent commits, "
                "file structure, and detected tech stack."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    }
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="get_changeset",
            description=(
                "Get the current git changeset: staged, unstaged, and untracked files, "
                "plus the full diff content."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    }
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="analyze_issues",
            description=(
                "Use Ollama (local LLM) to analyze the current git diff for bugs, "
                "security issues, or code quality problems."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    },
                    "model": {
                        "type": "string",
                        "description": "Ollama model to use (default: llama3.2:1b)",
                        "default": "llama3.2:1b",
                    },
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="generate_ci",
            description=(
                "Generate a .gitlab-ci.yml file for the repository using Ollama. "
                "Scans the repo first, then uses the LLM to produce tailored CI config."
            ),
            inputSchema={
                "type": "object",
                "properties": {
                    "repo_path": {
                        "type": "string",
                        "description": "Absolute path to the local git repository",
                    },
                    "model": {
                        "type": "string",
                        "description": "Ollama model to use (default: llama3.2:1b)",
                        "default": "llama3.2:1b",
                    },
                    "save_to_repo": {
                        "type": "boolean",
                        "description": "Write .gitlab-ci.yml to the repo root (default: false)",
                        "default": False,
                    },
                },
                "required": ["repo_path"],
            },
        ),
        Tool(
            name="list_ollama_models",
            description="List all available Ollama models installed on this machine.",
            inputSchema={"type": "object", "properties": {}},
        ),
    ]


# --------------------------------------------------
#  Tool Handlers
# --------------------------------------------------

@app.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:

    if name == "scan_repo":
        result = git_tools.scan_repo(arguments["repo_path"])
        return [TextContent(type="text", text=json.dumps(result, indent=2))]

    elif name == "get_changeset":
        result = git_tools.get_changeset(arguments["repo_path"])
        return [TextContent(type="text", text=json.dumps(result, indent=2))]

    elif name == "analyze_issues":
        changeset = git_tools.get_changeset(arguments["repo_path"])
        if "error" in changeset:
            return [TextContent(type="text", text=f"Error: {changeset['error']}")]
        if not changeset.get("has_changes"):
            return [TextContent(type="text", text="No changes detected in the repository.")]
        diff = changeset.get("diff", "")
        model = arguments.get("model", "llama3.2:1b")
        analysis = ollama_client.analyze_diff(diff, model=model)
        return [TextContent(type="text", text=analysis)]

    elif name == "generate_ci":
        repo_path = arguments["repo_path"]
        model = arguments.get("model", "llama3.2:1b")
        save = arguments.get("save_to_repo", False)

        scan = git_tools.scan_repo(repo_path)
        if "error" in scan:
            return [TextContent(type="text", text=f"Error: {scan['error']}")]

        ci_yaml = ci_generator.generate_gitlab_ci(scan, model=model)

        if save:
            output_path = f"{repo_path}/.gitlab-ci.yml"
            with open(output_path, "w") as f:
                f.write(ci_yaml)
            return [TextContent(
                type="text",
                text=f"Saved to {output_path}\n\n```yaml\n{ci_yaml}\n```",
            )]

        return [TextContent(type="text", text=f"```yaml\n{ci_yaml}\n```")]

    elif name == "list_ollama_models":
        models = ollama_client.list_models()
        if not models:
            return [TextContent(type="text", text="No Ollama models found. Run: ollama pull llama3.2:1b")]
        return [TextContent(type="text", text="Available models:\n" + "\n".join(f"- {m}" for m in models))]

    else:
        return [TextContent(type="text", text=f"Unknown tool: {name}")]


# --------------------------------------------------
#  Entry Point
# --------------------------------------------------

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write, app.create_initialization_options())


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Part 6 — Register with Claude Desktop

### Step 6.1 — Find the Config File

| OS | Location |
|----|----------|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

### Step 6.2 — Add the MCP Server

Open the config file and add your server:

```json
{
  "mcpServers": {
    "git-mcp": {
      "command": "/ABSOLUTE/PATH/TO/git-mcp-server/venv/bin/python3",
      "args": ["/ABSOLUTE/PATH/TO/git-mcp-server/server.py"],
      "env": {}
    }
  }
}
```

> **Note:** Replace `/ABSOLUTE/PATH/TO/` with your actual path.
>
> macOS example:
> `"command": "/Users/yourname/projects/git-mcp-server/venv/bin/python3"`

### Step 6.3 — Restart Claude Desktop

Quit and reopen Claude Desktop. You should see the tools icon with your 5 new tools listed.

---

## Part 7 — Hands-On Exercises

### Exercise 1 — Scan a Repository

In Claude Desktop, try:

```
Scan the repository at ~/projects/my-app and tell me what stack it uses.
```

Expected: Claude calls `scan_repo`, returns branch, commits, and stack.

---

### Exercise 2 — Inspect the Changeset

Make a change to a file in any git repo, then ask:

```
What files have changed in ~/projects/my-app? Any staged changes?
```

Expected: Claude calls `get_changeset`, lists staged/unstaged files and shows the diff.

---

### Exercise 3 — AI Code Review

Stage some changes (`git add .`) then ask:

```
Analyze the current changeset in ~/projects/my-app for any issues using llama3.2:1b.
```

Expected: Ollama reviews the diff and returns a bullet-point list of issues.

---

### Exercise 4 — Generate GitLab CI

```
Generate a .gitlab-ci.yml for ~/projects/my-app and save it to the repo.
```

Expected: Claude scans, calls Ollama, returns valid YAML, optionally writes the file.

---

### Exercise 5 — Full Workflow (Challenge)

```
1. Scan ~/projects/my-app
2. Check what changed since last commit
3. Review the changes for issues using llama3.2:1b
4. Generate a suitable .gitlab-ci.yml
```

---

## Part 8 — Troubleshooting

### MCP server not appearing in Claude

```bash
# Test the server manually
cd git-mcp-server
source venv/bin/activate
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' | python3 server.py
```

You should see a JSON response with your tools listed.

---

### Ollama not responding

```bash
# Check if Ollama is running
curl http://localhost:11434/api/tags

# Start Ollama
ollama serve

# Verify model is downloaded
ollama list
```

---

### Git tool errors

```bash
# Confirm the path is a git repo
git -C /your/repo/path status

# Confirm python3 can run it
python3 -c "import git_tools; print(git_tools.scan_repo('/your/repo/path'))"
```

---

### YAML validation error in CI output

```bash
# Install yamllint and validate
pip install yamllint
yamllint .gitlab-ci.yml
```

---

## Part 9 — Extend the Server (Bonus)

Ideas to add more tools:

```python3
# 1. Blame tool — who changed a file last?
Tool(name="git_blame", description="Show who last modified each line of a file")

# 2. Summary tool — summarize all commits in a branch
Tool(name="summarize_branch", description="Summarize all commits since branching from main")

# 3. PR description tool — auto-generate a merge request description
Tool(name="generate_mr_description", description="Generate MR description from the changeset")

# 4. Dependency audit — check for vulnerable packages
Tool(name="audit_dependencies", description="Check requirements.txt or package.json for known vulnerabilities")
```

---

## Reference

| Resource | URL |
|----------|-----|
| MCP python3 SDK | https://github.com/modelcontextprotocol/python3-sdk |
| MCP Specification | https://modelcontextprotocol.io/docs |
| Ollama API Docs | https://github.com/ollama/ollama/blob/main/docs/api.md |
| GitLab CI Reference | https://docs.gitlab.com/ee/ci/yaml/ |
| Claude MCP Guide | https://docs.anthropic.com/en/docs/mcp |

---

## Workshop Summary

By the end of this workshop you have:

- [x] Built a python3 MCP server with 5 custom tools
- [x] Integrated `gitpython3` to scan and diff local repos
- [x] Connected Ollama (llama3.2:1b) as a local LLM for code review
- [x] Used Ollama to auto-generate `.gitlab-ci.yml`
- [x] Registered the server with Claude Desktop
- [x] Run end-to-end prompts through the full pipeline

---

*Workshop — Git-Aware MCP + Ollama · April 2026*
