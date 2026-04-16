Great idea! The main challenge is that Ollama doesn't speak MCP protocol natively, so we'll replace the Claude ↔ MCP layer with **Ollama's native tool-calling API** and rewrite the Kubernetes tools in Python. The architecture shifts slightly:

```
User prompt → Ollama Agent (Python) → K8s tool functions → kubectl/k8s API
```

Here's the full self-hosted version:Here's the full Ollama-based version. The key shift: instead of Claude speaking MCP, we use **Ollama's native tool-calling API** (supported on `llama3.1`, `qwen2.5`, `mistral`) with a Python orchestrator.Now here's all the code:

---

### 1. Prerequisites

```bash
# Install Ollama (Linux/macOS)
curl -fsSL https://ollama.com/install.sh | sh

# Pull a model with tool-calling support
ollama pull llama3.1        # recommended
# or: ollama pull qwen2.5, mistral, mistral-nemo

# Python deps
pip install ollama kubernetes pyyaml
```

---

### 2. The Agent (`k8s_agent.py`)

```python
import json
import yaml
import ollama
from kubernetes import client, config

# ── Kubernetes client setup ─────────────────────────────────────────────────
try:
    config.load_incluster_config()   # running inside a Pod
except config.ConfigException:
    config.load_kube_config()        # local ~/.kube/config

core_v1   = client.CoreV1Api()
apps_v1   = client.AppsV1Api()

# ── Tool implementations ─────────────────────────────────────────────────────

def get_pods(namespace: str = "default", label_selector: str = None) -> str:
    kwargs = {"label_selector": label_selector} if label_selector else {}
    pods = core_v1.list_namespaced_pod(namespace, **kwargs).items
    if not pods:
        return f"No pods found in namespace '{namespace}'."
    lines = ["NAME  STATUS  READY  AGE"]
    for p in pods:
        phase = p.status.phase or "Unknown"
        statuses = p.status.container_statuses or []
        ready = sum(1 for c in statuses if c.ready)
        total = len(statuses)
        ts = p.metadata.creation_timestamp
        age = f"{int((__import__('datetime').datetime.now(ts.tzinfo) - ts).total_seconds() // 60)}m" if ts else "?"
        lines.append(f"{p.metadata.name}  {phase}  {ready}/{total}  {age}")
    return "\n".join(lines)


def get_logs(pod_name: str, namespace: str = "default",
             container: str = None, tail_lines: int = 100) -> str:
    kwargs = {"tail_lines": tail_lines}
    if container:
        kwargs["container"] = container
    return core_v1.read_namespaced_pod_log(pod_name, namespace, **kwargs)


def scale_deployment(name: str, replicas: int, namespace: str = "default") -> str:
    if not (0 <= replicas <= 50):
        return "Error: replicas must be between 0 and 50."
    apps_v1.patch_namespaced_deployment_scale(
        name, namespace,
        {"spec": {"replicas": replicas}}
    )
    return f'Deployment "{name}" in "{namespace}" scaled to {replicas} replicas.'


def describe_node(node_name: str) -> str:
    node = core_v1.read_node(node_name)
    cap   = node.status.capacity or {}
    alloc = node.status.allocatable or {}
    taints = ", ".join(
        f"{t.key}={t.value or ''}:{t.effect}"
        for t in (node.spec.taints or [])
    ) or "none"
    conditions = ", ".join(
        f"{c.type}={c.status}"
        for c in (node.status.conditions or [])
    )
    return "\n".join([
        f"Node: {node.metadata.name}",
        f"CPU capacity: {cap.get('cpu')}  allocatable: {alloc.get('cpu')}",
        f"Memory capacity: {cap.get('memory')}  allocatable: {alloc.get('memory')}",
        f"Max pods: {cap.get('pods')}",
        f"Taints: {taints}",
        f"Conditions: {conditions}",
    ])


def apply_manifest(manifest_yaml: str, namespace: str = "default") -> str:
    manifest = yaml.safe_load(manifest_yaml)
    kind = manifest.get("kind")
    ns   = manifest.get("metadata", {}).get("namespace", namespace)
    if kind == "Deployment":
        apps_v1.create_namespaced_deployment(ns, manifest)
        return f'Deployment "{manifest["metadata"]["name"]}" applied in "{ns}".'
    return f'Error: kind "{kind}" not yet supported.'


# ── Tool registry ─────────────────────────────────────────────────────────────

TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_pods",
            "description": "List all pods in a namespace with status and age",
            "parameters": {
                "type": "object",
                "properties": {
                    "namespace":      {"type": "string", "description": "Kubernetes namespace (default: default)"},
                    "label_selector": {"type": "string", "description": "Label selector e.g. app=nginx"},
                },
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "get_logs",
            "description": "Fetch recent logs from a pod container",
            "parameters": {
                "type": "object",
                "required": ["pod_name"],
                "properties": {
                    "pod_name":   {"type": "string"},
                    "namespace":  {"type": "string", "default": "default"},
                    "container":  {"type": "string"},
                    "tail_lines": {"type": "integer", "description": "Lines to return (default 100)"},
                },
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "scale_deployment",
            "description": "Scale a Deployment to a given replica count",
            "parameters": {
                "type": "object",
                "required": ["name", "replicas"],
                "properties": {
                    "name":      {"type": "string"},
                    "namespace": {"type": "string", "default": "default"},
                    "replicas":  {"type": "integer", "minimum": 0, "maximum": 50},
                },
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "describe_node",
            "description": "Show capacity, allocatable resources, taints, and conditions for a node",
            "parameters": {
                "type": "object",
                "required": ["node_name"],
                "properties": {
                    "node_name": {"type": "string"},
                },
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "apply_manifest",
            "description": "Apply a Kubernetes YAML manifest",
            "parameters": {
                "type": "object",
                "required": ["manifest_yaml"],
                "properties": {
                    "manifest_yaml": {"type": "string", "description": "Full YAML manifest"},
                    "namespace":     {"type": "string", "default": "default"},
                },
            },
        },
    },
]

TOOL_MAP = {
    "get_pods":          get_pods,
    "get_logs":          get_logs,
    "scale_deployment":  scale_deployment,
    "describe_node":     describe_node,
    "apply_manifest":    apply_manifest,
}

# ── Agent loop ────────────────────────────────────────────────────────────────

SYSTEM = (
    "You are a Kubernetes operations assistant. "
    "Use the provided tools to answer questions about the cluster. "
    "Always call a tool before answering factual questions about cluster state."
)

def run_agent(user_prompt: str, model: str = "llama3.1") -> str:
    messages = [
        {"role": "system",  "content": SYSTEM},
        {"role": "user",    "content": user_prompt},
    ]

    while True:
        response = ollama.chat(
            model=model,
            messages=messages,
            tools=TOOLS,
        )
        msg = response["message"]
        messages.append(msg)

        # No tool call → final answer
        if not msg.get("tool_calls"):
            return msg["content"]

        # Execute every tool call the model requested
        for tc in msg["tool_calls"]:
            fn_name = tc["function"]["name"]
            fn_args = tc["function"]["arguments"]
            if isinstance(fn_args, str):
                fn_args = json.loads(fn_args)

            print(f"  [tool] {fn_name}({fn_args})")
            try:
                result = TOOL_MAP[fn_name](**fn_args)
            except Exception as e:
                result = f"Error: {e}"

            messages.append({
                "role":    "tool",
                "content": result,
            })

# ── CLI ───────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    import sys

    model = "llama3.1"
    print(f"K8s Agent (Ollama/{model}) — type 'exit' to quit\n")

    while True:
        try:
            prompt = input("You: ").strip()
        except (EOFError, KeyboardInterrupt):
            break
        if prompt.lower() in ("exit", "quit"):
            break
        if not prompt:
            continue
        print(f"Agent: {run_agent(prompt, model=model)}\n")
```

---

### 3. RBAC (unchanged, still required)

```yaml
# k8s/rbac.yaml — same as the original lab
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-agent
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-agent-role
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "nodes", "namespaces"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "deployments/scale"]
    verbs: ["get", "list", "patch", "update"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k8s-agent-binding
subjects:
  - kind: ServiceAccount
    name: k8s-agent
    namespace: default
roleRef:
  kind: ClusterRole
  name: k8s-agent-role
  apiGroup: rbac.authorization.k8s.io
```

Apply it: `kubectl apply -f k8s/rbac.yaml`

---

### 4. Run it

```bash
# Local dev (uses ~/.kube/config)
python k8s_agent.py

# Example session:
You: List all crashing pods in the production namespace
  [tool] get_pods({'namespace': 'production'})
Agent: Here are the pods in production...

You: Show me the last 50 lines from api-server pod logs
  [tool] get_logs({'pod_name': 'api-server-xyz', 'tail_lines': 50})

You: Scale frontend to 3 replicas
  [tool] scale_deployment({'name': 'frontend', 'replicas': 3})
```

---

### Key differences vs the original MCP lab

| | Original (MCP + Claude) | This (Ollama) |
|---|---|---|
| LLM | Claude API (cloud) | Ollama (fully local) |
| Protocol | MCP stdio/SSE | Ollama native tool-call API |
| Language | TypeScript | Python |
| Multi-user | SSE transport | Add FastAPI wrapper |
| Cost | API tokens | Zero (local GPU/CPU) |
| Best model | Claude Sonnet | `llama3.1`, `qwen2.5` |

For a **multi-user / server deployment**, wrap `run_agent()` in a FastAPI endpoint and run Ollama as a systemd service. The tool functions stay identical.
