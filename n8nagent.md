# Workshop: n8n Agent for GitLab CI
### Trigger Pipelines · Monitor Status · Generate CI Config with Ollama · Auto-Fix Failures

---

## What is n8n?

n8n is a self-hostable workflow automation tool. You connect nodes visually to
build agents that react to events, call APIs, run code, and send notifications.
No Claude Desktop or MCP required — everything runs inside n8n.

---

## What the Agent Does

```
+------------------+     webhook / schedule / manual
|   Trigger node   |
+--------+---------+
         |
+--------v---------+
|  GitLab API node |  -- fetch pipeline status / MR diff
+--------+---------+
         |
+--------v---------+
|  Ollama HTTP node|  -- analyze failures OR generate CI YAML
+--------+---------+
         |
    +----+----+
    |         |
+---v---+ +---v--------+
| Write | | GitLab API |  -- commit .gitlab-ci.yml OR post MR comment
| File  | | (comment)  |
+-------+ +------------+
         |
+--------v---------+
| Notify (Slack /  |
| Email / webhook) |
+------------------+
```

We will build three separate agents:

- Agent 1 — Monitor pipeline and notify on failure
- Agent 2 — Analyze failure log with Ollama and post fix suggestion as MR comment
- Agent 3 — Generate .gitlab-ci.yml from repo info and commit it via API

---

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| Docker + Docker Compose | Run n8n locally | https://docs.docker.com/get-docker |
| Ollama | Local LLM (llama3.2:1b) | https://ollama.com |
| GitLab account | API access | https://gitlab.com or self-hosted |
| GitLab personal access token | API calls | Settings > Access Tokens |

---

## Part 1 — Run n8n Locally

### Step 1.1 — docker-compose.yml

Create a folder and add this file:

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

### Step 1.2 — Start n8n

```bash
docker compose up -d
```

Open: http://localhost:5678

Create an account when prompted (local only, no internet needed).

### Step 1.3 — Start Ollama

```bash
ollama serve &
ollama pull llama3.2:1b
```

> n8n runs inside Docker. To reach Ollama on your host machine use
> `http://host.docker.internal:11434` on macOS/Windows or your host IP
> on Linux (e.g. `http://172.17.0.1:11434`).

---

## Part 2 — GitLab Setup

### Step 2.1 — Create a Personal Access Token

1. GitLab > top-right avatar > Edit profile
2. Left menu: Access Tokens
3. Click "Add new token"
4. Name: `n8n-agent`
5. Scopes: `api`, `read_repository`, `write_repository`
6. Copy the token — you will not see it again

### Step 2.2 — Find Your Project ID

Open your GitLab project. The Project ID is shown below the project name on
the main page (e.g. `12345678`). Note it down.

### Step 2.3 — Store Credentials in n8n

1. n8n sidebar: Credentials > Add credential
2. Search for "Header Auth"
3. Name: `GitLab Token`
4. Header name: `PRIVATE-TOKEN`
5. Header value: your token
6. Save

---

## Part 3 — Agent 1: Monitor Pipeline and Notify

This agent polls GitLab every 5 minutes and sends a message when a pipeline fails.

### Nodes to add (in order)

```
[Schedule Trigger] --> [HTTP Request: get pipelines] --> [IF: status == failed]
                                                              |
                                                    [Slack / Email notify]
```

### Node 1 — Schedule Trigger

- Trigger interval: Every 5 minutes

### Node 2 — HTTP Request (Get Pipelines)

```
Method:  GET
URL:     https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/pipelines?per_page=5
Auth:    Header Auth -> GitLab Token
```

### Node 3 — IF node

```
Condition:
  Value 1:   {{ $json.status }}
  Operation: Equal
  Value 2:   failed
```

### Node 4 — Slack node (or Send Email)

Message template:

```
Pipeline {{ $json.id }} FAILED on branch {{ $json.ref }}
Started: {{ $json.created_at }}
URL: {{ $json.web_url }}
```

### How to test

1. Click "Test workflow" in n8n
2. Manually fail a pipeline in GitLab (push broken code)
3. Confirm the notification arrives

---

## Part 4 — Agent 2: Analyze Failure + Post Fix Suggestion

This agent is triggered by a GitLab webhook when a pipeline fails.
It fetches the job log, sends it to Ollama, and posts the suggestion as an MR comment.

### Step 4.1 — Create a Webhook in n8n

1. Add a "Webhook" trigger node
2. Method: POST
3. Path: `gitlab-pipeline`
4. Copy the webhook URL shown (e.g. `http://localhost:5678/webhook/gitlab-pipeline`)

### Step 4.2 — Register the Webhook in GitLab

1. GitLab project > Settings > Webhooks
2. URL: your n8n webhook URL
3. Trigger: Pipeline events
4. Add webhook

### Node flow

```
[Webhook Trigger]
       |
[IF: object_attributes.status == "failed"]
       |
[HTTP Request: get job log]
       |
[HTTP Request: Ollama analyze]
       |
[HTTP Request: post MR comment]
```

### Node 3 — HTTP Request: Get Job Log

```
Method: GET
URL:    https://gitlab.com/api/v4/projects/{{ $json.body.project.id }}/jobs/{{ $json.body.builds[0].id }}/trace
Auth:   Header Auth -> GitLab Token
```

### Node 4 — HTTP Request: Ollama Analyze

```
Method:  POST
URL:     http://host.docker.internal:11434/api/generate
Body (JSON):
{
  "model": "llama3.2:1b",
  "stream": false,
  "system": "You are a DevOps expert. Analyze CI failure logs and suggest fixes. Be concise.",
  "prompt": "This GitLab CI job failed. Analyze the log and suggest a fix:\n\n{{ $json.data.substring(0, 4000) }}"
}
```

### Node 5 — HTTP Request: Post MR Comment

First get the MR IID from the webhook payload, then post a note:

```
Method: POST
URL:    https://gitlab.com/api/v4/projects/{{ $('Webhook').item.json.body.project.id }}/merge_requests/{{ $('Webhook').item.json.body.merge_request.iid }}/notes
Auth:   Header Auth -> GitLab Token
Body (JSON):
{
  "body": "**CI Failure Analysis (llama3.2:1b)**\n\n{{ $json.response }}"
}
```

> If the pipeline is not linked to an MR, post to the commit instead:
> `POST /projects/:id/repository/commits/:sha/comments`

---

## Part 5 — Agent 3: Generate .gitlab-ci.yml and Commit It

This agent runs on demand (Manual Trigger). It reads the repo file tree, asks
Ollama to generate a CI config, then commits the file directly to GitLab via API.

### Node flow

```
[Manual Trigger]
       |
[HTTP Request: list repo tree]
       |
[Code node: build prompt]
       |
[HTTP Request: Ollama generate]
       |
[Code node: extract YAML + base64 encode]
       |
[HTTP Request: commit file to GitLab]
```

### Node 1 — Manual Trigger

No config needed. Click "Execute" in n8n to run.

### Node 2 — HTTP Request: List Repo Tree

```
Method: GET
URL:    https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/repository/tree?recursive=false&per_page=50
Auth:   Header Auth -> GitLab Token
```

### Node 3 — Code Node: Build Prompt

```javascript
// Code node (JavaScript)
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

### Node 4 — HTTP Request: Ollama Generate

```
Method:  POST
URL:     http://host.docker.internal:11434/api/generate
Body (JSON):
{
  "model": "llama3.2:1b",
  "stream": false,
  "system": "You are a DevOps engineer. Output valid GitLab CI YAML only. No markdown fences.",
  "prompt": "{{ $json.prompt }}"
}
```

### Node 5 — Code Node: Encode for GitLab API

GitLab requires file content to be base64 encoded when creating/updating files.

```javascript
// Code node (JavaScript)
const yamlContent = $input.first().json.response;

// Base64 encode
const encoded = Buffer.from(yamlContent).toString("base64");

return [{
  json: {
    raw_yaml: yamlContent,
    encoded:  encoded
  }
}];
```

### Node 6 — HTTP Request: Commit File to GitLab

This creates or updates `.gitlab-ci.yml` on the main branch.

```
Method: POST
URL:    https://gitlab.com/api/v4/projects/YOUR_PROJECT_ID/repository/files/.gitlab-ci.yml
Auth:   Header Auth -> GitLab Token
Body (JSON):
{
  "branch":         "main",
  "author_name":    "n8n Agent",
  "author_email":   "n8n@agent.local",
  "commit_message": "ci: auto-generate .gitlab-ci.yml via n8n + llama3.2:1b",
  "encoding":       "base64",
  "content":        "{{ $json.encoded }}"
}
```

> If the file already exists use `PUT` instead of `POST`. You can add an IF
> node before this step to check with a GET request first.

---

## Part 6 — Connecting All Three Agents

You can link agents together using n8n's "Execute Workflow" node.

Suggested combined flow:

```
[Webhook: pipeline failed]
         |
[Agent 2: analyze + comment]
         |
[IF: comment contains "missing .gitlab-ci.yml"]
         |
[Execute Workflow: Agent 3 -> generate + commit CI file]
         |
[Notify: Slack "CI file committed, re-run pipeline"]
```

---

## Part 7 — Useful GitLab API Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| List pipelines | GET | `/projects/:id/pipelines` |
| Get pipeline detail | GET | `/projects/:id/pipelines/:pid` |
| Get job log | GET | `/projects/:id/jobs/:jid/trace` |
| List jobs in pipeline | GET | `/projects/:id/pipelines/:pid/jobs` |
| Retry pipeline | POST | `/projects/:id/pipelines/:pid/retry` |
| List repo tree | GET | `/projects/:id/repository/tree` |
| Get file content | GET | `/projects/:id/repository/files/:path` |
| Create file | POST | `/projects/:id/repository/files/:path` |
| Update file | PUT | `/projects/:id/repository/files/:path` |
| Post MR comment | POST | `/projects/:id/merge_requests/:iid/notes` |
| Post commit comment | POST | `/projects/:id/repository/commits/:sha/comments` |

Base URL: `https://gitlab.com/api/v4`
For self-hosted: `https://your-gitlab.example.com/api/v4`

---

## Part 8 — Troubleshooting

### n8n cannot reach Ollama

On Linux, Docker cannot use `host.docker.internal`. Use your host IP instead:

```bash
# Find your Docker bridge IP
ip route | grep docker
# Usually 172.17.0.1

# Use in n8n nodes:
http://172.17.0.1:11434/api/generate
```

---

### GitLab returns 404 on file commit

- Confirm the project ID is correct
- Confirm the branch name matches (`main` vs `master`)
- Confirm the token has `write_repository` scope

---

### Ollama returns empty response

```bash
# Test directly from terminal
curl http://localhost:11434/api/generate \
  -d '{"model":"llama3.2:1b","prompt":"hello","stream":false}'
```

If it works in terminal but not in n8n, the URL inside Docker is wrong.

---

### Webhook not firing

- n8n must be reachable from GitLab. If running locally, use ngrok:

```bash
ngrok http 5678
# Use the https://xxxx.ngrok.io URL in GitLab webhook settings
```

---

## Part 9 — Exercises

### Exercise 1

Build Agent 1. Manually trigger a pipeline failure in GitLab and confirm the
notification arrives in n8n's execution log.

### Exercise 2

Build Agent 2. Push a broken `.gitlab-ci.yml` to trigger a real failure,
then check that an MR comment with Ollama's analysis appears.

### Exercise 3

Build Agent 3. Run it manually and confirm `.gitlab-ci.yml` appears as a new
commit in your GitLab repo.

### Exercise 4 (Challenge)

Link all three agents. Use an IF node to skip Agent 3 if a CI file already
exists. Add a Slack notification at the end of the full chain.

---

## Reference

| Resource | URL |
|----------|-----|
| n8n docs | https://docs.n8n.io |
| n8n HTTP Request node | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest |
| GitLab API docs | https://docs.gitlab.com/ee/api/rest |
| Ollama API docs | https://github.com/ollama/ollama/blob/main/docs/api.md |
| ngrok (local tunnel) | https://ngrok.com |

---

## Workshop Summary

By the end of this workshop you have:

- [x] Deployed n8n locally with Docker
- [x] Stored GitLab credentials securely in n8n
- [x] Built Agent 1: pipeline monitor with notifications
- [x] Built Agent 2: failure log analyzer using llama3.2:1b, result posted as MR comment
- [x] Built Agent 3: .gitlab-ci.yml generator committed directly to GitLab
- [x] Connected agents into a single automated pipeline

---

*Workshop — n8n + GitLab CI + Ollama · April 2026*
