# RAG-Based AWS Architecture Workshop

> **Local Models:** `llama3.2:1b` (1.3 GB) · `nomic-embed-text` (274 MB)
> **Pipeline:** Embed → Retrieve → Generate → Terraform

---

## Table of Contents

1. [Workshop Overview](#1-workshop-overview)
2. [RAG Pipeline Architecture](#2-rag-pipeline-architecture)
3. [Local Model Setup](#3-local-model-setup)
4. [Product → AWS Service Mapping](#4-product--aws-service-mapping)
5. [Terraform File Blueprints](#5-terraform-file-blueprints)
6. [Hands-On Exercises](#6-hands-on-exercises)
7. [Reference Cheat Sheet](#7-reference-cheat-sheet)

---

## 1. Workshop Overview

This workshop teaches you to build a **Retrieval-Augmented Generation (RAG)** pipeline that:

- Accepts a product description as input
- Embeds it using `nomic-embed-text` for semantic search
- Retrieves the closest AWS architecture patterns from a knowledge base
- Uses `llama3.2:1b` to generate service recommendations
- Outputs a ready-to-use Terraform file structure

**What you will build:**

```
[Product Input] → [nomic-embed-text] → [Vector Store]
                                              ↓
                                    [Top-K Retrieval]
                                              ↓
                               [llama3.2:1b + Context]
                                              ↓
                          [AWS Services + Terraform Files]
```

---

## 2. RAG Pipeline Architecture

### 2.1 Embedding Phase

`nomic-embed-text` converts your product description into a 768-dimensional vector.

```python
import ollama

def embed_query(product_description: str) -> list[float]:
    response = ollama.embeddings(
        model="nomic-embed-text",
        prompt=product_description
    )
    return response["embedding"]  # 768-dim vector
```

### 2.2 Retrieval Phase

Compute cosine similarity against your knowledge base of architecture patterns.

```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

def retrieve_top_k(query_vec, kb_embeddings, k=3):
    scores = [
        (i, cosine_similarity(query_vec, emb))
        for i, emb in enumerate(kb_embeddings)
    ]
    return sorted(scores, key=lambda x: x[1], reverse=True)[:k]
```

### 2.3 Generation Phase

Pass retrieved context to `llama3.2:1b` and prompt for structured output.

```python
def generate_aws_plan(product: str, retrieved_context: str) -> str:
    prompt = f"""
You are an AWS solutions architect.

Product: {product}

Relevant architecture patterns:
{retrieved_context}

Return:
1. Recommended AWS services (name, category, purpose)
2. Terraform file names and what each manages

Be specific and concise.
"""
    response = ollama.chat(
        model="llama3.2:1b",
        messages=[{"role": "user", "content": prompt}]
    )
    return response["message"]["content"]
```

---

## 3. Local Model Setup

### 3.1 Verify Models Are Running

```bash
ollama list
```

Expected output:

```
NAME                       ID              SIZE      MODIFIED
llama3.2:1b                baf6a787fdff    1.3 GB    16 minutes ago
nomic-embed-text:latest    0a109f422b47    274 MB    28 hours ago
```

### 3.2 Test Embedding Model

```bash
curl http://localhost:11434/api/embeddings \
  -d '{"model": "nomic-embed-text", "prompt": "e-commerce platform with cart and payments"}'
```

### 3.3 Test LLM

```bash
curl http://localhost:11434/api/chat \
  -d '{
    "model": "llama3.2:1b",
    "messages": [{"role": "user", "content": "List 3 AWS services for a SaaS CRM"}],
    "stream": false
  }'
```

### 3.4 Install Python Dependencies

```bash
pip install ollama numpy
```

### 3.5 Full Pipeline Script

```python
# rag_aws_workshop.py

import ollama
import numpy as np

KNOWLEDGE_BASE = [
    {
        "pattern": "e-commerce marketplace",
        "services": ["ECS", "Aurora RDS", "ElastiCache", "S3", "CloudFront", "SQS"],
        "terraform": ["vpc.tf", "ecs.tf", "rds.tf", "elasticache.tf", "s3.tf", "cloudfront.tf", "sqs.tf"]
    },
    {
        "pattern": "real-time IoT data pipeline",
        "services": ["IoT Core", "Kinesis", "Lambda", "DynamoDB", "Timestream", "Grafana"],
        "terraform": ["vpc.tf", "iot_core.tf", "kinesis.tf", "lambda.tf", "dynamodb.tf", "timestream.tf"]
    },
    {
        "pattern": "ML model serving API",
        "services": ["SageMaker", "ECR", "API Gateway", "S3", "Lambda", "CloudWatch"],
        "terraform": ["vpc.tf", "sagemaker.tf", "ecr.tf", "api_gateway.tf", "lambda.tf", "s3.tf"]
    },
    {
        "pattern": "multi-tenant SaaS CRM",
        "services": ["EKS", "Aurora", "Cognito", "OpenSearch", "SES", "WAF"],
        "terraform": ["vpc.tf", "eks.tf", "rds.tf", "cognito.tf", "opensearch.tf", "ses.tf", "waf.tf"]
    },
    {
        "pattern": "video streaming platform",
        "services": ["S3", "MediaConvert", "CloudFront", "DynamoDB", "Lambda", "IVS"],
        "terraform": ["vpc.tf", "s3.tf", "mediaconvert.tf", "cloudfront.tf", "dynamodb.tf", "ivs.tf"]
    },
    {
        "pattern": "serverless microservices backend",
        "services": ["Lambda", "API Gateway", "DynamoDB", "EventBridge", "SQS", "X-Ray"],
        "terraform": ["vpc.tf", "api_gateway.tf", "lambda.tf", "dynamodb.tf", "eventbridge.tf", "sqs.tf"]
    },
]

def embed(text: str) -> list:
    return ollama.embeddings(model="nomic-embed-text", prompt=text)["embedding"]

def cosine(a, b) -> float:
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

def run(product: str):
    print(f"\nProduct: {product}\n")

    # Embed KB
    print("Embedding knowledge base...")
    kb_vecs = [embed(p["pattern"]) for p in KNOWLEDGE_BASE]

    # Embed query
    print("Embedding query...")
    q_vec = embed(product)

    # Retrieve top-2
    scores = [(i, cosine(q_vec, v)) for i, v in enumerate(kb_vecs)]
    top = sorted(scores, key=lambda x: x[1], reverse=True)[:2]

    context = ""
    for idx, score in top:
        p = KNOWLEDGE_BASE[idx]
        context += f"Pattern: {p['pattern']} (score: {score:.2f})\n"
        context += f"Services: {', '.join(p['services'])}\n"
        context += f"Terraform: {', '.join(p['terraform'])}\n\n"

    print("\nRetrieved context:\n", context)

    # Generate
    print("Generating recommendations...\n")
    prompt = f"""Product: {product}

Relevant AWS architecture patterns:
{context}

Provide:
1. Top AWS services to use (name + one-line purpose)
2. Terraform files needed (filename + what it manages)
"""
    resp = ollama.chat(
        model="llama3.2:1b",
        messages=[{"role": "user", "content": prompt}],
        stream=False
    )
    print(resp["message"]["content"])

if __name__ == "__main__":
    product = input("Describe your product: ")
    run(product)
```

Run it:

```bash
python rag_aws_workshop.py
```

---

## 4. Product → AWS Service Mapping

### 4.1 E-Commerce Marketplace

| AWS Service | Category | Purpose |
|---|---|---|
| Amazon ECS | Compute | Container orchestration for storefront & order services |
| Amazon Aurora | Database | Multi-AZ relational DB for catalog & orders |
| Amazon ElastiCache | Caching | Redis for session store & cart caching |
| Amazon S3 | Storage | Product images, static assets |
| Amazon CloudFront | CDN | Global asset delivery, WAF integration |
| AWS SQS | Messaging | Async order processing queue |

### 4.2 Real-Time IoT Pipeline

| AWS Service | Category | Purpose |
|---|---|---|
| AWS IoT Core | IoT | Managed MQTT broker for device connections |
| Amazon Kinesis | Streaming | High-throughput sensor data ingest |
| AWS Lambda | Compute | Stream enrichment and routing |
| Amazon DynamoDB | Database | Device state, sub-millisecond reads |
| Amazon Timestream | Time-series | Long-term IoT metrics & analytics |
| Amazon Managed Grafana | Observability | Real-time telemetry dashboards |

### 4.3 ML Model Serving API

| AWS Service | Category | Purpose |
|---|---|---|
| Amazon SageMaker | ML Platform | Model training, registry, endpoint hosting |
| Amazon ECR | Registry | Custom inference container images |
| AWS API Gateway | API Layer | REST/HTTP API fronting inference |
| Amazon S3 | Storage | Model artefacts, training datasets |
| AWS Lambda | Compute | Pre/post-processing, auth, routing |
| Amazon CloudWatch | Observability | Latency alarms, data drift detection |

### 4.4 Multi-Tenant SaaS CRM

| AWS Service | Category | Purpose |
|---|---|---|
| Amazon EKS | Compute | Kubernetes, namespace-per-tenant isolation |
| Amazon Aurora | Database | Tenant-sharded relational storage |
| Amazon Cognito | Auth | Multi-tenant user pools, SSO, OIDC |
| Amazon OpenSearch | Search | Full-text contact & deal search |
| Amazon SES | Email | Transactional email sequences |
| AWS WAF | Security | Rate limiting, cross-tenant protection |

### 4.5 Video Streaming Platform

| AWS Service | Category | Purpose |
|---|---|---|
| Amazon S3 | Storage | Source files, HLS segments, thumbnails |
| AWS Elemental MediaConvert | Transcoding | ABR transcoding to HLS/DASH |
| Amazon CloudFront | CDN | Global delivery with signed URL auth |
| Amazon DynamoDB | Database | Video metadata, watch progress |
| AWS Lambda | Compute | Upload triggers, transcoding jobs |
| Amazon IVS | Live Streaming | Ultra-low latency live channels |

### 4.6 Serverless Microservices Backend

| AWS Service | Category | Purpose |
|---|---|---|
| AWS Lambda | Compute | Event-driven functions per service |
| Amazon API Gateway | API Layer | HTTP API, JWT auth, rate limiting |
| Amazon DynamoDB | Database | On-demand key-value, per-service tables |
| Amazon EventBridge | Event Bus | Service-to-service events, schedules |
| AWS SQS | Queuing | Async work queues between services |
| AWS X-Ray | Observability | Distributed tracing end-to-end |

---

## 5. Terraform File Blueprints

### 5.1 Standard Files (All Projects)

| File | Purpose |
|---|---|
| `main.tf` | Root module — provider config, S3 backend, required_providers |
| `variables.tf` | Input variables — env, region, instance sizes |
| `outputs.tf` | Exported values — endpoints, ARNs, DNS names |
| `versions.tf` | Provider version constraints |
| `iam.tf` | All IAM roles, policies, instance profiles |

### 5.2 E-Commerce (`aws-ecommerce/`)

```
aws-ecommerce/
├── main.tf           # Root module, S3 backend, provider
├── vpc.tf            # VPC, subnets, NAT gateway, route tables
├── ecs.tf            # ECS cluster, task definitions, ALB, services
├── rds.tf            # Aurora PostgreSQL, multi-AZ, parameter groups
├── elasticache.tf    # Redis cluster, subnet group, security group
├── s3.tf             # Assets bucket, lifecycle rules, CORS
├── cloudfront.tf     # Distribution, OAC, cache behaviours, WAF ACL
├── sqs.tf            # Order queue, DLQ, redrive policy
├── iam.tf            # ECS task roles, S3 policies, SQS permissions
├── variables.tf      # env, container_image, db_instance_class
└── outputs.tf        # ALB DNS, DB endpoint, CloudFront domain
```

### 5.3 IoT Pipeline (`aws-iot/`)

```
aws-iot/
├── main.tf           # Root module, backend
├── vpc.tf            # VPC, private subnets, VPC endpoints
├── iot_core.tf       # IoT policies, thing types, topic rules
├── kinesis.tf        # Data stream, shards, enhanced fanout
├── lambda.tf         # Processor functions, ESM, concurrency
├── dynamodb.tf       # Device state table, GSI, TTL
├── timestream.tf     # Database, tables, retention policies
├── grafana.tf        # Managed workspace, data sources, SAML
├── iam.tf            # Lambda roles, IoT rule action roles
├── variables.tf      # shard_count, retention_days, env
└── outputs.tf        # IoT endpoint, stream ARN, dashboard URL
```

### 5.4 ML Serving (`aws-ml/`)

```
aws-ml/
├── main.tf           # Root module, S3 backend
├── vpc.tf            # VPC, private subnets, VPC endpoints
├── sagemaker.tf      # Endpoints, endpoint configs, model registry
├── ecr.tf            # Repositories, lifecycle policies
├── api_gateway.tf    # HTTP API, JWT authoriser, stages, routes
├── lambda.tf         # Auth, pre/post-processing, routing
├── s3.tf             # Artefacts bucket, dataset bucket, versioning
├── cloudwatch.tf     # Dashboards, alarms, log groups
├── iam.tf            # SageMaker execution role, Lambda permissions
├── variables.tf      # instance_type, model_name, env
└── outputs.tf        # API URL, SageMaker endpoint name, ECR URI
```

### 5.5 SaaS CRM (`aws-crm/`)

```
aws-crm/
├── main.tf           # Root module, remote state
├── vpc.tf            # VPC, subnets, security groups, NACLs
├── eks.tf            # Cluster, node groups, managed add-ons, IRSA
├── rds.tf            # Aurora cluster, multi-AZ, read replicas
├── cognito.tf        # User pools, app clients, identity pools
├── opensearch.tf     # Domain, access policies, SAML auth
├── ses.tf            # Identity, sending limits, DKIM DNS records
├── waf.tf            # Web ACL, managed rule groups, rate limits
├── iam.tf            # IRSA roles, service accounts, policies
├── variables.tf      # cluster_version, node_count, env
└── outputs.tf        # Cluster endpoint, Cognito pool IDs, OS domain
```

### 5.6 Video Streaming (`aws-video/`)

```
aws-video/
├── main.tf           # Root module, S3 backend
├── s3.tf             # Upload bucket, HLS bucket, CORS, lifecycle
├── mediaconvert.tf   # Job templates, queue, IAM role
├── cloudfront.tf     # Distribution, signed URL key group, cache
├── dynamodb.tf       # Videos table, watch-progress table, streams
├── lambda.tf         # Upload trigger, transcoding orchestrator
├── ivs.tf            # Live channels, stream keys, recording config
├── iam.tf            # MediaConvert role, Lambda execution roles
├── variables.tf      # region, env, cloudfront_price_class
└── outputs.tf        # CloudFront domain, IVS endpoints, S3 URLs
```

### 5.7 Serverless Backend (`aws-serverless/`)

```
aws-serverless/
├── main.tf           # Root module, S3 backend
├── api_gateway.tf    # HTTP API, JWT authoriser, stages, CORS
├── lambda.tf         # Function per microservice, layers, concurrency
├── dynamodb.tf       # Per-service tables, GSIs, streams
├── eventbridge.tf    # Event bus, rules, targets
├── sqs.tf            # Service queues, DLQs, redrive policies
├── xray.tf           # Sampling rules, groups, encryption config
├── iam.tf            # Lambda execution roles, resource policies
├── variables.tf      # env, memory_size, timeout, log_retention
└── outputs.tf        # API base URL, Lambda ARNs, table names
```

---

## 6. Hands-On Exercises

### Exercise 1 — Run the Embedding Pipeline

1. Start Ollama: `ollama serve`
2. Embed your product: `curl http://localhost:11434/api/embeddings -d '{"model":"nomic-embed-text","prompt":"YOUR PRODUCT HERE"}'`
3. Note the 768 dimensions in the response
4. Try two similar products — compare how close the vectors are

### Exercise 2 — Query the LLM

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2:1b",
  "stream": false,
  "messages": [{
    "role": "user",
    "content": "What AWS services should I use for a B2B SaaS invoice management tool? List service name and purpose only."
  }]
}'
```

### Exercise 3 — Build the Full Pipeline

1. Copy the full pipeline script from Section 3.5
2. Run: `python rag_aws_workshop.py`
3. Try these inputs:
   - `"B2B invoice management SaaS"`
   - `"Real-time sports analytics dashboard"`
   - `"Food delivery marketplace with driver tracking"`
4. Compare the retrieved patterns and generated Terraform structures

### Exercise 4 — Extend the Knowledge Base

Add a new pattern to `KNOWLEDGE_BASE` in the script:

```python
{
    "pattern": "healthcare patient portal",
    "services": ["ECS", "RDS", "Cognito", "S3", "KMS", "CloudTrail"],
    "terraform": ["vpc.tf", "ecs.tf", "rds.tf", "cognito.tf", "s3.tf", "kms.tf", "cloudtrail.tf"]
}
```

Re-embed and test: `"HIPAA-compliant patient appointment system"` — does it retrieve your new pattern?

### Exercise 5 — Generate a Real Terraform File

Use the LLM to scaffold an actual `variables.tf`:

```python
prompt = """
Generate a Terraform variables.tf file for an e-commerce ECS deployment.
Include: environment, aws_region, container_image, db_instance_class, desired_count.
Use proper variable blocks with type, description, and default.
"""
```

---

## 7. Reference Cheat Sheet

### Ollama Commands

```bash
ollama list                          # List installed models
ollama pull llama3.2:1b              # Pull model
ollama pull nomic-embed-text         # Pull embed model
ollama serve                         # Start server (default: localhost:11434)
ollama run llama3.2:1b               # Interactive chat
```

### Ollama Python API

```python
import ollama

# Embeddings
ollama.embeddings(model="nomic-embed-text", prompt="text")

# Chat
ollama.chat(model="llama3.2:1b", messages=[{"role":"user","content":"..."}])

# Stream
for chunk in ollama.chat(model="llama3.2:1b", messages=[...], stream=True):
    print(chunk["message"]["content"], end="")
```

### Terraform Conventions

```hcl
# Naming: <env>-<product>-<resource>
resource "aws_ecs_cluster" "main" {
  name = "${var.env}-${var.product}-cluster"

  tags = {
    Environment = var.env
    Product     = var.product
    Team        = var.team
    ManagedBy   = "terraform"
  }
}
```

### RAG Quality Tips

| Problem | Fix |
|---|---|
| Wrong pattern retrieved | Add more specific KB entries |
| Hallucinated services | Add services explicitly to context |
| Generic Terraform output | Include concrete variable names in prompt |
| Slow embedding | Batch embed KB upfront, cache vectors |
| Model too slow | Use `stream: true` for perceived speed |

---

*Generated by RAG Workshop · Models: llama3.2:1b + nomic-embed-text · Stack: Ollama + Python*
