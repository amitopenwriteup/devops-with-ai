#  Workshop Guide: AWS Terraform Generator with Llama 3.2:1b (On-Prem)

> **Goal:** Run an AI-powered tool locally that reads product installation guides and generates Terraform code — using Llama 3.2:1b entirely on your machine. No cloud API. No data leaves your laptop.

---

## Prerequisites

| Requirement | Minimum Spec |
|---|---|
| OS | macOS 12+, Ubuntu 20.04+, or Windows 11 with WSL2 |
| RAM | 4 GB free (8 GB recommended) |
| Disk | 2 GB free for model weights |
| CPU | Any modern x86-64 or Apple Silicon |
| GPU | Optional (CPU-only works fine for 1b model) |
| Browser | Chrome, Firefox, or Edge (latest) |

---

## Step 1 — Install Ollama

Ollama is the local runtime that serves Llama models via a REST API.

### macOS / Linux

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

### Windows (WSL2)

Open WSL2 terminal and run the same command above.

### Verify installation

```bash
ollama --version
# Expected: ollama version 0.x.x
```

---

## Step 2 — Pull Llama 3.2:1b Model

This is the smallest Llama 3.2 model (~1.3 GB). It runs on CPU with 4 GB RAM.

```bash
ollama pull llama3.2:1b
```

**Expected output:**
```
pulling manifest
pulling 74701a8c35f6... 100% ▕████████████████▏ 1.3 GB
verifying sha256 digest
writing manifest
success
```

>  Download time: ~5–10 minutes on a typical connection.

### Verify the model is available

```bash
ollama list
# Should show: llama3.2:1b
```

---

## Step 3 — Start Ollama Server

Ollama must be running in the background before using the tool.

```bash
# Allow browser access (required for Claude artifact to call localhost)
OLLAMA_ORIGINS="*" ollama serve
```

**Expected output:**
```
Listening on 127.0.0.1:11434 (version 0.x.x)
```

>  Keep this terminal open throughout the workshop. Open a new terminal for any other commands.

### Test the server

```bash
curl http://localhost:11434/api/tags
# Expected: JSON listing available models
```

---

## Step 4 — Test Llama 3.2:1b Locally

Before using the tool, do a quick sanity check:

```bash
ollama run llama3.2:1b "List 3 AWS services needed to host a web app"
```

You should get a short, coherent response within 10–30 seconds on CPU.

Type `/bye` to exit the interactive session.

---

## Step 5 — Open the Terraform Generator Tool

1. Go to the Claude chat where the **AWS Terraform Generator** artifact was shared
2. The tool loads directly in the browser — no installation needed

---

## Step 6 — Configure Ollama Settings in the Tool

In the tool UI:

| Field | Value |
|---|---|
| **Ollama base URL** | `http://localhost:11434` |
| **Model** | Select `llama3.2:1b` from dropdown (or type custom) |

>  If `llama3.2:1b` is not in the dropdown, select **custom...** and type `llama3.2:1b` manually.

---

## Step 7 — Paste a Product Installation Guide

In the **"product installation guide / prompt"** text area, paste any of the following:

### Option A — Use the sample prompt (quick test)

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

Copy-paste the **"Requirements"** or **"Infrastructure"** section from any product README, deployment doc, or AWS architecture guide.

---

## Step 8 — Run the Pipeline

Click **"analyze & generate terraform ↗"**

The tool runs 3 sequential calls to your local Llama model:

| Step | What happens | Typical time (1b, CPU) |
|---|---|---|
| 1. Resource extraction | Llama reads the guide and outputs structured JSON of AWS resources | 15–40 sec |
| 2. Terraform generation | Llama writes HCL code for all extracted resources | 30–90 sec |
| 3. Variables.tf | Llama extracts variables and writes `variables.tf` | 15–40 sec |

> Total: ~1–3 minutes on CPU. Be patient — 1b is lightweight but not instant.

---

## Step 9 — Review the Output

The tool shows 3 tabs:

### Tab 1 — AWS Resources

Colored chips showing each AWS service detected (EC2, RDS, S3, VPC, IAM, ALB, etc.).

**What to check:**
- Are all expected services shown?
- If a resource is missing, add it explicitly to your guide text and re-run.

### Tab 2 — Terraform Code

Complete HCL code including:
- `provider "aws"` block
- All resource blocks with dependencies
- Security group rules
- IAM roles and policies
- Outputs

Click **copy** to copy the code.

### Tab 3 — variables.tf

Extracted variables with types, descriptions, and default values.

---

## Step 10 — Save and Use the Terraform Code

### Save to files

```bash
mkdir my-terraform-project
cd my-terraform-project
```

Paste the **Terraform code** tab output into `main.tf`:

```bash
# Paste the copied code, then save
nano main.tf
```

Paste the **variables.tf** tab output:

```bash
nano variables.tf
```

Create a `terraform.tfvars` for your environment:

```hcl
# terraform.tfvars
aws_region   = "us-east-1"
environment  = "dev"
project_name = "my-app"
```

### Initialize and validate

```bash
terraform init
terraform validate
terraform plan
```

> Always review the plan output before applying. The 1b model may occasionally miss a dependency — fix manually if `terraform validate` reports errors.

---

## Troubleshooting

### "Could not reach Ollama" error in the tool

**Cause:** Ollama is not running, or CORS is blocked.

**Fix:**
```bash
# Stop any running Ollama, restart with CORS open
pkill ollama
OLLAMA_ORIGINS="*" ollama serve
```

### Response is empty or garbled

**Cause:** 1b model has limited context — your guide may be too long.

**Fix:** Trim the guide to under 500 words, focusing on infrastructure requirements only. Remove marketing copy, screenshots descriptions, and non-infra sections.

### Model is very slow

**Cause:** System RAM is low or other processes are competing.

**Fix:**
```bash
# Check available RAM
free -h        # Linux
vm_stat        # macOS

# Close other applications and retry
```

### Terraform code has syntax errors

**Cause:** 1b model occasionally produces slightly malformed HCL.

**Fix:** Run `terraform validate` and fix the specific line it points to. Common issues:
- Missing closing `}` brace
- Wrong attribute name (e.g., `vpc_id` vs `vpcid`)
- Missing `depends_on` for ordering

---

## Tips for Better Results with 1b

The 1b model has a small context window. These tips significantly improve output quality:

1. **Be specific in your guide** — write `"t3.medium EC2 instance"` not `"a server"`
2. **List one resource per line** — bullet-point format works better than paragraphs
3. **Include sizes and configs** — `"RDS db.t3.small, 20GB, PostgreSQL 14"` gives better Terraform than `"a database"`
4. **Keep it under 400 words** — trim non-infrastructure content before pasting
5. **Re-run if output looks wrong** — LLM outputs are non-deterministic; a second run often fixes issues

---

## Workshop Exercises

### Exercise 1 — Basic Web App (10 min)
Paste the sample prompt from Step 7 → review all 3 tabs → copy and save `main.tf`

### Exercise 2 — Your Own Product (15 min)
Find the deployment docs for a product your team uses → paste the infrastructure section → compare Llama's output to your actual setup

### Exercise 3 — Fix the Terraform (10 min)
Run `terraform init && terraform validate` on the generated code → identify and fix any issues → run `terraform plan`

---

## What Was Built (Architecture Summary)

```
Browser (Claude artifact)
    │
    │  HTTP POST /api/chat
    ▼
Ollama (localhost:11434)          ← 100% on-prem
    │
    │  inference
    ▼
Llama 3.2:1b weights              ← running on your CPU/GPU
    │
    ▼
JSON resources → HCL Terraform → variables.tf
```

No internet calls. No API keys. No data sent externally.

---

## Reference

| Resource | URL |
|---|---|
| Ollama documentation | https://ollama.com/docs |
| Llama 3.2 model page | https://ollama.com/library/llama3.2 |
| Terraform AWS provider docs | https://registry.terraform.io/providers/hashicorp/aws/latest |
| Terraform CLI install | https://developer.hashicorp.com/terraform/install |

---

*Workshop duration: ~45–60 minutes | Difficulty: Beginner–Intermediate*
