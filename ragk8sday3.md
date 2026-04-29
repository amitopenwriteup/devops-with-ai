# KinD Kubernetes Agent 🤖

> An interactive CLI agent powered by **Ollama (llama3.1)** that reads the
> official KinD docs, guides you through cluster setup, handles ingress
> configuration, and generates a downloadable workshop `.md` file.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    kind_k8s_agent.py                    │
│                                                         │
│  ┌─────────────┐    ┌──────────────┐    ┌───────────┐  │
│  │  Web Scraper│───▶│  Knowledge   │───▶│  System   │  │
│  │  (bs4)      │    │  Base        │    │  Prompt   │  │
│  └─────────────┘    └──────────────┘    └─────┬─────┘  │
│  kind.sigs.k8s.io                             │         │
│                                        ┌──────▼──────┐  │
│  ┌─────────────┐                       │  Ollama API │  │
│  │   User      │◀─────────────────────▶│  llama3.1   │  │
│  │  (CLI)      │                       └─────────────┘  │
│  └──────┬──────┘                                        │
│         │                                               │
│  ┌──────▼──────────────────────────────────────────┐   │
│  │              Output Files                        │   │
│  │  kind-config.yaml   •   workshop.md              │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## Prerequisites

| Tool | Purpose | Install |
|------|---------|---------|
| Python 3.9+ | Run the agent | pre-installed |
| Ollama | Local LLM server | https://ollama.com |
| llama3.1 model | The LLM brain | `ollama pull llama3.1` |
| Docker | Run KinD nodes | https://docs.docker.com/get-docker/ |
| KinD | Kubernetes in Docker | `brew install kind` or see below |
| kubectl | Kubernetes CLI | `brew install kubectl` |

---

## Quick Start

### 1. Install Python dependencies

```bash
pip install requests beautifulsoup4 ollama
```

### 2. Start Ollama and pull the model

```bash
# Start Ollama server (runs on localhost:11434)
ollama serve

# In another terminal, pull the model
ollama pull llama3.1
```

### 3. Run the agent

```bash
python kind_k8s_agent.py
```

---

## What the agent does

```
Phase 1 ── Scrapes https://kind.sigs.k8s.io/docs/user/quick-start/
           (falls back to built-in knowledge if offline)

Phase 2 ── Asks you:
           • Cluster name
           • Kubernetes version
           • Node topology (single / HA / custom)
           • Port mappings
           • Whether to set up NGINX Ingress
           • Optional bind mounts

Phase 3 ── LLM generates kind-config.yaml

Phase 4 ── LLM generates step-by-step setup commands

Phase 5 ── (If ingress) LLM generates full NGINX Ingress guide
           including a demo Deployment + Service + Ingress YAML

Phase 6 ── Interactive Q&A — ask anything about KinD

Phase 7 ── Generates workshop.md combining everything above
```

---

## Output Files

After running, you get two files:

```
<cluster-name>-kind-config.yaml    ← ready to use with `kind create cluster`
<cluster-name>-workshop.md         ← full workshop document (share with your team)
```

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_HOST` | `http://localhost:11434` | Ollama server URL |
| `OLLAMA_MODEL` | `llama3.1` | Model name |

```bash
# Use a remote Ollama server or a different model
OLLAMA_HOST=http://myserver:11434 OLLAMA_MODEL=llama3.2 python kind_k8s_agent.py
```

---

## Example Session

```
╔══════════════════════════════════════════════════════════╗
║         KinD Kubernetes Agent  •  Ollama LLaMA 3.1       ║
╚══════════════════════════════════════════════════════════╝

[0] Checking Ollama connection …
✓  Ollama running — model 'llama3.1' ready.
[0] Loading KinD documentation …
ℹ  Fetching KinD docs from https://kind.sigs.k8s.io/docs/user/quick-start/ …
✓  Loaded 11,842 chars of KinD documentation.

════ KinD Cluster Configuration ════

? Cluster name [kind-workshop]: my-cluster
? Kubernetes version (leave blank for latest) [latest]:

[1] Node topology
  1) Single node
  2) HA control-plane (3 cp + 1 worker)
  3) Custom
? Choose [1/2/3] [1]: 1

[2] Port mappings
? Add host → container port mapping? (y/N) n

[3] Ingress controller
? Set up NGINX Ingress controller? (Y/n): y
✓  NGINX Ingress will be configured.

...

✓  Workshop document saved → my-cluster-workshop.md
```

---

## Ingress vs No Ingress

| Feature | Without Ingress | With Ingress |
|---------|-----------------|--------------|
| Access services | NodePort / port-forward | HTTP/HTTPS via domain |
| Setup complexity | Simple | Slightly more steps |
| Production-like routing | ❌ | ✅ |
| TLS termination | ❌ | ✅ (with cert-manager) |
| Path-based routing | ❌ | ✅ |

---

## Extending the Agent

The agent is a single Python file with clear sections:

- **`fetch_kind_docs()`** — swap in a different URL or add more doc pages
- **`KindAgent.gather_config()`** — add more configuration questions
- **`KindAgent.system_prompt`** — tune the LLM persona / knowledge
- **`KindAgent.generate_workshop_doc()`** — customise the output format
- **`OLLAMA_MODEL`** env var — try `llama3.2`, `mistral`, `codellama`, etc.

---

## Troubleshooting

**Ollama not found**
```bash
# Check if Ollama is running
curl http://localhost:11434/api/tags

# Start it
ollama serve
```

**Model not found**
```bash
ollama pull llama3.1
ollama list   # should show llama3.1
```

**Slow responses**  
llama3.1 is ~4GB. First run downloads the model. On CPU-only machines,
responses take 10–30 seconds. Use `llama3.2:1b` for faster (smaller) responses:
```bash
OLLAMA_MODEL=llama3.2:1b python kind_k8s_agent.py
```

---

*KinD Agent — MIT License*
