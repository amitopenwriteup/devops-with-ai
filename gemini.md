Yes! Gemini has a generous free tier (via Google AI Studio) and supports native function/tool calling. Here's the rewritten agent:

---

### 1. Get your free Gemini API key

Go to [aistudio.google.com](https://aistudio.google.com/app/apikey) → **Get API key** → copy it. Free tier gives you **1,500 requests/day** on `gemini-1.5-flash`.

---

### 2. Install deps

```bash
pip install google-generativeai kubernetes pyyaml
```

---

### 3. `k8s_agent.py` (Gemini version)

```python
import json
import os
import yaml
import google.generativeai as genai
from google.generativeai.types import FunctionDeclaration, Tool
from kubernetes import client, config

# ── Gemini setup ──────────────────────────────────────────────────────────────
genai.configure(api_key=os.environ["GEMINI_API_KEY"])

# ── Kubernetes client setup ───────────────────────────────────────────────────
try:
    config.load_incluster_config()
except config.ConfigException:
    config.load_kube_config()

core_v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()

# ── Tool implementations ──────────────────────────────────────────────────────

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
        name, namespace, {"spec": {"replicas": replicas}}
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
        f"{c.type}={c.status}" for c in (node.status.conditions or [])
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


# ── Tool map ──────────────────────────────────────────────────────────────────

TOOL_MAP = {
    "get_pods":         get_pods,
    "get_logs":         get_logs,
    "scale_deployment": scale_deployment,
    "describe_node":    describe_node,
    "apply_manifest":   apply_manifest,
}

# ── Gemini tool declarations ──────────────────────────────────────────────────

gemini_tools = Tool(function_declarations=[
    FunctionDeclaration(
        name="get_pods",
        description="List all pods in a namespace with their status and age.",
        parameters={
            "type": "object",
            "properties": {
                "namespace":      {"type": "string", "description": "Kubernetes namespace (default: default)"},
                "label_selector": {"type": "string", "description": "Label selector e.g. app=nginx"},
            },
        },
    ),
    FunctionDeclaration(
        name="get_logs",
        description="Fetch recent logs from a pod container.",
        parameters={
            "type": "object",
            "required": ["pod_name"],
            "properties": {
                "pod_name":   {"type": "string"},
                "namespace":  {"type": "string"},
                "container":  {"type": "string"},
                "tail_lines": {"type": "integer", "description": "Number of lines to return (default 100)"},
            },
        },
    ),
    FunctionDeclaration(
        name="scale_deployment",
        description="Scale a Deployment to a given replica count.",
        parameters={
            "type": "object",
            "required": ["name", "replicas"],
            "properties": {
                "name":      {"type": "string"},
                "namespace": {"type": "string"},
                "replicas":  {"type": "integer"},
            },
        },
    ),
    FunctionDeclaration(
        name="describe_node",
        description="Show capacity, allocatable resources, taints, and conditions for a node.",
        parameters={
            "type": "object",
            "required": ["node_name"],
            "properties": {
                "node_name": {"type": "string"},
            },
        },
    ),
    FunctionDeclaration(
        name="apply_manifest",
        description="Apply a Kubernetes YAML manifest to the cluster.",
        parameters={
            "type": "object",
            "required": ["manifest_yaml"],
            "properties": {
                "manifest_yaml": {"type": "string", "description": "Full YAML manifest as a string"},
                "namespace":     {"type": "string"},
            },
        },
    ),
])

# ── Agent loop ────────────────────────────────────────────────────────────────

SYSTEM = (
    "You are a Kubernetes operations assistant. "
    "Use the provided tools to answer questions about the cluster. "
    "Always call a tool before answering factual questions about cluster state."
)

def run_agent(user_prompt: str) -> str:
    model = genai.GenerativeModel(
        model_name="gemini-1.5-flash",   # free tier; swap to gemini-1.5-pro if needed
        system_instruction=SYSTEM,
        tools=[gemini_tools],
    )
    chat = model.start_chat()

    # Send the first user message
    response = chat.send_message(user_prompt)

    # Agentic loop — keep going while Gemini wants to call tools
    while True:
        part = response.candidates[0].content.parts[0]

        # If it's a function call, execute it and feed result back
        if hasattr(part, "function_call") and part.function_call.name:
            fn_name = part.function_call.name
            fn_args = dict(part.function_call.args)

            print(f"  [tool] {fn_name}({fn_args})")
            try:
                result = TOOL_MAP[fn_name](**fn_args)
            except Exception as e:
                result = f"Error: {e}"

            # Return tool result to the model
            response = chat.send_message(
                genai.protos.Part(
                    function_response=genai.protos.FunctionResponse(
                        name=fn_name,
                        response={"result": result},
                    )
                )
            )
        else:
            # Plain text → final answer
            return part.text

# ── CLI ───────────────────────────────────────────────────────────────────────

if __name__ == "__main__":
    print("K8s Agent (Gemini 1.5 Flash / free tier) — type 'exit' to quit\n")
    while True:
        try:
            prompt = input("You: ").strip()
        except (EOFError, KeyboardInterrupt):
            break
        if prompt.lower() in ("exit", "quit"):
            break
        if not prompt:
            continue
        print(f"Agent: {run_agent(prompt)}\n")
```

---

### 4. Run it

```bash
export GEMINI_API_KEY="your-key-here"
python k8s_agent.py
```

---

### What changed vs the Ollama version

| | Ollama | Gemini (free) |
|---|---|---|
| **SDK** | `ollama` | `google-generativeai` |
| **Tool format** | OpenAI-style JSON dict | `FunctionDeclaration` objects |
| **Tool result** | `{"role": "tool", ...}` message | `FunctionResponse` proto part |
| **Cost** | Free (local GPU) | Free (1,500 req/day) |
| **Model** | `llama3.1` | `gemini-1.5-flash` |
| **Internet required** | No | Yes |

The Kubernetes tool functions themselves (`get_pods`, `get_logs`, etc.) and the RBAC YAML are **completely unchanged** — only the LLM layer swaps out.
