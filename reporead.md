#!/usr/bin/env python3
"""
Dockerfile Suggester
Scans a local git repo, queries llama3.2:1b via Ollama, and auto-generates:
  - Dockerfile
  - .dockerignore
  - docker-compose.yml
  - DOCKERFILE_SUGGESTION.md  (full report)
"""

import os
import re
import subprocess
import json
import sys
import fnmatch
import urllib.request
import urllib.error

OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL = "llama3.2:1b"

SCAN_FILES = [
    "requirements.txt", "requirements*.txt",
    "Pipfile", "Pipfile.lock",
    "pyproject.toml", "setup.py", "setup.cfg",
    "package.json", "package-lock.json", "yarn.lock",
    "go.mod", "go.sum",
    "pom.xml", "build.gradle",
    "Cargo.toml",
    "Gemfile", "Gemfile.lock",
    "composer.json",
    "*.csproj",
    "Makefile",
    "Dockerfile", "docker-compose.yml", "docker-compose.yaml",
    ".env.example",
    "main.py", "app.py", "server.py", "manage.py", "wsgi.py", "asgi.py",
    "index.js", "server.js", "app.js",
    "main.go",
    "main.rs",
]

SKIP_DIRS = {
    "node_modules", "__pycache__", ".venv", "venv", "env",
    "dist", "build", "target", ".git", ".idea", ".vscode"
}


# ─── Repo Scanning ────────────────────────────────────────────────────────────

def prompt_repo_path():
    while True:
        path = input("\nEnter the path to your local git repo: ").strip()
        path = os.path.expanduser(path)
        if not path:
            print("  [!] Path cannot be empty.")
            continue
        if not os.path.isdir(path):
            print(f"  [!] Directory not found: {path}")
            continue
        if not os.path.isdir(os.path.join(path, ".git")):
            ans = input(f"  [?] No .git folder at '{path}'. Use anyway? (y/n): ").strip().lower()
            if ans != "y":
                continue
        return path


def get_git_info(repo_path):
    info = {}
    for cmd, key in [
        (["git", "-C", repo_path, "log", "--oneline", "-5"], "recent_commits"),
        (["git", "-C", repo_path, "branch", "--show-current"], "current_branch"),
        (["git", "-C", repo_path, "remote", "get-url", "origin"], "remote_url"),
    ]:
        try:
            r = subprocess.run(cmd, capture_output=True, text=True)
            info[key] = r.stdout.strip()
        except Exception:
            info[key] = "N/A"
    return info


def scan_repo(repo_path):
    collected = {}
    for root, dirs, files in os.walk(repo_path):
        dirs[:] = [d for d in dirs if not d.startswith(".") and d not in SKIP_DIRS]
        rel_root = os.path.relpath(root, repo_path)
        for fname in files:
            matched = any(
                fnmatch.fnmatch(fname, p) if "*" in p else fname == p
                for p in SCAN_FILES
            )
            if matched:
                filepath = os.path.join(root, fname)
                rel_path = os.path.join(rel_root, fname) if rel_root != "." else fname
                try:
                    with open(filepath, "r", encoding="utf-8", errors="ignore") as f:
                        collected[rel_path] = f.read(3000)
                    print(f"  [+] {rel_path}")
                except Exception as e:
                    print(f"  [!] Could not read {rel_path}: {e}")
    return collected


def get_directory_structure(repo_path, max_depth=3):
    lines = []

    def walk(path, prefix="", depth=0):
        if depth > max_depth:
            return
        try:
            entries = sorted(
                e for e in os.listdir(path)
                if not e.startswith(".") and e not in SKIP_DIRS
            )
        except PermissionError:
            return
        for i, entry in enumerate(entries):
            full = os.path.join(path, entry)
            conn = "└── " if i == len(entries) - 1 else "├── "
            lines.append(f"{prefix}{conn}{entry}")
            if os.path.isdir(full):
                ext = "    " if i == len(entries) - 1 else "│   "
                walk(full, prefix + ext, depth + 1)

    walk(repo_path)
    return "\n".join(lines[:120])


# ─── LLM ──────────────────────────────────────────────────────────────────────

def build_prompt(repo_path, scanned_files, git_info, dir_structure):
    files_summary = "".join(
        f"\n### {p}\n```\n{c[:1500]}\n```\n"
        for p, c in scanned_files.items()
    ) or "No key config files found."

    return f"""You are a Docker expert. Analyze this git repository and generate all required Docker files.

## Repository Info
- Path: {repo_path}
- Branch: {git_info.get('current_branch', 'unknown')}
- Remote: {git_info.get('remote_url', 'N/A')}

## Directory Structure
```
{dir_structure}
```

## Key Files Found
{files_summary}

## Your Task
Detect the stack and generate ALL of the following. Use EXACTLY these section headers:

### STACK SUMMARY
(Describe detected language, framework, runtime, port)

### Dockerfile
```dockerfile
(complete best-practices Dockerfile: pinned base image, multi-stage if applicable,
non-root user, chained RUN commands, HEALTHCHECK, EXPOSE, ARG/ENV best practices)
```

### .dockerignore
```
(complete .dockerignore)
```

### docker-compose.yml
```yaml
(complete docker-compose.yml: build context, ports, environment, healthcheck, restart policy)
```

### RECOMMENDATIONS
(bullet-point security and best-practice tips specific to this project)

Output ONLY these sections. No extra prose outside the sections.
"""


def call_ollama(prompt):
    payload = json.dumps({"model": MODEL, "prompt": prompt, "stream": True}).encode()
    req = urllib.request.Request(
        OLLAMA_URL, data=payload,
        headers={"Content-Type": "application/json"}, method="POST"
    )
    print(f"\n{'='*60}")
    print(f"  Querying {MODEL} via Ollama — please wait...")
    print(f"{'='*60}\n")

    tokens = []
    try:
        with urllib.request.urlopen(req, timeout=180) as resp:
            for line in resp:
                line = line.decode("utf-8").strip()
                if not line:
                    continue
                try:
                    data = json.loads(line)
                    tok = data.get("response", "")
                    print(tok, end="", flush=True)
                    tokens.append(tok)
                    if data.get("done"):
                        break
                except json.JSONDecodeError:
                    continue
    except urllib.error.URLError as e:
        print(f"\n\n[ERROR] Cannot connect to Ollama at {OLLAMA_URL}")
        print("  → Run: ollama serve")
        print(f"  → Run: ollama pull {MODEL}")
        print(f"  Details: {e}")
        sys.exit(1)

    return "".join(tokens)


# ─── File Extraction & Writing ────────────────────────────────────────────────

def extract_code_block(text, section_header):
    """Extract the first fenced code block under a ### section header."""
    pattern = rf"###\s*{re.escape(section_header)}\s*\n.*?```[^\n]*\n(.*?)```"
    m = re.search(pattern, text, re.DOTALL | re.IGNORECASE)
    return m.group(1).strip() if m else None


def extract_plain_section(text, section_header):
    """Extract plain text (no code fence) under a ### section header."""
    pattern = rf"###\s*{re.escape(section_header)}\s*\n(.*?)(?=\n###|\Z)"
    m = re.search(pattern, text, re.DOTALL | re.IGNORECASE)
    return m.group(1).strip() if m else None


def write_file(path, content, label):
    try:
        with open(path, "w", encoding="utf-8") as f:
            f.write(content + "\n")
        print(f"  [OK] {label:30s} → {path}")
        return True
    except Exception as e:
        print(f"  [!] Failed to write {path}: {e}")
        return False


def generate_files(llm_response, repo_path):
    print(f"\n{'='*60}")
    print("  Generating files...")
    print(f"{'='*60}")

    generated = []

    # 1. Dockerfile
    content = extract_code_block(llm_response, "Dockerfile")
    if content:
        if write_file(os.path.join(repo_path, "Dockerfile"), content, "Dockerfile"):
            generated.append("Dockerfile")
    else:
        print("  [!] Could not extract Dockerfile block.")

    # 2. .dockerignore
    content = extract_code_block(llm_response, ".dockerignore")
    if content:
        if write_file(os.path.join(repo_path, ".dockerignore"), content, ".dockerignore"):
            generated.append(".dockerignore")
    else:
        print("  [!] Could not extract .dockerignore block.")

    # 3. docker-compose.yml
    content = extract_code_block(llm_response, "docker-compose.yml")
    if content:
        if write_file(os.path.join(repo_path, "docker-compose.yml"), content, "docker-compose.yml"):
            generated.append("docker-compose.yml")
    else:
        print("  [!] Could not extract docker-compose.yml block.")

    # 4. Markdown report
    stack = extract_plain_section(llm_response, "STACK SUMMARY") or "_Not detected_"
    recommendations = extract_plain_section(llm_response, "RECOMMENDATIONS") or "_None_"

    report = f"""# Dockerfile Suggestion Report

> Generated by `dockerfile_suggester.py` using `{MODEL}`

---

## Stack Summary

{stack}

---

## Generated Files

{chr(10).join(f'- `{f}`' for f in generated)}

---

## Recommendations

{recommendations}

---

## Full LLM Response

<details>
<summary>Click to expand</summary>

```
{llm_response}
```

</details>
"""
    report_path = os.path.join(repo_path, "DOCKERFILE_SUGGESTION.md")
    if write_file(report_path, report, "DOCKERFILE_SUGGESTION.md"):
        generated.append("DOCKERFILE_SUGGESTION.md")

    return generated


# ─── Main ─────────────────────────────────────────────────────────────────────

def main():
    print("=" * 60)
    print("   Dockerfile Suggester — powered by llama3.2:1b (Ollama)")
    print("=" * 60)

    repo_path = prompt_repo_path()
    print(f"\n  Repo : {repo_path}")

    print("\n  [1/4] Gathering git info...")
    git_info = get_git_info(repo_path)
    print(f"        Branch : {git_info.get('current_branch', 'N/A')}")
    print(f"        Remote : {git_info.get('remote_url', 'N/A')}")

    print("\n  [2/4] Building directory tree...")
    dir_structure = get_directory_structure(repo_path)

    print("\n  [3/4] Scanning key project files...")
    scanned_files = scan_repo(repo_path)
    if not scanned_files:
        print("  [!] No recognizable project files found — using directory structure only.")

    print("\n  [4/4] Querying LLM...")
    prompt = build_prompt(repo_path, scanned_files, git_info, dir_structure)
    llm_response = call_ollama(prompt)

    generated = generate_files(llm_response, repo_path)

    print(f"\n{'='*60}")
    print(f"  Done! {len(generated)} file(s) written to:")
    for f in generated:
        print(f"    → {os.path.join(repo_path, f)}")
    print(f"{'='*60}\n")


if __name__ == "__main__":
    main()
