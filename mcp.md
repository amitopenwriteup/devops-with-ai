# Building a Production MCP Server for Kubernetes

## Architecture Overview

The MCP server sits between Claude AI and your Kubernetes cluster. Claude sends tool calls over stdio or SSE, the MCP server translates them into Kubernetes API requests, and results flow back as structured text.

```
Claude AI  ──stdio/SSE──▶  MCP Server (Node.js)  ──REST──▶  Kubernetes API Server
                               │                                      │
                               │  Tools:                  Namespaces: production
                               │  • get_pods                          staging
                               │  • get_logs                          kube-system
                               │  • scale_deployment                  monitoring
                               │  • describe_node
                               └  • apply_manifest        Auth: RBAC / ServiceAccount
```

---

## 1. Project Setup

```bash
mkdir k8s-mcp-server && cd k8s-mcp-server
npm init -y
npm install @modelcontextprotocol/sdk @kubernetes/client-node zod
```

---

## 2. The MCP Server (`src/index.ts`)

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import * as k8s from "@kubernetes/client-node";
import { z } from "zod";

// ─── Kubernetes client setup ─────────────────────────────────────────────────
const kc = new k8s.KubeConfig();

// In-cluster (Pod with ServiceAccount) or local kubeconfig
if (process.env.KUBERNETES_SERVICE_HOST) {
  kc.loadFromCluster();
} else {
  kc.loadFromDefault(); // reads ~/.kube/config
}

const k8sApi  = kc.makeApiClient(k8s.CoreV1Api);
const appsApi = kc.makeApiClient(k8s.AppsV1Api);

// ─── MCP Server ──────────────────────────────────────────────────────────────
const server = new Server(
  { name: "k8s-mcp-server", version: "1.0.0" },
  { capabilities: { tools: {} } }
);

// ─── Tool definitions ────────────────────────────────────────────────────────
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "get_pods",
      description: "List all pods in a namespace with their status and age",
      inputSchema: {
        type: "object",
        properties: {
          namespace: { type: "string", description: "Kubernetes namespace (default: default)" },
          label_selector: { type: "string", description: "Optional label selector e.g. app=nginx" },
        },
      },
    },
    {
      name: "get_logs",
      description: "Fetch recent logs from a pod container",
      inputSchema: {
        type: "object",
        required: ["pod_name"],
        properties: {
          pod_name:   { type: "string" },
          namespace:  { type: "string", default: "default" },
          container:  { type: "string", description: "Container name (omit for single-container pods)" },
          tail_lines: { type: "number", description: "Number of lines to return (default 100)" },
        },
      },
    },
    {
      name: "scale_deployment",
      description: "Scale a Deployment to a given replica count",
      inputSchema: {
        type: "object",
        required: ["name", "replicas"],
        properties: {
          name:      { type: "string" },
          namespace: { type: "string", default: "default" },
          replicas:  { type: "number", minimum: 0, maximum: 50 },
        },
      },
    },
    {
      name: "describe_node",
      description: "Return capacity, allocatable resources, taints, and conditions for a node",
      inputSchema: {
        type: "object",
        required: ["node_name"],
        properties: {
          node_name: { type: "string" },
        },
      },
    },
    {
      name: "apply_manifest",
      description: "Apply a raw Kubernetes YAML/JSON manifest (server-side apply)",
      inputSchema: {
        type: "object",
        required: ["manifest"],
        properties: {
          manifest:  { type: "string", description: "Full YAML or JSON manifest" },
          namespace: { type: "string", default: "default" },
        },
      },
    },
  ],
}));

// ─── Tool handlers ───────────────────────────────────────────────────────────
server.setRequestHandler(CallToolRequestSchema, async (req) => {
  const { name, arguments: args } = req.params;

  try {
    switch (name) {
      // ── get_pods ────────────────────────────────────────────────────────────
      case "get_pods": {
        const ns  = (args?.namespace as string) || "default";
        const sel = (args?.label_selector as string) || undefined;
        const res = await k8sApi.listNamespacedPod(
          ns, undefined, undefined, undefined, undefined, sel
        );
        const rows = res.body.items.map((p) => {
          const phase     = p.status?.phase ?? "Unknown";
          const ready     = p.status?.containerStatuses?.filter(c => c.ready).length ?? 0;
          const total     = p.status?.containerStatuses?.length ?? 0;
          const startTime = p.metadata?.creationTimestamp;
          const age       = startTime
            ? Math.floor((Date.now() - new Date(startTime).getTime()) / 60000) + "m"
            : "?";
          return `${p.metadata?.name}  ${phase}  ${ready}/${total}  ${age}`;
        });
        return {
          content: [{ type: "text", text: `NAME  STATUS  READY  AGE\n${rows.join("\n")}` }],
        };
      }

      // ── get_logs ────────────────────────────────────────────────────────────
      case "get_logs": {
        const pod       = args?.pod_name as string;
        const ns        = (args?.namespace as string) || "default";
        const container = args?.container as string | undefined;
        const tailLines = (args?.tail_lines as number) || 100;
        const res = await k8sApi.readNamespacedPodLog(
          pod, ns, container, undefined, undefined,
          undefined, undefined, undefined, undefined, tailLines
        );
        return { content: [{ type: "text", text: res.body }] };
      }

      // ── scale_deployment ────────────────────────────────────────────────────
      case "scale_deployment": {
        const name_    = args?.name as string;
        const ns       = (args?.namespace as string) || "default";
        const replicas = args?.replicas as number;
        await appsApi.patchNamespacedDeploymentScale(name_, ns, {
          spec: { replicas },
        }, undefined, undefined, undefined, undefined, undefined, {
          headers: { "Content-Type": "application/strategic-merge-patch+json" },
        });
        return {
          content: [{ type: "text", text: `Deployment "${name_}" scaled to ${replicas} replicas.` }],
        };
      }

      // ── describe_node ───────────────────────────────────────────────────────
      case "describe_node": {
        const res  = await k8sApi.readNode(args?.node_name as string);
        const node = res.body;
        const cap  = node.status?.capacity ?? {};
        const alloc = node.status?.allocatable ?? {};
        const taints = (node.spec?.taints ?? [])
          .map(t => `${t.key}=${t.value ?? ""}:${t.effect}`).join(", ") || "none";
        const conditions = (node.status?.conditions ?? [])
          .map(c => `${c.type}=${c.status}`).join(", ");
        return {
          content: [{
            type: "text",
            text: [
              `Node: ${node.metadata?.name}`,
              `CPU capacity: ${cap.cpu}  allocatable: ${alloc.cpu}`,
              `Memory capacity: ${cap.memory}  allocatable: ${alloc.memory}`,
              `Pods max: ${cap.pods}`,
              `Taints: ${taints}`,
              `Conditions: ${conditions}`,
            ].join("\n"),
          }],
        };
      }

      // ── apply_manifest ──────────────────────────────────────────────────────
      case "apply_manifest": {
        const yaml     = await import("js-yaml");
        const manifest = yaml.load(args?.manifest as string) as Record<string, unknown>;
        const ns       = (args?.namespace as string) || (manifest?.metadata as any)?.namespace || "default";
        if (manifest.kind === "Deployment") {
          await appsApi.createNamespacedDeployment(ns, manifest as k8s.V1Deployment);
          return { content: [{ type: "text", text: `Deployment applied in namespace "${ns}".` }] };
        }
        throw new Error(`apply_manifest: kind "${manifest.kind}" not yet supported`);
      }

      default:
        throw new Error(`Unknown tool: ${name}`);
    }
  } catch (err: any) {
    return {
      content: [{ type: "text", text: `Error: ${err.message}` }],
      isError: true,
    };
  }
});

// ─── Start ───────────────────────────────────────────────────────────────────
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("k8s-mcp-server running on stdio");
}

main().catch(console.error);
```

---

## 3. RBAC Manifest (`k8s/rbac.yaml`)

Least-privilege ServiceAccount — read-only on most resources, write only on scale.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mcp-server
  namespace: mcp-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: mcp-server-role
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
  name: mcp-server-binding
subjects:
  - kind: ServiceAccount
    name: mcp-server
    namespace: mcp-system
roleRef:
  kind: ClusterRole
  name: mcp-server-role
  apiGroup: rbac.authorization.k8s.io
```

---

## 4. Deployment Manifest (`k8s/deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-server
  namespace: mcp-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mcp-server
  template:
    metadata:
      labels:
        app: mcp-server
    spec:
      serviceAccountName: mcp-server   # picks up in-cluster auth automatically
      containers:
        - name: server
          image: your-registry/k8s-mcp-server:latest
          resources:
            requests: { cpu: "100m", memory: "128Mi" }
            limits:   { cpu: "500m", memory: "256Mi" }
          readinessProbe:
            exec:
              command: ["node", "-e", "process.exit(0)"]
            initialDelaySeconds: 5
```

---

## 5. Claude Desktop Config (`~/.claude/claude_desktop_config.json`)

```json
{
  "mcpServers": {
    "kubernetes": {
      "command": "node",
      "args": ["dist/index.js"],
      "cwd": "/path/to/k8s-mcp-server"
    }
  }
}
```

---

## Example Claude Conversations

| Prompt | Tool called |
|---|---|
| "List all crashing pods in production" | `get_pods` namespace=production |
| "Show me the last 50 lines of the api-server pod logs" | `get_logs` tail_lines=50 |
| "Scale the frontend deployment to 5 replicas" | `scale_deployment` replicas=5 |
| "What is node worker-3's available memory?" | `describe_node` node_name=worker-3 |
| "Deploy this Deployment YAML: …" | `apply_manifest` |

---

## Production Checklist

**Security** — always run with a dedicated ServiceAccount using RBAC least-privilege. Never mount a `cluster-admin` kubeconfig inside the Pod.

**Validation** — use `zod` schemas on every tool input before touching the k8s API. The `scale_deployment` example caps replicas at 50 — add similar guards everywhere.

**Transport** — use stdio for local dev (Claude Desktop). Switch to SSE (`SSEServerTransport`) for multi-user or remote deployments, behind an mTLS-authenticated gateway.

**Namespacing** — scope every API call to an explicit namespace. Never default to cluster-wide calls unless the tool genuinely requires it.

**Error surfaces** — return `isError: true` on failures so Claude treats them as tool errors rather than silently empty results.
