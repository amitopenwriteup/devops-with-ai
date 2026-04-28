# n8n Agent Workshop — GitLab CI Automation

> **Trigger Pipelines · Monitor Status · Generate CI Config with Ollama · Auto-Fix Failures**

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [One-Time Setup](#one-time-setup)
   - [Run n8n with Docker](#run-n8n-with-docker)
   - [Start Ollama](#start-ollama)
   - [GitLab Personal Access Token](#gitlab-personal-access-token)
   - [Store Credentials in n8n](#store-credentials-in-n8n)
4. [Agent 1 — Monitor Pipeline & Notify](#agent-1--monitor-pipeline--notify)
5. [Agent 2 — Analyze Failure & Post Fix Suggestion](#agent-2--analyze-failure--post-fix-suggestion)
6. [Agent 3 — Generate .gitlab-ci.yml & Commit](#agent-3--generate-gitlab-ciyml--commit)
7. [Connecting All Three Agents](#connecting-all-three-agents)
8. [GitLab API Quick Reference](#gitlab-api-quick-reference)
9. [Troubleshooting](#troubleshooting)
10. [Exercises](#exercises)

---

## Overview

This workshop builds three n8n automation agents that integrate GitLab CI with a local Ollama LLM. No Claude Desktop or MCP required — everything runs inside n8n.

### What Each Agent Does

| Agent | Trigger | Action |
|-------|---------|--------|
| **Agent 1** | Schedule (every 5 min) | Polls GitLab for failed pipelines → sends Slack/email notification |
| **Agent 2** | GitLab webhook (pipeline event) | Fetches job log → asks Ollama to analyze → posts fix suggestion as MR comment |
| **Agent 3** | Manual trigger | Reads repo file tree → asks Ollama to generate `.gitlab-ci.yml` → commits file to GitLab |

### Combined Flow Diagram

```
[Webhook: pipeline failed]
         │
[Agent 2: analyze + post MR comment]
         │
[IF: "missing .gitlab-ci.yml" in analysis?]
         │
[Execute Workflow: Agent 3 → generate + commit CI file]
         │
[Notify: Slack — "CI file committed, re-run pipeline"]
```

---

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| Docker + Docker Compose | Run n8n locally | https://docs.docker.com/get-docker |
| Ollama | Local LLM (`llama3.2:1b`) | https://ollama.com |
| GitLab account | API access | https://gitlab.com or self-hosted |
| GitLab personal access token | Authenticate API calls | Settings → Access Tokens |

---

## One-Time Setup

### Run n8n with Docker

**Step 1** — Create a new folder and add the following `docker-compose.yml`:

```yaml
# docker-compose.yml
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://localhost:5678
      - GENERIC_TIMEZONE=Asia/Kolkata
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

**Step 2** — Start n8n:

```bash
docker compose up -d
```

**Step 3** — Open n8n in your browser at `http://localhost:5678` and create a local account when prompted.

---

### Start Ollama

```bash
ollama serve &
ollama pull llama3.2:1b
```

> **Important — Docker networking:**
> n8n runs inside a Docker container, so it cannot use `localhost` to reach Ollama on your host machine.
>
> | OS | URL to use inside n8n |
> |----|----------------------|
> | macOS / Windows | `http://host.docker.internal:11434` |
> | Linux | `http://172.17.0.1:11434` (find with `ip route \| grep docker`) |

---

### GitLab Personal Access Token

1. Go to GitLab → top-right avatar → **Edit profile**
2. Left menu → **Access Tokens**
3. Click **Add new token**
4. Name: `n8n-agent`
5. Scopes: check `api`, `read_repository`, `write_repository`
6. Click **Create** and copy the token immediately — you will not see it again

**Find Your Project ID** — Open your GitLab project. The Project ID is displayed below the project name on the main page (e.g. `12345678`). Note it down.

---

### Store Credentials in n8n

1. In the n8n sidebar click **Credentials → Add credential**
2. Search for **Header Auth**
3. Fill in:

| Field | Value |
|-------|-------|
| Name | `GitLab Token` |
| Header Name | `PRIVATE-TOKEN` |
| Header Value | *(paste your token)* |

4. Click **Save**

---

## Agent 1 — Monitor Pipeline & Notify

This agent polls GitLab every 5 minutes and sends a notification whenever a pipeline fails.

### Node Flow

```
[Schedule Trigger] ──► [HTTP Request: get pipelines] ──► [IF: status == "failed"]
                                                                    │
                                                         [Slack / Send Email]
```

---

### Node 1 — Schedule Trigger

In the n8n canvas, click **+ Add first step** → search **Schedule Trigger**.

| Setting | Value |
|---------|-------|
| Trigger interval | Every **5 Minutes** |

---

### Node 2 — HTTP Request: Get Pipelines

Click **+** on the right of the Schedule Trigger → search **HTTP Request**.

| Setting | Value |
|---------|-------|
| Method | `GET` |
| URL | `https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/pipelines?per_page=5` |
| Authentication | Header Auth → **GitLab Token** |

> Replace `YOUR_PROJECT_ID` with the numeric ID from your GitLab project page.

---

### Node 3 — IF Node: Check Status

Click **+** → search **IF**.

| Setting | Value |
|---------|-------|
| Value 1 | `{{ $json.status }}` |
| Operation | Equal |
| Value 2 | `failed` |

Two output branches appear: **true** (pipeline failed) and **false** (pipeline ok). Connect the notify node to the **true** branch.

---

### Node 4 — Slack or Send Email: Notify

Connect from the **true** branch → search **Slack** or **Send Email**.

Use this message template:

```
Pipeline {{ $json.id }} FAILED on branch {{ $json.ref }}
Started: {{ $json.created_at }}
URL: {{ $json.web_url }}
```

### How to Test Agent 1

1. Click **Test workflow** in n8n
2. Push broken code to GitLab to manually trigger a pipeline failure
3. Confirm the notification arrives in Slack / email

---

## Agent 2 — Analyze Failure & Post Fix Suggestion

This agent is triggered by a GitLab webhook when a pipeline fails. It fetches the job log, sends it to Ollama, and posts the suggestion as an MR comment.

### Node Flow

```
[Webhook Trigger]
       │
[IF: status == "failed"]
       │
[HTTP Request: get job log]
       │
[HTTP Request: Ollama analyze]
       │
[HTTP Request: post MR comment]
```

---

### Node 1 — Webhook Trigger

Click **+ Add first step** → search **Webhook**.

| Setting | Value |
|---------|-------|
| HTTP Method | `POST` |
| Path | `gitlab-pipeline` |

After saving, **copy the Webhook URL** shown at the bottom of the panel (e.g. `http://localhost:5678/webhook/gitlab-pipeline`). You will paste this into GitLab in the next step.

---

### Register the Webhook in GitLab

1. GitLab project → **Settings → Webhooks**
2. URL: paste the n8n webhook URL
3. Trigger: check **Pipeline events**
4. Click **Add webhook**

> **Can't reach localhost from GitLab?** Use ngrok:
> ```bash
> ngrok http 5678
> # Paste the https://xxxx.ngrok.io URL into GitLab instead
> ```

---

### Node 2 — IF Node: Check Failed

Click **+** → search **IF**.

| Setting | Value |
|---------|-------|
| Value 1 | `{{ $json.body.object_attributes.status }}` |
| Operation | Equal |
| Value 2 | `failed` |

Connect subsequent nodes to the **true** branch.

---

### Node 3 — HTTP Request: Get Job Log

| Setting | Value |
|---------|-------|
| Method | `GET` |
| URL | `https://gitlab.com/api/v4/projects/{{ $json.body.project.id }}/jobs/{{ $json.body.builds[0].id }}/trace` |
| Authentication | Header Auth → **GitLab Token** |

---

### Node 4 — HTTP Request: Ollama Analyze

| Setting | Value |
|---------|-------|
| Method | `POST` |
| URL | `http://host.docker.internal:11434/api/generate` |

**Body (JSON):**

```json
{
  "model": "llama3.2:1b",
  "stream": false,
  "system": "You are a DevOps expert. Analyze CI failure logs and suggest fixes. Be concise.",
  "prompt": "This GitLab CI job failed. Analyze the log and suggest a fix:\n\n{{ $json.data.substring(0, 4000) }}"
}
```

---

### Node 5 — HTTP Request: Post MR Comment

| Setting | Value |
|---------|-------|
| Method | `POST` |
| URL | `https://gitlab.com/api/v4/projects/{{ $('Webhook').item.json.body.project.id }}/merge_requests/{{ $('Webhook').item.json.body.merge_request.iid }}/notes` |
| Authentication | Header Auth → **GitLab Token** |

**Body (JSON):**

```json
{
  "body": "**CI Failure Analysis (llama3.2:1b)**\n\n{{ $json.response }}"
}
```

> If the pipeline is not linked to an MR, post to the commit instead:
> `POST /projects/:id/repository/commits/:sha/comments`

### How to Test Agent 2

1. Push a broken `.gitlab-ci.yml` to GitLab
2. Wait for the pipeline to fail and fire the webhook
3. Confirm an MR comment with Ollama's analysis appears

---

## Agent 3 — Generate .gitlab-ci.yml & Commit

This agent runs on demand. It reads the repo file tree, asks Ollama to generate a CI config, then commits the file directly to GitLab via the API.

### Node Flow

```
[Manual Trigger]
       │
[HTTP Request: list repo tree]
       │
[Code: build prompt]
       │
[HTTP Request: Ollama generate]
       │
[Code: extract YAML + base64 encode]
       │
[HTTP Request: commit file to GitLab]
```

---

### Node 1 — Manual Trigger

Click **+ Add first step** → search **Manual Trigger**.

No configuration needed. Click **Execute workflow** in n8n to run on demand.

---

### Node 2 — HTTP Request: List Repo Tree

| Setting | Value |
|---------|-------|
| Method | `GET` |
| URL | `https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/repository/tree?recursive=false&per_page=50` |
| Authentication | Header Auth → **GitLab Token** |

---

### Node 3 — Code Node: Build Prompt

Click **+** → search **Code** → set language to **JavaScript**.

```javascript
const files = $input.all().map(item => item.json.path).join("\n");

return [{
  json: {
    file_list: files,
    prompt:
      "Generate a .gitlab-ci.yml for a project with these files:\n\n" +
      files +
      "\n\nInclude stages: build, test, deploy. " +
      "Use caching. Add only:deploy for the main branch. " +
      "Return raw YAML only. No explanation. No markdown fences."
  }
}];
```

---

### Node 4 — HTTP Request: Ollama Generate

| Setting | Value |
|---------|-------|
| Method | `POST` |
| URL | `http://host.docker.internal:11434/api/generate` |

**Body (JSON):**

```json
{
  "model": "llama3.2:1b",
  "stream": false,
  "system": "You are a DevOps engineer. Output valid GitLab CI YAML only. No markdown fences.",
  "prompt": "{{ $json.prompt }}"
}
```

---

### Node 5 — Code Node: Base64 Encode

Add another **Code** node → language: **JavaScript**.

```javascript
const yamlContent = $input.first().json.response;

// GitLab API requires base64-encoded file content
const encoded = Buffer.from(yamlContent).toString("base64");

return [{
  json: {
    raw_yaml: yamlContent,
    encoded:  encoded
  }
}];
```

---

### Node 6 — HTTP Request: Commit File to GitLab

| Setting | Value |
|---------|-------|
| Method | `POST` *(use `PUT` if file already exists)* |
| URL | `https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/repository/files/.gitlab-ci.yml` |
| Authentication | Header Auth → **GitLab Token** |

**Body (JSON):**

```json
{
  "branch":         "main",
  "author_name":    "n8n Agent",
  "author_email":   "n8n@agent.local",
  "commit_message": "ci: auto-generate .gitlab-ci.yml via n8n + llama3.2:1b",
  "encoding":       "base64",
  "content":        "{{ $json.encoded }}"
}
```

> To safely handle both create and update, add an IF node before this step that first does a `GET` on the file path and switches between `POST` and `PUT` based on the response status.

### How to Test Agent 3

1. Click **Execute workflow** in n8n
2. Open your GitLab repo and confirm `.gitlab-ci.yml` appears as a new commit on `main`

---

## Connecting All Three Agents

Use n8n's **Execute Workflow** node to chain agents together.

### Combined Flow

```
[Webhook: pipeline failed]
         │
[Agent 2: analyze + post MR comment]
         │
[IF: comment contains "missing .gitlab-ci.yml"]
         │
[Execute Workflow: Agent 3 → generate + commit CI file]
         │
[Slack notify: "CI file committed — re-run pipeline"]
```

**Steps to link:**

1. In Agent 2's workflow, add an **IF** node after the MR comment step
2. Check if `{{ $json.response }}` contains `missing .gitlab-ci.yml`
3. On the **true** branch, add an **Execute Workflow** node pointing to Agent 3's workflow ID
4. After that, add a **Slack** node with the commit notification

---

## GitLab API Quick Reference

Base URL: `https://gitlab.com/api/v4`
Self-hosted: `https://your-gitlab.example.com/api/v4`

| Action | Method | Endpoint |
|--------|--------|----------|
| List pipelines | `GET` | `/projects/:id/pipelines` |
| Get pipeline detail | `GET` | `/projects/:id/pipelines/:pid` |
| Get job log | `GET` | `/projects/:id/jobs/:jid/trace` |
| List jobs in pipeline | `GET` | `/projects/:id/pipelines/:pid/jobs` |
| Retry pipeline | `POST` | `/projects/:id/pipelines/:pid/retry` |
| List repo tree | `GET` | `/projects/:id/repository/tree` |
| Get file content | `GET` | `/projects/:id/repository/files/:path` |
| Create file | `POST` | `/projects/:id/repository/files/:path` |
| Update file | `PUT` | `/projects/:id/repository/files/:path` |
| Post MR comment | `POST` | `/projects/:id/merge_requests/:iid/notes` |
| Post commit comment | `POST` | `/projects/:id/repository/commits/:sha/comments` |

---

## Troubleshooting

### n8n cannot reach Ollama

On Linux, Docker cannot resolve `host.docker.internal`. Find your Docker bridge IP and use it instead:

```bash
# Find your Docker bridge IP
ip route | grep docker
# Usually 172.17.0.1

# Use in n8n nodes:
# http://172.17.0.1:11434/api/generate
```

---

### GitLab returns 404 on file commit

Check all three of the following:

- The project ID in the URL is correct
- The branch name matches exactly (`main` vs `master`)
- The token has `write_repository` scope enabled

---

### Ollama returns an empty response

Test Ollama directly from your terminal to isolate the issue:

```bash
curl http://localhost:11434/api/generate \
  -d '{"model":"llama3.2:1b","prompt":"hello","stream":false}'
```

If this works in the terminal but not in n8n, the URL you're using inside Docker is incorrect. Check the networking note in the [Start Ollama](#start-ollama) section.

---

### Webhook not firing

n8n must be reachable from GitLab's servers. If running locally, expose it with ngrok:

```bash
ngrok http 5678
# Use the https://xxxx.ngrok.io URL in GitLab → Settings → Webhooks
```

---

## Exercises

### Exercise 1 — Basic Monitor (Agent 1)

Build Agent 1. Manually trigger a pipeline failure in GitLab and confirm the notification arrives in n8n's execution log under the **Executions** tab.

### Exercise 2 — Failure Analyzer (Agent 2)

Build Agent 2. Push a broken `.gitlab-ci.yml` to trigger a real failure, then check that an MR comment with Ollama's analysis appears on the merge request.

### Exercise 3 — CI Generator (Agent 3)

Build Agent 3. Run it manually and confirm `.gitlab-ci.yml` appears as a new commit in your GitLab repository.

### Exercise 4 — Full Chain (Challenge)

Link all three agents into a single automated pipeline. Use an IF node to skip Agent 3 if a CI file already exists. Add a Slack notification at the end of the chain confirming what was committed.

---

## Reference Links

| Resource | URL |
|----------|-----|
| n8n documentation | https://docs.n8n.io |
| n8n HTTP Request node | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest |
| GitLab REST API docs | https://docs.gitlab.com/ee/api/rest |
| Ollama API docs | https://github.com/ollama/ollama/blob/main/docs/api.md |
| ngrok (local tunnel) | https://ngrok.com |

---

*Workshop — n8n + GitLab CI + Ollama · April 2026*
