# Workshop Guide: AWS Terraform RAG Generator (On-Prem, Ollama)

> **Goal:** Use a RAG-powered browser tool to read infrastructure/deployment docs and generate Terraform code — entirely on your machine using `nomic-embed-text` for embeddings and `llama3.2:1b` for generation. No cloud API. No data leaves your machine.

---

## Prerequisites (Already Completed)

You should already have the following running:

| Item | Status |
|---|---|
| Ollama installed | ✅ Done |
| `llama3.2:1b` pulled | ✅ Done |
| `nomic-embed-text` pulled | ✅ Done |
| Ollama server running with CORS open | ✅ Done |

If Ollama is not running, start it now:

```bash
pkill ollama
OLLAMA_ORIGINS="*" ollama serve
```

Verify both models are available:

```bash
ollama list
# Expected output:
# NAME                       ID              SIZE
# llama3.2:1b                baf6a787fdff    1.3 GB
# nomic-embed-text:latest    0a109f422b47    274 MB
```

---

## What is RAG?

**RAG = Retrieval-Augmented Generation**

Instead of passing your entire document to the LLM (which overflows the small 1b context window), RAG:

1. **Chunks** your document into small pieces (~300 words each)
2. **Embeds** each chunk into a vector using `nomic-embed-text`
3. **Retrieves** only the most relevant chunks when generating
4. **Passes** those chunks as focused context to `llama3.2:1b`

This means the tool handles long deployment guides, full READMEs, and multi-section docs — without truncation or garbled output.

```
Your Document
     │
     ▼
 Chunking (300-word windows)
     │
     ▼
 nomic-embed-text → vector embeddings (stored in memory)
     │
     ▼
 Query → cosine similarity → top-4 relevant chunks
     │
     ▼
 llama3.2:1b → JSON resources → HCL Terraform → variables.tf
```

---

## Step 1 — Open the RAG Terraform Generator Tool

Open the **AWS Terraform RAG Generator** artifact in Claude chat.

The tool loads directly in the browser — no installation needed.

---

## Step 2 — Configure Ollama Settings

In the tool UI, verify these fields:

| Field | Value |
|---|---|
| **Ollama URL** | `http://localhost:11434` |
| **Embed model** | `nomic-embed-text` |
| **Gen model** | `llama3.2:1b` |

Click **ping ↗** to test the connection.

**Expected:** `connected — models: llama3.2:1b, nomic-embed-text:latest`

If the ping fails:

```bash
# CORS must be open for browser → localhost communication
pkill ollama
OLLAMA_ORIGINS="*" ollama serve
```

---

## Step 3 — Paste Your Infrastructure Document

In the **"Paste your infrastructure / deployment guide"** text area, paste any of the following:

### Option A — Sample prompt (quick test)

```
Deploy on EC2 t3.medium with 50GB EBS gp3 volume
RDS PostgreSQL 14, db.t3.small, 20GB storage, Multi-AZ disabled
S3 bucket for build artifacts with versioning enabled
VPC with one public subnet and one private subnet in us-east-1
Application Load Balancer with HTTPS listener on port 443
ACM certificate for SSL
IAM role for EC2 to read S3 and connect to RDS
Security groups: ALB (80/443 open), EC2 (port 8080 from ALB), RDS (5432 from EC2)
```

### Option B — Paste a real guide

Copy-paste the **"Requirements"**, **"Infrastructure"**, or **"Deployment"** section from any product README, architecture doc, or AWS setup guide.

> **Tip:** Unlike the non-RAG version, you do NOT need to trim the document. RAG handles long docs by selecting only the relevant parts. Full READMEs work fine.

---

## Step 4 — Index the Document

Click **"index document ↗"**

The tool will:

| Step | What happens | Typical time |
|---|---|---|
| Chunking | Splits your doc into ~300-word overlapping windows | < 1 sec |
| Embedding | Calls `nomic-embed-text` for each chunk | 2–5 sec per chunk |
| Vector store | Stores embeddings in memory for retrieval | Instant |

You will see live counters for **chunks**, **embedded**, and **vectors**.

> ⏱️ For a 1,000-word doc (~4 chunks): ~10–20 seconds total.

**Expected result:** `index complete — N vectors ready`

---

## Step 5 — Run the Pipeline

Click **"analyze & generate ↗"**

The tool runs 4 sequential steps:

| Step | What happens | Typical time |
|---|---|---|
| 1. Retrieve | Embeds 3 infrastructure queries → cosine similarity → top-4 chunks | 5–15 sec |
| 2. Extract | `llama3.2:1b` reads retrieved chunks → outputs structured JSON of AWS resources | 15–40 sec |
| 3. Terraform | `llama3.2:1b` writes HCL for all resources using retrieved context | 30–90 sec |
| 4. Variables.tf | `llama3.2:1b` extracts variables and writes `variables.tf` | 15–40 sec |

> ⏱️ Total: ~1–3 minutes on CPU.

Watch the step indicators — each dot turns green when complete.

---

## Step 6 — Review the Output

The tool shows 4 tabs:

### Tab 1 — AWS Resources

Colored chips for each AWS service detected (EC2, RDS, S3, VPC, IAM, ALB, etc.).

**What to check:**
- Are all expected services shown?
- Hover over a chip to see the config detail detected
- If a service is missing, make it more explicit in your doc and re-index

### Tab 2 — Terraform Code

Complete HCL including:
- `provider "aws"` block
- All resource blocks with dependencies
- Security group ingress/egress rules
- IAM roles and policies
- Output blocks for ARNs and IDs

Click **copy** to copy the code.

### Tab 3 — variables.tf

All variables with types, descriptions, and default values. Copy separately.

### Tab 4 — Retrieved Chunks

Shows exactly which parts of your document were used, with relevance scores. Useful for debugging if output is missing a resource.

---

## Step 7 — Save and Use the Terraform Code

```bash
mkdir my-terraform-project && cd my-terraform-project
```

Paste Terraform tab → `main.tf`:

```bash
nano main.tf
```

Paste Variables tab → `variables.tf`:

```bash
nano variables.tf
```

Create `terraform.tfvars`:

```hcl
aws_region   = "us-east-1"
environment  = "dev"
project_name = "my-app"
```

Initialize and validate:

```bash
terraform init
terraform validate
terraform plan
```

> Always review the plan before applying. If `terraform validate` reports errors, check the **Troubleshooting** section below.

---

## Troubleshooting

### Ping fails — "could not reach Ollama"

```bash
pkill ollama
OLLAMA_ORIGINS="*" ollama serve
```

Then refresh the tool page.

### Index step fails — embedding error

Verify `nomic-embed-text` is installed:

```bash
ollama list | grep nomic
```

If missing:

```bash
ollama pull nomic-embed-text
```

### No resources detected in Tab 1

The retrieval found chunks but they didn't contain clear infrastructure keywords. Fix:

1. Check **Tab 4 — Retrieved Chunks** to see what was retrieved
2. Make infrastructure requirements more explicit in your doc (e.g. `"EC2 t3.medium"` not `"a server"`)
3. Re-index and re-run

### Terraform has syntax errors

Run `terraform validate` — it will point to the exact line. Common 1b model issues:

- Missing closing `}` brace
- Wrong attribute name (`vpc_id` vs `vpcid`)
- Missing `depends_on` for resource ordering

### Generation is very slow

```bash
free -h        # Linux — check available RAM
vm_stat        # macOS — check memory pressure
```

Close other apps and retry. The 1b model needs ~2 GB free RAM.

---

## Tips for Best Results

1. **Paste full docs freely** — RAG handles length; you don't need to trim
2. **Be explicit about sizes** — `"db.t3.small, 20GB, PostgreSQL 14"` beats `"a database"`
3. **One resource per line** in your doc gives cleaner extraction
4. **Check Tab 4** if something is missing — it shows which chunks were retrieved
5. **Re-run if output looks off** — LLM outputs are non-deterministic; a second run often improves results
6. **Re-index if you edit the doc** — the vector store is rebuilt fresh each time you click "index document"

---

## Architecture Summary

```
Browser (RAG Terraform Tool)
     │
     ├── /api/embeddings ──► nomic-embed-text   ← chunk your doc into vectors
     │
     ├── cosine similarity ──► top-4 chunks retrieved from memory
     │
     └── /api/generate ──► llama3.2:1b          ← generate with focused context
                                │
                                ▼
                  JSON resources → HCL Terraform → variables.tf
```

100% on-premises. No internet calls. No API keys. No data sent externally.

---

## Workshop Exercises

### Exercise 1 — Basic Web App (10 min)

Paste the sample prompt from Step 3 → index → generate → review all 4 tabs → copy and save `main.tf`

### Exercise 2 — Your Own Product (15 min)

Find deployment docs for a product your team uses → paste the full infrastructure section → compare the generated Terraform to your actual setup → check Tab 4 to see which chunks were retrieved

### Exercise 3 — Fix the Terraform (10 min)

Run `terraform init && terraform validate` on the generated code → identify and fix any issues → run `terraform plan`

### Exercise 4 — Long Document Test (10 min)

Paste a long README (500+ words) that includes non-infrastructure content → index it → observe how RAG retrieves only the infra-relevant chunks → compare output quality to the non-RAG version

---

## Reference

| Resource | URL |
|---|---|
| Ollama documentation | https://ollama.com/docs |
| nomic-embed-text model | https://ollama.com/library/nomic-embed-text |
| Llama 3.2 model page | https://ollama.com/library/llama3.2 |
| Terraform AWS provider | https://registry.terraform.io/providers/hashicorp/aws/latest |
| Terraform CLI install | https://developer.hashicorp.com/terraform/install |

---

*Workshop duration: ~45–60 minutes | Difficulty: Beginner–Intermediate | Prereq: Ollama + models already installed*
