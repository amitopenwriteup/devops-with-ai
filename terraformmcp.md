# Workshop: Build a Terraform AI Provisioning Tool with Ollama
### Describe Your App · AI Detects AWS Resources · Auto-Generates `.tf` Files · 30-Day Cost Estimate

No Claude Desktop required. Everything runs from the terminal or browser.  
Model: **llama3.2:1b** (local, free, private)

---

## What We're Building

```
+------------------------+       +-------------------------+
|   cli.py  (terminal)   |       |  app.py  (browser UI)   |
|   interactive menu     |       |  http://localhost:5000   |
+-----------+------------+       +----------+--------------+
            |                               |
            +-------------+-----------------+
                          |
            +-------------v--------------+
            |         tools.py           |
            |  - parse_product_stack      |
            |  - decide_aws_resources     |
            |  - generate_tf_files        |
            |  - estimate_cost_30d        |
            +-------------+--------------+
                          |
            +-------------v--------------+
            |   Ollama  llama3.2:1b      |
            |   localhost:11434           |
            +----------------------------+
```

**Two interfaces, one brain:**
- **Option A** — `cli.py` : interactive terminal menu
- **Option B** — `app.py` : browser UI using Flask

---

## Project Structure

```
terraform-ai-tool/
├── tools.py            <- Core logic (Ollama + Terraform generation)
├── cli.py              <- Terminal interface (Option A)
├── app.py              <- Browser UI with Flask (Option B)
├── requirements.txt    <- Python dependencies
├── output/             <- Generated .tf files land here
└── README.md
```

---

## Prerequisites Checklist

| Tool | Version | Check Command |
|------|---------|---------------|
| Python | >= 3.11 | `python --version` |
| Ollama | Latest | `ollama --version` |
| pip | Latest | `pip --version` |
| Terraform (optional) | >= 1.5 | `terraform --version` |

> Terraform is optional for the workshop — we generate the files; you can apply them later.

---

## Part 1 — Setup and Installation

### Step 1.1 — Create the Project

```bash
mkdir terraform-ai-tool
cd terraform-ai-tool
python3 -m venv venv

# Activate
# macOS/Linux:
source venv/bin/activate
# Windows:
# venv\Scripts\activate

mkdir output
```

### Step 1.2 — Install Dependencies

Create `requirements.txt`:

```txt
httpx>=0.27.0
flask>=3.0.0
```

Install:

```bash
pip install -r requirements.txt
```

### Step 1.3 — Pull the Ollama Model

```bash
# Start Ollama (skip if already running)
ollama serve &

# Pull llama3.2:1b (~1.3 GB)
ollama pull llama3.2:1b

# Verify it works
ollama run llama3.2:1b "Say hello"
```

---

## Part 2 — The Product Installation Step

This is the key innovation. Before generating any Terraform, the user **describes what they are building** — their product stack. The AI reads the description and decides which AWS resources are needed.

### Input Examples

| What you type | What the AI infers |
|---|---|
| "Python Django app, PostgreSQL, image uploads, background jobs" | EC2, RDS PostgreSQL, S3, SQS, ElastiCache |
| "Static React site with REST API and user auth" | S3 + CloudFront, Lambda + API Gateway, Cognito, DynamoDB |
| "Node.js microservices with auto-scaling" | ECS Fargate, ALB, ECR, RDS, ElastiCache, SQS |
| "Data pipeline ingesting CSV files nightly" | S3, Lambda (trigger), Glue, Redshift, CloudWatch |

The `parse_product_stack` function sends the description to Ollama which returns structured JSON — the resource plan. Everything downstream (Terraform files, cost estimate) is generated from that plan.

---

## Part 3 — Build `tools.py`

Create `tools.py`:

```python
# tools.py
import httpx
import json
import os
import re
from pathlib import Path

OLLAMA_BASE_URL = "http://localhost:11434"
DEFAULT_MODEL   = "llama3.2:1b"
OUTPUT_DIR      = Path("output")
OUTPUT_DIR.mkdir(exist_ok=True)


# ─────────────────────────────────────────────
# Ollama helpers
# ─────────────────────────────────────────────

def ollama_chat(prompt: str, system: str = "", model: str = DEFAULT_MODEL) -> str:
    payload = {"model": model, "prompt": prompt, "stream": False}
    if system:
        payload["system"] = system
    try:
        with httpx.Client(timeout=180.0) as client:
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


def _extract_json(text: str) -> dict:
    """Pull the first JSON object from a string."""
    match = re.search(r'\{.*\}', text, re.DOTALL)
    if match:
        try:
            return json.loads(match.group())
        except json.JSONDecodeError:
            pass
    return {}


# ─────────────────────────────────────────────
# Step 1 — Parse product description → resource plan
# ─────────────────────────────────────────────

def parse_product_stack(description: str, model: str = DEFAULT_MODEL) -> dict:
    """
    Send the user's product description to Ollama.
    Returns a structured dict of AWS resources to provision.
    """
    system = (
        "You are an AWS solutions architect. "
        "Analyse the product description and return ONLY a JSON object. "
        "No markdown, no explanation. Valid JSON only."
    )
    prompt = f"""
Product description:
\"{description}\"

Return a JSON object with this exact structure:
{{
  "app_name": "short-slug-name",
  "summary": "one sentence summary",
  "resources": {{
    "ec2": {{"needed": true/false, "instance_type": "t3.micro", "count": 1, "reason": "..."}},
    "rds": {{"needed": true/false, "engine": "postgres|mysql|none", "instance_class": "db.t3.micro", "reason": "..."}},
    "s3": {{"needed": true/false, "buckets": ["assets", "backups"], "reason": "..."}},
    "lambda": {{"needed": true/false, "count": 1, "runtime": "python3.11|nodejs20.x|none", "reason": "..."}},
    "api_gateway": {{"needed": true/false, "type": "HTTP|REST|none", "reason": "..."}},
    "dynamodb": {{"needed": true/false, "tables": ["sessions"], "reason": "..."}},
    "ecs": {{"needed": true/false, "launch_type": "FARGATE|EC2|none", "count": 1, "reason": "..."}},
    "elasticache": {{"needed": true/false, "engine": "redis|memcached|none", "reason": "..."}},
    "sqs": {{"needed": true/false, "queues": ["jobs"], "reason": "..."}},
    "cloudfront": {{"needed": true/false, "reason": "..."}},
    "cognito": {{"needed": true/false, "reason": "..."}},
    "alb": {{"needed": true/false, "reason": "..."}}
  }},
  "region": "us-east-1",
  "environment": "production"
}}
"""
    raw = ollama_chat(prompt, system=system, model=model)
    plan = _extract_json(raw)

    if not plan:
        # Fallback: build a minimal plan
        plan = _fallback_plan(description)

    return plan


def _fallback_plan(description: str) -> dict:
    desc_lower = description.lower()
    resources = {
        "ec2":          {"needed": any(k in desc_lower for k in ["ec2", "server", "vm", "django", "flask", "rails", "express"]),       "instance_type": "t3.micro",    "count": 1,   "reason": "Application server"},
        "rds":          {"needed": any(k in desc_lower for k in ["postgres", "mysql", "database", "db", "rds"]),                        "engine": "postgres",           "instance_class": "db.t3.micro", "reason": "Relational database"},
        "s3":           {"needed": any(k in desc_lower for k in ["s3", "upload", "file", "image", "static", "storage"]),               "buckets": ["assets"],          "reason": "Object storage"},
        "lambda":       {"needed": any(k in desc_lower for k in ["lambda", "serverless", "function", "trigger", "event"]),             "count": 1, "runtime": "python3.11", "reason": "Serverless function"},
        "api_gateway":  {"needed": any(k in desc_lower for k in ["api", "rest", "gateway", "endpoint"]),                               "type": "HTTP",                 "reason": "API endpoint"},
        "dynamodb":     {"needed": any(k in desc_lower for k in ["dynamo", "nosql", "key-value"]),                                      "tables": ["items"],            "reason": "NoSQL table"},
        "ecs":          {"needed": any(k in desc_lower for k in ["ecs", "container", "docker", "fargate", "microservice"]),            "launch_type": "FARGATE",       "count": 1, "reason": "Container orchestration"},
        "elasticache":  {"needed": any(k in desc_lower for k in ["redis", "cache", "elasticache", "session", "queue"]),                "engine": "redis",              "reason": "Caching layer"},
        "sqs":          {"needed": any(k in desc_lower for k in ["sqs", "queue", "job", "worker", "background", "async"]),             "queues": ["jobs"],             "reason": "Message queue"},
        "cloudfront":   {"needed": any(k in desc_lower for k in ["cdn", "cloudfront", "static", "react", "next", "vue", "angular"]),   "reason": "CDN distribution"},
        "cognito":      {"needed": any(k in desc_lower for k in ["auth", "cognito", "login", "user", "signup", "jwt"]),                "reason": "User authentication"},
        "alb":          {"needed": any(k in desc_lower for k in ["alb", "load balancer", "lb", "scaling", "ecs", "fargate"]),          "reason": "Load balancing"},
    }
    return {
        "app_name":    "my-app",
        "summary":     description[:80],
        "resources":   resources,
        "region":      "us-east-1",
        "environment": "production",
    }


# ─────────────────────────────────────────────
# Step 2 — Pretty-print the resource decision
# ─────────────────────────────────────────────

def decide_aws_resources(plan: dict) -> str:
    lines = [
        f"App:         {plan.get('app_name', 'my-app')}",
        f"Summary:     {plan.get('summary', '')}",
        f"Region:      {plan.get('region', 'us-east-1')}",
        f"Environment: {plan.get('environment', 'production')}",
        "",
        "AWS Resources selected:",
    ]
    for name, cfg in plan.get("resources", {}).items():
        if cfg.get("needed"):
            reason = cfg.get("reason", "")
            lines.append(f"  ✓  {name.upper():15s}  —  {reason}")
    return "\n".join(lines)


# ─────────────────────────────────────────────
# Step 3 — Generate Terraform files
# ─────────────────────────────────────────────

def generate_tf_files(plan: dict, model: str = DEFAULT_MODEL) -> dict:
    """
    Generate one .tf file per active resource group.
    Returns dict of {filename: content}.
    """
    app     = plan.get("app_name", "my-app")
    region  = plan.get("region", "us-east-1")
    env     = plan.get("environment", "production")
    res     = plan.get("resources", {})
    files   = {}

    # Always generate main.tf + variables.tf + outputs.tf
    files["main.tf"]      = _gen_main(app, region, env)
    files["variables.tf"] = _gen_variables(app, region, env)
    files["outputs.tf"]   = _gen_outputs(res, app)

    # Per-resource files
    generators = {
        "ec2":         (_gen_ec2,         res.get("ec2",         {})),
        "rds":         (_gen_rds,         res.get("rds",         {})),
        "s3":          (_gen_s3,          res.get("s3",          {})),
        "lambda":      (_gen_lambda,      res.get("lambda",      {})),
        "api_gateway": (_gen_api_gateway, res.get("api_gateway", {})),
        "dynamodb":    (_gen_dynamodb,    res.get("dynamodb",    {})),
        "ecs":         (_gen_ecs,         res.get("ecs",         {})),
        "elasticache": (_gen_elasticache, res.get("elasticache", {})),
        "sqs":         (_gen_sqs,         res.get("sqs",         {})),
        "cloudfront":  (_gen_cloudfront,  res.get("cloudfront",  {})),
        "cognito":     (_gen_cognito,     res.get("cognito",     {})),
        "alb":         (_gen_alb,         res.get("alb",         {})),
    }

    active = []
    for key, (fn, cfg) in generators.items():
        if cfg.get("needed"):
            fname = f"{key}.tf"
            files[fname] = fn(cfg, app, env)
            active.append(key)

    # AI enhancement pass — ask Ollama to add tags, descriptions
    enhanced = _ai_enhance_tf(files, plan, model)
    files.update(enhanced)

    # Save to disk
    app_dir = OUTPUT_DIR / app
    app_dir.mkdir(exist_ok=True)
    for fname, content in files.items():
        (app_dir / fname).write_text(content)

    return files


def _ai_enhance_tf(files: dict, plan: dict, model: str) -> dict:
    """Ask Ollama to add comments and improve a sample file."""
    sample_key = next((k for k in files if k not in ("main.tf", "variables.tf", "outputs.tf")), None)
    if not sample_key:
        return {}

    system = "You are a senior DevOps engineer. Return only valid Terraform HCL. No markdown."
    prompt = (
        f"Improve this Terraform file with helpful inline comments, "
        f"tags, and best-practice suggestions. "
        f"App: {plan.get('app_name')}, Environment: {plan.get('environment')}.\n\n"
        f"Original {sample_key}:\n\n{files[sample_key]}"
    )
    improved = ollama_chat(prompt, system=system, model=model)
    if improved and not improved.startswith("[ERROR]"):
        return {f"ai_{sample_key}": improved}
    return {}


# ─────────────────────────────────────────────
# Step 4 — 30-day cost estimate
# ─────────────────────────────────────────────

# AWS pricing constants (USD/month, approximate on-demand us-east-1)
PRICING = {
    "ec2": {
        "t3.micro":   8.47,   "t3.small":  16.94,  "t3.medium":  33.87,
        "t3.large":   67.74,  "t3.xlarge": 135.48, "t3.2xlarge": 270.96,
        "m5.large":   69.12,  "m5.xlarge": 138.24, "c5.large":   61.20,
    },
    "rds": {
        "db.t3.micro":  15.33,  "db.t3.small":  30.66, "db.t3.medium": 61.32,
        "db.t3.large":  122.64, "db.r5.large":  175.20,
    },
    "elasticache": {
        "cache.t3.micro":  13.68, "cache.t3.small": 27.36, "cache.t3.medium": 54.72,
    },
    "s3_per_bucket":   2.30,
    "lambda_per_fn":   0.20,
    "api_gateway":     3.50,
    "dynamodb_per_table": 1.00,
    "ecs_fargate_per_task": 14.40,
    "cloudfront":     10.00,
    "cognito":         0.00,   # free tier covers most dev workloads
    "sqs_per_queue":   0.40,
    "alb":             16.20,
    "nat_gateway":     32.40,
    "data_transfer":   9.00,
}

def estimate_cost_30d(plan: dict) -> dict:
    res       = plan.get("resources", {})
    breakdown = {}
    total     = 0.0

    def add(label, cost):
        nonlocal total
        if cost > 0:
            breakdown[label] = round(cost, 2)
            total += cost

    # EC2
    ec2 = res.get("ec2", {})
    if ec2.get("needed"):
        itype = ec2.get("instance_type", "t3.micro")
        count = int(ec2.get("count", 1))
        price = PRICING["ec2"].get(itype, 8.47) * count
        add(f"EC2 {itype} x{count}", price)

    # RDS
    rds = res.get("rds", {})
    if rds.get("needed"):
        iclass = rds.get("instance_class", "db.t3.micro")
        price  = PRICING["rds"].get(iclass, 15.33)
        add(f"RDS {rds.get('engine','postgres')} {iclass}", price)
        add("RDS Storage 20GB gp2", 2.30)

    # S3
    s3 = res.get("s3", {})
    if s3.get("needed"):
        buckets = s3.get("buckets", ["assets"])
        add(f"S3 ({len(buckets)} bucket(s), 50GB est.)", PRICING["s3_per_bucket"] * len(buckets) + 1.15)

    # Lambda
    lam = res.get("lambda", {})
    if lam.get("needed"):
        count = int(lam.get("count", 1))
        add(f"Lambda x{count} (1M req/mo est.)", PRICING["lambda_per_fn"] * count)

    # API Gateway
    apigw = res.get("api_gateway", {})
    if apigw.get("needed"):
        add(f"API Gateway {apigw.get('type','HTTP')}", PRICING["api_gateway"])

    # DynamoDB
    ddb = res.get("dynamodb", {})
    if ddb.get("needed"):
        tables = ddb.get("tables", ["items"])
        add(f"DynamoDB ({len(tables)} table(s))", PRICING["dynamodb_per_table"] * len(tables))

    # ECS
    ecs = res.get("ecs", {})
    if ecs.get("needed"):
        count = int(ecs.get("count", 1))
        add(f"ECS Fargate x{count} task(s)", PRICING["ecs_fargate_per_task"] * count)

    # ElastiCache
    ec = res.get("elasticache", {})
    if ec.get("needed"):
        add(f"ElastiCache {ec.get('engine','redis')} cache.t3.micro", PRICING["elasticache"]["cache.t3.micro"])

    # SQS
    sqs = res.get("sqs", {})
    if sqs.get("needed"):
        queues = sqs.get("queues", ["jobs"])
        add(f"SQS ({len(queues)} queue(s))", PRICING["sqs_per_queue"] * len(queues))

    # CloudFront
    cf = res.get("cloudfront", {})
    if cf.get("needed"):
        add("CloudFront (100GB transfer est.)", PRICING["cloudfront"])

    # Cognito
    cog = res.get("cognito", {})
    if cog.get("needed"):
        add("Cognito User Pool (free tier)", PRICING["cognito"])

    # ALB
    alb = res.get("alb", {})
    if alb.get("needed"):
        add("ALB (Application Load Balancer)", PRICING["alb"])

    # Always add data transfer + NAT (if any private resources)
    private_resources = any([
        res.get("ec2", {}).get("needed"),
        res.get("rds", {}).get("needed"),
        res.get("ecs", {}).get("needed"),
        res.get("elasticache", {}).get("needed"),
    ])
    if private_resources:
        add("NAT Gateway (1 AZ)", PRICING["nat_gateway"])
    add("Data Transfer (10GB est.)", PRICING["data_transfer"])

    return {
        "breakdown": breakdown,
        "total_30d":  round(total, 2),
        "total_daily": round(total / 30, 2),
        "note": "Estimates based on us-east-1 on-demand pricing. Savings Plans / Reserved Instances can reduce by 30–60%.",
    }


def format_cost_report(cost: dict) -> str:
    lines = ["", "╔══ 30-Day AWS Cost Estimate ══════════════════════╗"]
    for label, amount in cost["breakdown"].items():
        lines.append(f"  ${amount:>8.2f}   {label}")
    lines.append("  " + "─" * 46)
    lines.append(f"  ${cost['total_30d']:>8.2f}   TOTAL / 30 days")
    lines.append(f"  ${cost['total_daily']:>8.2f}   per day")
    lines.append("╚══════════════════════════════════════════════════╝")
    lines.append(f"\n  ℹ  {cost['note']}")
    return "\n".join(lines)


# ─────────────────────────────────────────────
# Terraform file generators
# ─────────────────────────────────────────────

def _gen_main(app, region, env):
    return f'''terraform {{
  required_version = ">= 1.5.0"
  required_providers {{
    aws = {{
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }}
  }}
}}

provider "aws" {{
  region = var.aws_region

  default_tags {{
    tags = {{
      Project     = "{app}"
      Environment = "{env}"
      ManagedBy   = "Terraform"
    }}
  }}
}}
'''

def _gen_variables(app, region, env):
    return f'''variable "app_name" {{
  description = "Application name"
  type        = string
  default     = "{app}"
}}

variable "environment" {{
  description = "Deployment environment"
  type        = string
  default     = "{env}"
}}

variable "aws_region" {{
  description = "AWS region"
  type        = string
  default     = "{region}"
}}

variable "vpc_cidr" {{
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}}
'''

def _gen_outputs(resources, app):
    lines = ['# outputs.tf — export key resource identifiers\n']
    if resources.get("ec2", {}).get("needed"):
        lines.append('output "ec2_public_ip" {\n  value = aws_instance.app.public_ip\n}\n')
    if resources.get("rds", {}).get("needed"):
        lines.append('output "rds_endpoint" {\n  value = aws_db_instance.main.endpoint\n}\n')
    if resources.get("s3", {}).get("needed"):
        lines.append('output "s3_bucket_name" {\n  value = aws_s3_bucket.assets.bucket\n}\n')
    if resources.get("alb", {}).get("needed"):
        lines.append('output "alb_dns_name" {\n  value = aws_lb.main.dns_name\n}\n')
    if resources.get("cloudfront", {}).get("needed"):
        lines.append('output "cloudfront_domain" {\n  value = aws_cloudfront_distribution.cdn.domain_name\n}\n')
    return "\n".join(lines)

def _gen_ec2(cfg, app, env):
    itype = cfg.get("instance_type", "t3.micro")
    count = cfg.get("count", 1)
    return f'''# EC2 — Application server
data "aws_ami" "amazon_linux_2023" {{
  most_recent = true
  owners      = ["amazon"]
  filter {{
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }}
}}

resource "aws_instance" "app" {{
  count         = {count}
  ami           = data.aws_ami.amazon_linux_2023.id
  instance_type = "{itype}"
  key_name      = "{app}-{env}-key"

  vpc_security_group_ids = [aws_security_group.ec2.id]
  subnet_id              = aws_subnet.public[0].id

  root_block_device {{
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }}

  tags = {{
    Name = "{app}-{env}-server-${{count.index + 1}}"
  }}
}}

resource "aws_security_group" "ec2" {{
  name_prefix = "{app}-ec2-"
  vpc_id      = aws_vpc.main.id

  ingress {{
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }}
  ingress {{
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }}
  egress {{
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }}
}}

# VPC (shared — referenced by other resources)
resource "aws_vpc" "main" {{
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true
  tags = {{ Name = "{app}-{env}-vpc" }}
}}

resource "aws_subnet" "public" {{
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = {{ Name = "{app}-{env}-public-${{count.index + 1}}" }}
}}

data "aws_availability_zones" "available" {{}}
'''

def _gen_rds(cfg, app, env):
    engine  = cfg.get("engine", "postgres")
    iclass  = cfg.get("instance_class", "db.t3.micro")
    port    = 5432 if engine == "postgres" else 3306
    version = "15" if engine == "postgres" else "8.0"
    return f'''# RDS — Managed relational database ({engine})
resource "aws_db_instance" "main" {{
  identifier        = "{app}-{env}-db"
  engine            = "{engine}"
  engine_version    = "{version}"
  instance_class    = "{iclass}"
  allocated_storage = 20
  storage_type      = "gp3"
  storage_encrypted = true

  db_name  = "{app.replace("-", "_")}"
  username = "{app.replace("-", "_")}_user"
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period = 7
  skip_final_snapshot     = false
  final_snapshot_identifier = "{app}-{env}-final"
  deletion_protection     = true

  tags = {{ Name = "{app}-{env}-db" }}
}}

resource "aws_db_subnet_group" "main" {{
  name       = "{app}-{env}-db-subnet"
  subnet_ids = aws_subnet.private[*].id
}}

resource "aws_subnet" "private" {{
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = {{ Name = "{app}-{env}-private-${{count.index + 1}}" }}
}}

resource "aws_security_group" "rds" {{
  name_prefix = "{app}-rds-"
  vpc_id      = aws_vpc.main.id
  ingress {{
    from_port       = {port}
    to_port         = {port}
    protocol        = "tcp"
    security_groups = [aws_security_group.ec2.id]
  }}
}}

variable "db_password" {{
  description = "RDS master password"
  type        = string
  sensitive   = true
}}
'''

def _gen_s3(cfg, app, env):
    buckets = cfg.get("buckets", ["assets"])
    blocks  = []
    for b in buckets:
        bname = f"{app}-{env}-{b}"
        blocks.append(f'''resource "aws_s3_bucket" "{b}" {{
  bucket = "{bname}"
  tags   = {{ Name = "{bname}", Purpose = "{b}" }}
}}

resource "aws_s3_bucket_versioning" "{b}" {{
  bucket = aws_s3_bucket.{b}.id
  versioning_configuration {{ status = "Enabled" }}
}}

resource "aws_s3_bucket_server_side_encryption_configuration" "{b}" {{
  bucket = aws_s3_bucket.{b}.id
  rule {{
    apply_server_side_encryption_by_default {{ sse_algorithm = "AES256" }}
  }}
}}

resource "aws_s3_bucket_public_access_block" "{b}" {{
  bucket                  = aws_s3_bucket.{b}.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}}
''')
    return "\n".join(blocks)

def _gen_lambda(cfg, app, env):
    runtime = cfg.get("runtime", "python3.11")
    count   = int(cfg.get("count", 1))
    fns     = []
    for i in range(count):
        fname = f"fn{i+1}" if count > 1 else "fn"
        fns.append(f'''resource "aws_lambda_function" "{fname}" {{
  function_name = "{app}-{env}-{fname}"
  role          = aws_iam_role.lambda_exec.arn
  runtime       = "{runtime}"
  handler       = "handler.lambda_handler"
  filename      = "${{path.module}}/lambda_{fname}.zip"
  timeout       = 30
  memory_size   = 256

  environment {{
    variables = {{
      ENVIRONMENT = "{env}"
      APP_NAME    = "{app}"
    }}
  }}
}}
''')
    fns.append(f'''resource "aws_iam_role" "lambda_exec" {{
  name = "{app}-{env}-lambda-role"
  assume_role_policy = jsonencode({{
    Version = "2012-10-17"
    Statement = [{{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = {{ Service = "lambda.amazonaws.com" }}
    }}]
  }})
}}

resource "aws_iam_role_policy_attachment" "lambda_basic" {{
  role       = aws_iam_role.lambda_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}}
''')
    return "\n".join(fns)

def _gen_api_gateway(cfg, app, env):
    gtype = cfg.get("type", "HTTP")
    return f'''# API Gateway ({gtype})
resource "aws_apigatewayv2_api" "main" {{
  name          = "{app}-{env}-api"
  protocol_type = "{gtype}"
  cors_configuration {{
    allow_origins = ["*"]
    allow_methods = ["GET", "POST", "PUT", "DELETE", "OPTIONS"]
    allow_headers = ["Content-Type", "Authorization"]
  }}
}}

resource "aws_apigatewayv2_stage" "default" {{
  api_id      = aws_apigatewayv2_api.main.id
  name        = "$default"
  auto_deploy = true
}}
'''

def _gen_dynamodb(cfg, app, env):
    tables = cfg.get("tables", ["items"])
    blocks = []
    for t in tables:
        blocks.append(f'''resource "aws_dynamodb_table" "{t}" {{
  name         = "{app}-{env}-{t}"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "id"

  attribute {{
    name = "id"
    type = "S"
  }}

  point_in_time_recovery {{ enabled = true }}
  server_side_encryption {{ enabled = true }}

  tags = {{ Name = "{app}-{env}-{t}" }}
}}
''')
    return "\n".join(blocks)

def _gen_ecs(cfg, app, env):
    launch = cfg.get("launch_type", "FARGATE")
    count  = int(cfg.get("count", 1))
    return f'''# ECS — Container orchestration ({launch})
resource "aws_ecs_cluster" "main" {{
  name = "{app}-{env}-cluster"
  setting {{
    name  = "containerInsights"
    value = "enabled"
  }}
}}

resource "aws_ecs_task_definition" "app" {{
  family                   = "{app}-{env}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["{launch}"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_exec.arn

  container_definitions = jsonencode([{{
    name  = "{app}"
    image = "${{var.ecr_image_uri}}"
    portMappings = [{{ containerPort = 8080, protocol = "tcp" }}]
    environment = [
      {{ name = "ENVIRONMENT", value = "{env}" }}
    ]
    logConfiguration = {{
      logDriver = "awslogs"
      options = {{
        "awslogs-group"         = "/ecs/{app}-{env}"
        "awslogs-region"        = var.aws_region
        "awslogs-stream-prefix" = "app"
      }}
    }}
  }}])
}}

resource "aws_ecs_service" "app" {{
  name            = "{app}-{env}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = {count}
  launch_type     = "{launch}"

  network_configuration {{
    subnets         = aws_subnet.private[*].id
    security_groups = [aws_security_group.ecs.id]
  }}
}}

resource "aws_security_group" "ecs" {{
  name_prefix = "{app}-ecs-"
  vpc_id      = aws_vpc.main.id
  egress {{
    from_port   = 0; to_port = 0; protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }}
}}

resource "aws_iam_role" "ecs_exec" {{
  name = "{app}-{env}-ecs-exec"
  assume_role_policy = jsonencode({{
    Version = "2012-10-17"
    Statement = [{{
      Action = "sts:AssumeRole"; Effect = "Allow"
      Principal = {{ Service = "ecs-tasks.amazonaws.com" }}
    }}]
  }})
}}

resource "aws_iam_role_policy_attachment" "ecs_exec_policy" {{
  role       = aws_iam_role.ecs_exec.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}}

resource "aws_cloudwatch_log_group" "ecs" {{
  name              = "/ecs/{app}-{env}"
  retention_in_days = 14
}}

variable "ecr_image_uri" {{
  description = "ECR image URI for the ECS task"
  type        = string
}}
'''

def _gen_elasticache(cfg, app, env):
    engine = cfg.get("engine", "redis")
    return f'''# ElastiCache — {engine.capitalize()} caching
resource "aws_elasticache_subnet_group" "main" {{
  name       = "{app}-{env}-cache-subnet"
  subnet_ids = aws_subnet.private[*].id
}}

resource "aws_elasticache_cluster" "main" {{
  cluster_id           = "{app}-{env}-cache"
  engine               = "{engine}"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "{engine}.7" if "{engine}" == "redis" else "default.memcached1.6"
  port                 = 6379 if "{engine}" == "redis" else 11211
  subnet_group_name    = aws_elasticache_subnet_group.main.name
  security_group_ids   = [aws_security_group.cache.id]
}}

resource "aws_security_group" "cache" {{
  name_prefix = "{app}-cache-"
  vpc_id      = aws_vpc.main.id
  ingress {{
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.ec2.id]
  }}
}}
'''

def _gen_sqs(cfg, app, env):
    queues = cfg.get("queues", ["jobs"])
    blocks = []
    for q in queues:
        blocks.append(f'''resource "aws_sqs_queue" "{q}" {{
  name                      = "{app}-{env}-{q}"
  message_retention_seconds = 86400
  visibility_timeout_seconds = 30
  receive_wait_time_seconds  = 20

  redrive_policy = jsonencode({{
    deadLetterTargetArn = aws_sqs_queue.{q}_dlq.arn
    maxReceiveCount     = 5
  }})
}}

resource "aws_sqs_queue" "{q}_dlq" {{
  name = "{app}-{env}-{q}-dlq"
}}
''')
    return "\n".join(blocks)

def _gen_cloudfront(cfg, app, env):
    return f'''# CloudFront — CDN distribution
resource "aws_cloudfront_distribution" "cdn" {{
  enabled             = true
  default_root_object = "index.html"
  price_class         = "PriceClass_100"

  origin {{
    domain_name              = aws_s3_bucket.assets.bucket_regional_domain_name
    origin_id                = "s3-assets"
    origin_access_control_id = aws_cloudfront_origin_access_control.main.id
  }}

  default_cache_behavior {{
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "s3-assets"
    viewer_protocol_policy = "redirect-to-https"
    compress               = true
    cache_policy_id        = "658327ea-f89d-4fab-a63d-7e88639e58f6" # CachingOptimized
  }}

  restrictions {{
    geo_restriction {{ restriction_type = "none" }}
  }}

  viewer_certificate {{
    cloudfront_default_certificate = true
  }}
}}

resource "aws_cloudfront_origin_access_control" "main" {{
  name                              = "{app}-{env}-oac"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}}
'''

def _gen_cognito(cfg, app, env):
    return f'''# Cognito — User authentication
resource "aws_cognito_user_pool" "main" {{
  name = "{app}-{env}-users"

  password_policy {{
    minimum_length    = 8
    require_lowercase = true
    require_uppercase = true
    require_numbers   = true
    require_symbols   = false
  }}

  auto_verified_attributes = ["email"]

  account_recovery_setting {{
    recovery_mechanism {{
      name     = "verified_email"
      priority = 1
    }}
  }}
}}

resource "aws_cognito_user_pool_client" "app" {{
  name         = "{app}-{env}-client"
  user_pool_id = aws_cognito_user_pool.main.id

  explicit_auth_flows = [
    "ALLOW_USER_PASSWORD_AUTH",
    "ALLOW_REFRESH_TOKEN_AUTH",
    "ALLOW_USER_SRP_AUTH",
  ]

  token_validity_units {{
    access_token  = "hours"
    refresh_token = "days"
  }}
  access_token_validity  = 1
  refresh_token_validity = 30
}}
'''

def _gen_alb(cfg, app, env):
    return f'''# ALB — Application Load Balancer
resource "aws_lb" "main" {{
  name               = "{app}-{env}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true
  enable_http2               = true
}}

resource "aws_lb_target_group" "app" {{
  name        = "{app}-{env}-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"

  health_check {{
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
  }}
}}

resource "aws_lb_listener" "http" {{
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {{
    type = "redirect"
    redirect {{
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }}
  }}
}}

resource "aws_security_group" "alb" {{
  name_prefix = "{app}-alb-"
  vpc_id      = aws_vpc.main.id
  ingress {{
    from_port = 80;  to_port = 80;  protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"]
  }}
  ingress {{
    from_port = 443; to_port = 443; protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"]
  }}
  egress {{
    from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"]
  }}
}}
'''


# ─────────────────────────────────────────────
# Full pipeline (convenience)
# ─────────────────────────────────────────────

def full_pipeline(description: str, model: str = DEFAULT_MODEL) -> dict:
    plan     = parse_product_stack(description, model)
    decision = decide_aws_resources(plan)
    files    = generate_tf_files(plan, model)
    cost     = estimate_cost_30d(plan)
    return {
        "plan":     plan,
        "decision": decision,
        "files":    files,
        "cost":     cost,
    }
```

---

## Part 4 — Option A: Terminal CLI

Create `cli.py`:

```python
# cli.py
import sys
import tools

DIVIDER = "─" * 62

def prompt(msg: str) -> str:
    return input(f"\n  {msg}: ").strip()

def show(title: str, content: str):
    print(f"\n{DIVIDER}")
    print(f"  {title}")
    print(DIVIDER)
    print(content)

def menu():
    print(f"\n{DIVIDER}")
    print("  Terraform AI Provisioner  —  llama3.2:1b via Ollama")
    print(DIVIDER)
    print("  1.  Describe product → detect AWS resources")
    print("  2.  Generate Terraform files (from last plan)")
    print("  3.  Show 30-day cost estimate (from last plan)")
    print("  4.  Full pipeline  (1 + 2 + 3 in one shot)")
    print("  5.  List Ollama models")
    print("  0.  Exit")
    return input("\n  Choose: ").strip()

def main():
    last_plan = None

    while True:
        choice = menu()

        if choice == "0":
            print("  Goodbye.")
            sys.exit(0)

        elif choice == "1":
            desc = prompt("Describe your product / tech stack")
            if not desc:
                print("  Please enter a description.")
                continue
            print("\n  Sending to Ollama... (may take 10–30s)")
            last_plan = tools.parse_product_stack(desc)
            decision  = tools.decide_aws_resources(last_plan)
            show("AWS Resource Plan", decision)

        elif choice == "2":
            if not last_plan:
                print("\n  Run option 1 first to create a plan.")
                continue
            print("\n  Generating Terraform files...")
            files = tools.generate_tf_files(last_plan)
            app   = last_plan.get("app_name", "my-app")
            show("Generated Terraform Files", "\n".join(f"  output/{app}/{f}" for f in files))
            print(f"\n  ✓  Files saved to ./output/{app}/")

        elif choice == "3":
            if not last_plan:
                print("\n  Run option 1 first to create a plan.")
                continue
            cost   = tools.estimate_cost_30d(last_plan)
            report = tools.format_cost_report(cost)
            show("30-Day Cost Estimate", report)

        elif choice == "4":
            desc = prompt("Describe your product / tech stack")
            if not desc:
                continue
            print("\n  Running full pipeline (this may take 30–60s)...")
            result    = tools.full_pipeline(desc)
            last_plan = result["plan"]

            show("AWS Resource Plan", result["decision"])

            app   = last_plan.get("app_name", "my-app")
            files = result["files"]
            print(f"\n  Terraform files generated ({len(files)} files):")
            for f in files:
                print(f"    output/{app}/{f}")

            report = tools.format_cost_report(result["cost"])
            show("30-Day Cost Estimate", report)

        elif choice == "5":
            models = tools.list_models()
            show("Available Ollama Models",
                 "\n  ".join(models) if models else "  No models found. Run: ollama pull llama3.2:1b")

        else:
            print("  Invalid choice.")

if __name__ == "__main__":
    main()
```

Run:

```bash
python cli.py
```

---

## Part 5 — Option B: Browser UI

Create `app.py`:

```python
# app.py
from flask import Flask, request, jsonify, render_template_string
import tools, json

app = Flask(__name__)

HTML = """
<!DOCTYPE html>
<html>
<head>
  <title>Terraform AI Provisioner</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Courier New', monospace; background: #0f1117; color: #e0e0e0;
           max-width: 980px; margin: 0 auto; padding: 30px 20px; }
    h1   { font-size: 1.3em; color: #4fc3f7; border-bottom: 1px solid #333;
           padding-bottom: 10px; margin-bottom: 20px; }
    .subtitle { font-size: 0.8em; color: #888; margin-bottom: 24px; }
    textarea { width: 100%; height: 80px; background: #1a1d27; color: #e0e0e0;
               border: 1px solid #333; padding: 10px; font-family: monospace;
               font-size: 0.9em; resize: vertical; }
    .btn-row { display: flex; gap: 8px; flex-wrap: wrap; margin: 12px 0; }
    button { padding: 9px 18px; background: #1565c0; color: #fff; border: none;
             cursor: pointer; font-family: monospace; font-size: 0.85em;
             border-radius: 2px; }
    button:hover  { background: #1976d2; }
    button.accent { background: #2e7d32; }
    button.accent:hover { background: #388e3c; }
    .panel { background: #1a1d27; border: 1px solid #2a2d3a; margin-top: 16px;
             border-radius: 4px; overflow: hidden; }
    .panel-header { background: #21253a; padding: 8px 14px; font-size: 0.8em;
                    color: #90caf9; text-transform: uppercase; letter-spacing: 1px; }
    pre  { padding: 14px; white-space: pre-wrap; word-break: break-word;
           font-size: 0.85em; max-height: 420px; overflow-y: auto; color: #cfd8dc; }
    .cost-box { color: #a5d6a7; }
    .spinner { display: none; color: #ffd54f; padding: 12px; }
    .file-list { padding: 12px 14px; }
    .file-item { cursor: pointer; color: #80cbc4; padding: 3px 0; font-size: 0.85em; }
    .file-item:hover { color: #fff; }
    .tabs { display: flex; gap: 0; border-bottom: 1px solid #333; margin-top: 16px; }
    .tab  { padding: 8px 16px; cursor: pointer; font-size: 0.82em; color: #888;
             border-bottom: 2px solid transparent; }
    .tab.active { color: #4fc3f7; border-bottom-color: #4fc3f7; }
    #tab-plan, #tab-tf, #tab-cost { display: none; }
  </style>
</head>
<body>
  <h1>⚡ Terraform AI Provisioner</h1>
  <p class="subtitle">Describe your product → AI picks AWS resources → generates .tf files → estimates 30-day cost</p>
  <p class="subtitle">Model: <strong style="color:#4fc3f7">llama3.2:1b</strong> via Ollama (local)</p>

  <textarea id="desc" placeholder="Describe your product stack.&#10;Examples:&#10;  • Python Django app with PostgreSQL, Redis, image uploads, and async workers&#10;  • Static React site with a Node.js REST API and user authentication&#10;  • Containerised microservices on Docker with auto-scaling and a message queue"></textarea>

  <div class="btn-row">
    <button onclick="runStep('plan')">1 · Detect Resources</button>
    <button onclick="runStep('tf')">2 · Generate Terraform</button>
    <button onclick="runStep('cost')">3 · Cost Estimate</button>
    <button class="accent" onclick="runStep('full')">⚡ Full Pipeline</button>
  </div>
  <div class="spinner" id="spinner">⏳ Ollama is thinking…</div>

  <div class="tabs">
    <div class="tab active" onclick="switchTab('plan')">Resource Plan</div>
    <div class="tab"        onclick="switchTab('tf')">Terraform Files</div>
    <div class="tab"        onclick="switchTab('cost')">Cost Estimate</div>
  </div>

  <div id="tab-plan" style="display:block">
    <div class="panel">
      <div class="panel-header">AWS Resource Decision</div>
      <pre id="out-plan">Run "Detect Resources" or "Full Pipeline" to start…</pre>
    </div>
  </div>

  <div id="tab-tf">
    <div class="panel">
      <div class="panel-header">Generated Files</div>
      <div class="file-list" id="file-list"></div>
    </div>
    <div class="panel" style="margin-top:10px">
      <div class="panel-header" id="tf-filename">Select a file above</div>
      <pre id="out-tf">Click a filename to view its contents.</pre>
    </div>
  </div>

  <div id="tab-cost">
    <div class="panel">
      <div class="panel-header">30-Day Cost Estimate (us-east-1 on-demand)</div>
      <pre id="out-cost" class="cost-box">Run "Cost Estimate" or "Full Pipeline" to see costs.</pre>
    </div>
  </div>

  <script>
    let tfFiles = {};

    function switchTab(name) {
      ['plan','tf','cost'].forEach(t => {
        document.getElementById('tab-'+t).style.display = t === name ? 'block' : 'none';
        document.querySelectorAll('.tab').forEach((el,i) => {
          el.classList.toggle('active', ['plan','tf','cost'][i] === name);
        });
      });
    }

    async function runStep(step) {
      const desc = document.getElementById('desc').value.trim();
      document.getElementById('spinner').style.display = 'block';
      try {
        const r    = await fetch('/api/' + step, {
          method: 'POST',
          headers: {'Content-Type': 'application/json'},
          body: JSON.stringify({ description: desc })
        });
        const data = await r.json();

        if (data.decision)     document.getElementById('out-plan').textContent = data.decision;
        if (data.cost_report)  document.getElementById('out-cost').textContent = data.cost_report;
        if (data.files) {
          tfFiles = data.files;
          renderFileList(data.files);
        }

        // Auto-switch tabs based on step
        if (step === 'plan') switchTab('plan');
        if (step === 'tf')   switchTab('tf');
        if (step === 'cost') switchTab('cost');
        if (step === 'full') switchTab('plan');

      } catch(e) {
        document.getElementById('out-plan').textContent = 'Error: ' + e;
      } finally {
        document.getElementById('spinner').style.display = 'none';
      }
    }

    function renderFileList(files) {
      const el = document.getElementById('file-list');
      el.innerHTML = '';
      Object.keys(files).forEach(fname => {
        const div = document.createElement('div');
        div.className = 'file-item';
        div.textContent = '📄 ' + fname;
        div.onclick = () => {
          document.getElementById('out-tf').textContent = files[fname];
          document.getElementById('tf-filename').textContent = fname;
        };
        el.appendChild(div);
      });
    }
  </script>
</body>
</html>
"""

_last_plan = {}

@app.route("/")
def index():
    return render_template_string(HTML)

@app.route("/api/plan", methods=["POST"])
def api_plan():
    global _last_plan
    desc = request.json.get("description", "")
    _last_plan = tools.parse_product_stack(desc)
    decision   = tools.decide_aws_resources(_last_plan)
    return jsonify(decision=decision)

@app.route("/api/tf", methods=["POST"])
def api_tf():
    global _last_plan
    desc = request.json.get("description", "")
    if not _last_plan:
        _last_plan = tools.parse_product_stack(desc)
    files = tools.generate_tf_files(_last_plan)
    return jsonify(files=files, decision=tools.decide_aws_resources(_last_plan))

@app.route("/api/cost", methods=["POST"])
def api_cost():
    global _last_plan
    desc = request.json.get("description", "")
    if not _last_plan:
        _last_plan = tools.parse_product_stack(desc)
    cost   = tools.estimate_cost_30d(_last_plan)
    report = tools.format_cost_report(cost)
    return jsonify(cost_report=report)

@app.route("/api/full", methods=["POST"])
def api_full():
    global _last_plan
    desc   = request.json.get("description", "")
    result = tools.full_pipeline(desc)
    _last_plan = result["plan"]
    return jsonify(
        decision    = result["decision"],
        files       = result["files"],
        cost_report = tools.format_cost_report(result["cost"]),
    )

if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

Run:

```bash
python app.py
```

Open: [http://localhost:5000](http://localhost:5000)

---

## Part 6 — Hands-On Exercises

### Exercise 1 — Minimal Serverless API

**Input description:**
```
Simple REST API with Lambda and DynamoDB. No servers. User auth with Cognito.
```

**Expected resources:** Lambda, API Gateway, DynamoDB, Cognito  
**Expected 30-day cost:** ~$5–12

---

### Exercise 2 — Traditional Web App

**Input description:**
```
Python Django application with PostgreSQL database, Redis cache, 
S3 for file uploads, and Celery background workers via SQS.
```

**Expected resources:** EC2, RDS (Postgres), ElastiCache (Redis), S3, SQS  
**Expected 30-day cost:** ~$80–130

---

### Exercise 3 — Containerised Microservices

**Input description:**
```
Containerised Node.js microservices with Docker, auto-scaling,
load balancer, PostgreSQL, and a Redis cache.
```

**Expected resources:** ECS Fargate, ALB, RDS, ElastiCache, VPC  
**Expected 30-day cost:** ~$120–200

---

### Exercise 4 — Static Site + API

**Input description:**
```
React single-page app served via CDN, with a serverless API backend
and user authentication.
```

**Expected resources:** S3, CloudFront, Lambda, API Gateway, Cognito  
**Expected 30-day cost:** ~$15–30

---

### Exercise 5 — Full Pipeline Challenge

Run the **Full Pipeline** on a description of your own current or side project.

1. Check the resource decisions — are they accurate?
2. Open the generated `.tf` files in `output/<app-name>/`
3. Note which resources drive the most cost
4. Try adjusting the instance types manually and re-run the cost estimate

---

## Part 7 — Troubleshooting

### Ollama not responding

```bash
# Check status
curl http://localhost:11434/api/tags

# Start if needed
ollama serve

# Confirm model is downloaded
ollama list
```

---

### JSON parse errors from Ollama

`llama3.2:1b` sometimes wraps JSON in markdown fences. The `_extract_json` helper handles this automatically. If the plan still looks wrong, the fallback keyword matcher kicks in.

---

### Flask not found

```bash
pip install flask
```

---

### Terraform files not appearing

```bash
# Confirm output directory exists
ls output/

# Test tools directly
python -c "import tools; print(tools.full_pipeline('Django app with Postgres'))"
```

---

### Validate the generated Terraform

```bash
cd output/<app-name>
terraform init
terraform validate
```

---

## Part 8 — Cost Reference Table

| Resource | Type | $/month (us-east-1 on-demand) |
|---|---|---|
| EC2 | t3.micro | $8.47 |
| EC2 | t3.small | $16.94 |
| EC2 | t3.medium | $33.87 |
| EC2 | m5.large | $69.12 |
| RDS | db.t3.micro (Postgres) | $15.33 |
| RDS | db.t3.medium | $61.32 |
| RDS | db.r5.large | $175.20 |
| ElastiCache | cache.t3.micro (Redis) | $13.68 |
| ECS Fargate | 0.25 vCPU / 0.5 GB | $14.40 |
| ALB | per LCU (base) | $16.20 |
| NAT Gateway | 1 AZ | $32.40 |
| CloudFront | 100 GB transfer | $10.00 |
| S3 | 50 GB + requests | ~$2–4 |
| Lambda | 1M requests | ~$0.20 |
| API Gateway | HTTP | ~$3.50 |
| DynamoDB | On-demand (light use) | ~$1.00 |
| SQS | 1M messages | ~$0.40 |
| Cognito | Up to 50K MAU | $0.00 |

> **Tip:** Reserved Instances (1-year) save ~30–40%. Savings Plans save up to 60% on Compute.

---

## Part 9 — Bonus Extensions

Add these to `tools.py` and wire into CLI / UI:

```python
def generate_tfvars(plan: dict) -> str:
    """Generate a terraform.tfvars file with sensible defaults."""
    app = plan.get("app_name", "my-app")
    env = plan.get("environment", "production")
    return f'''# terraform.tfvars — fill in real values before applying
app_name    = "{app}"
environment = "{env}"
aws_region  = "{plan.get("region", "us-east-1")}"
# db_password = "CHANGE_ME"
# ecr_image_uri = "123456789.dkr.ecr.us-east-1.amazonaws.com/{app}:latest"
'''


def generate_readme(plan: dict, cost: dict) -> str:
    """Generate a README.md for the Terraform module."""
    app   = plan.get("app_name", "my-app")
    lines = [
        f"# {app} — Terraform Infrastructure",
        "",
        plan.get("summary", ""),
        "",
        "## Quick Start",
        "```bash",
        "terraform init",
        "terraform plan -var-file=terraform.tfvars",
        "terraform apply -var-file=terraform.tfvars",
        "```",
        "",
        f"## Estimated Monthly Cost: ${cost['total_30d']}",
        "",
        "| Resource | Monthly Cost |",
        "|---|---|",
    ]
    for label, amount in cost["breakdown"].items():
        lines.append(f"| {label} | ${amount:.2f} |")
    return "\n".join(lines)
```

---

## Reference

| Resource | URL |
|---|---|
| Ollama API docs | https://github.com/ollama/ollama/blob/main/docs/api.md |
| Ollama model library | https://ollama.com/library |
| Terraform AWS provider | https://registry.terraform.io/providers/hashicorp/aws/latest/docs |
| AWS pricing calculator | https://calculator.aws/pricing/2/home |
| Terraform best practices | https://developer.hashicorp.com/terraform/language/style |

---

## Workshop Summary

By the end of this workshop you have:

- [x] Built a product-description parser that decides which AWS resources to provision
- [x] Generated production-quality `.tf` files for EC2, RDS, S3, Lambda, ECS, ALB, CloudFront, Cognito, SQS, ElastiCache, DynamoDB, and API Gateway
- [x] Computed a detailed 30-day AWS cost estimate from the resource plan
- [x] Used `llama3.2:1b` locally (free, private) for AI inference
- [x] Built both a terminal CLI and a browser UI
- [x] Saved all outputs to `output/<app-name>/` ready for `terraform init`

---

*Workshop — Terraform AI Provisioner + Ollama · April 2026*
