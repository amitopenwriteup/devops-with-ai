# ☸️ Copilot Studio Agent — Kubernetes kind Cluster Setup
### *Build an AI Agent that provisions, configures & troubleshoots kind clusters*

---

## 📋 Agent Overview

| Detail | Info |
|--------|------|
| **Agent Name** | `KubeKind Assistant` |
| **Platform** | Microsoft Copilot Studio (Free Tier) |
| **Target** | Kubernetes via kind (Kubernetes in Docker) |
| **Topics Covered** | Install, Create Cluster, Deploy Apps, Debug, Teardown |

---

## 🧠 What This Agent Does

```
User (Plain English)          KubeKind Assistant             Your Machine
─────────────────────         ──────────────────             ────────────
"Set up a kind cluster"  ──►  Generates install steps   ──►  kind create cluster
"Add 3 worker nodes"     ──►  Outputs kind config YAML  ──►  kubectl apply
"My pod is crashing"     ──►  Diagnoses + fix commands  ──►  kubectl describe
"Tear down cluster"      ──►  Safe delete sequence      ──►  kind delete cluster
```

---

## 🏗️ Step 1: Create the Agent in Copilot Studio

1. Go to [https://copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Click the **`Agent`** button on the home screen
3. In the description box type:

   > *"You are a Kubernetes expert assistant that helps users install, configure, and troubleshoot kind (Kubernetes in Docker) clusters. You generate kubectl commands, kind config YAML files, and debug pod/node issues."*

4. Click **→** — the agent `KubeKind Assistant` is created
5. Go to **Topics** tab to add the topics below

---

## 📂 Step 2: Agent Topics to Create

Create **5 Topics** — one for each workflow:

| # | Topic Name | Description Field |
|---|-----------|-------------------|
| 1 | `Install kind` | User wants to install kind or kubectl on their machine |
| 2 | `Create Cluster` | User wants to create a new kind Kubernetes cluster |
| 3 | `Deploy Application` | User wants to deploy an app or manifest to the kind cluster |
| 4 | `Debug Cluster` | User has a problem, error, or failing pod in their kind cluster |
| 5 | `Delete Cluster` | User wants to stop, reset, or delete a kind cluster |

---

## 🔧 Topic 1: Install kind

### Describe what the topic does:
```
User wants to install kind (Kubernetes in Docker) or kubectl on Linux, Mac, or Windows
```

### Conversation Flow:

```
[The agent chooses]
  "User wants to install kind"
       ↓
[Question node]
  "Which operating system are you using?"
  Quick replies: Linux | macOS | Windows
  Save as: {Topic.OS}
       ↓
[Create generative answers]
  Input: Activity.Text
  ✅ Web search ON
  ✅ General knowledge ON
  Custom prompt (paste below)
```

### Customize Your Prompt (paste into 0/8000 box):

```
The user wants to install kind and kubectl on {Topic.OS}.

Generate step-by-step installation commands for:
1. Docker (prerequisite) — verify or install
2. kind — latest stable version
3. kubectl — matching Kubernetes version

Rules:
- Use official install sources only (kind.sigs.k8s.io, kubernetes.io)
- Include version verification commands after each install
- For Windows: use both PowerShell and Chocolatey options
- For macOS: use Homebrew as primary method
- For Linux: use curl method + package manager fallback
- End with: kind version && kubectl version --client
- Wrap all commands in code blocks
```

### Generated Output Example — Linux:

```bash
# ── Step 1: Verify Docker is running ──────────────────────
docker --version
# Expected: Docker version 24.x.x or later
# If not installed: https://docs.docker.com/engine/install/

# ── Step 2: Install kind ──────────────────────────────────
# Download latest kind binary
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version
# Expected: kind v0.23.0 go1.21.x linux/amd64

# ── Step 3: Install kubectl ───────────────────────────────
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client
# Expected: Client Version: v1.30.x

# ── Step 4: Verify everything ─────────────────────────────
kind version && kubectl version --client
docker info | grep "Server Version"
```

### Generated Output Example — macOS:

```bash
# ── Step 1: Install via Homebrew ──────────────────────────
brew update
brew install kind
brew install kubectl

# Verify
kind version
kubectl version --client

# ── Step 2: Verify Docker Desktop is running ──────────────
docker info | grep "Server Version"
# If Docker not running: open Docker Desktop app first
```

### Generated Output Example — Windows (PowerShell):

```powershell
# ── Option A: Chocolatey ──────────────────────────────────
choco install kind -y
choco install kubernetes-cli -y

# ── Option B: Manual (PowerShell as Admin) ────────────────
curl.exe -Lo kind-windows-amd64.exe `
  https://kind.sigs.k8s.io/dl/v0.23.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe C:\Windows\kind.exe

# Verify
kind version
kubectl version --client
```

---

## 🔧 Topic 2: Create Cluster

### Describe what the topic does:
```
User wants to create a new kind Kubernetes cluster with single or multi-node setup
```

### Conversation Flow:

```
[The agent chooses]
  "User wants to create a kind cluster"
       ↓
[Question node]
  "What type of cluster do you need?"
  Quick replies:
    Single Node (default) |
    Multi-Node (1 control + workers) |
    HA Control Plane (3 control nodes)
  Save as: {Topic.ClusterType}
       ↓
[Question node]
  "What will you name your cluster?"
  (free text — default: 'kind')
  Save as: {Topic.ClusterName}
       ↓
[Create generative answers]
  Input: Activity.Text
  ✅ Web search ON
  ✅ General knowledge ON
  Custom prompt (paste below)
```

### Customize Your Prompt:

```
Generate a complete kind cluster setup for:
- Cluster type: {Topic.ClusterType}
- Cluster name: {Topic.ClusterName}

Output:
1. The kind-config.yaml file content (complete, with all fields)
2. The exact command to create the cluster using this config
3. Commands to verify the cluster is healthy:
   - kind get clusters
   - kubectl cluster-info --context kind-{Topic.ClusterName}
   - kubectl get nodes -o wide
4. How to set kubectl context to this cluster

Always include:
- Resource limits per node (memory/CPU requests)
- Port mappings for accessing services locally
- Labels on worker nodes for workload targeting
- Comments in the YAML explaining each field
```

### Generated Output — Single Node:

```bash
# Single-node kind cluster (1 control-plane = also schedules workloads)
kind create cluster --name my-cluster

# Verify
kubectl cluster-info --context kind-my-cluster
kubectl get nodes
```

### Generated Output — Multi-Node (1 control + 2 workers):

**`kind-config.yaml`**
```yaml
# kind Cluster Configuration — Multi-Node
# 1 control-plane node + 2 worker nodes
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster

name: my-cluster   # Change to your cluster name

nodes:
  # ── Control Plane Node ──────────────────────────────────
  - role: control-plane
    # Expose port 80 and 443 for ingress
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
    # Mount a local directory into the node
    extraMounts:
      - hostPath: /tmp/kind-data
        containerPath: /data

  # ── Worker Node 1 ───────────────────────────────────────
  - role: worker
    labels:
      node-type: worker
      zone: a

  # ── Worker Node 2 ───────────────────────────────────────
  - role: worker
    labels:
      node-type: worker
      zone: b

# ── Networking ──────────────────────────────────────────────
networking:
  # Disable default CNI to use a custom one (e.g. Calico)
  # disableDefaultCNI: false   # uncomment to use custom CNI
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

```bash
# Create cluster from config
kind create cluster --config kind-config.yaml

# Verify all nodes are Ready
kubectl get nodes -o wide

# Expected output:
# NAME                       STATUS   ROLES           AGE
# my-cluster-control-plane   Ready    control-plane   60s
# my-cluster-worker          Ready    <none>          45s
# my-cluster-worker2         Ready    <none>          45s

# Set context
kubectl config use-context kind-my-cluster
```

### Generated Output — HA Control Plane (3 control + 2 workers):

**`kind-ha-config.yaml`**
```yaml
# kind HA Cluster — 3 control-plane nodes + 2 workers
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster

name: ha-cluster

nodes:
  # ── Control Plane Nodes (3 for HA) ──────────────────────
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
  - role: control-plane
  - role: control-plane

  # ── Worker Nodes ─────────────────────────────────────────
  - role: worker
    labels:
      tier: app
  - role: worker
    labels:
      tier: app
```

```bash
# Create HA cluster
kind create cluster --config kind-ha-config.yaml

# List all nodes
kubectl get nodes

# Expected: 5 nodes (3 control-plane + 2 worker)
```

---

## 🔧 Topic 3: Deploy Application

### Describe what the topic does:
```
User wants to deploy an application, pod, deployment, or service to their kind cluster
```

### Conversation Flow:

```
[The agent chooses]
  "User wants to deploy an app to kind"
       ↓
[Question node]
  "What do you want to deploy?"
  Quick replies:
    Sample NGINX app |
    Custom Docker image |
    Full app with Service + Ingress |
    Helm chart
  Save as: {Topic.DeployType}
       ↓
[Create generative answers]
  Input: Activity.Text
  ✅ Web search ON
  ✅ General knowledge ON
  Custom prompt (paste below)
```

### Customize Your Prompt:

```
Generate Kubernetes manifests and kubectl commands to deploy
{Topic.DeployType} on a local kind cluster.

Always include:
1. Namespace creation (never deploy to default namespace)
2. Deployment manifest with resource limits
3. Service manifest (ClusterIP + NodePort for local access)
4. Commands to load a local Docker image into kind
5. Port-forward command to test locally
6. Health check commands (kubectl rollout status, kubectl get pods)

Format each manifest as a separate YAML code block.
Add comments explaining every field.
```

### Generated Output — NGINX Sample App:

```bash
# ── Step 1: Load local image into kind ────────────────────
# (only needed for locally-built images)
docker build -t my-app:latest .
kind load docker-image my-app:latest --name my-cluster
```

**`namespace.yaml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
  labels:
    env: workshop
```

**`deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo
  labels:
    app: nginx-demo
spec:
  replicas: 2                   # Run 2 pod replicas
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25-alpine   # Lightweight alpine image
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "100m"
            limits:
              memory: "128Mi"
              cpu: "200m"
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 20
```

**`service.yaml`**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo-svc
  namespace: demo
spec:
  selector:
    app: nginx-demo           # Matches pod labels
  ports:
    - name: http
      protocol: TCP
      port: 80                # Service port
      targetPort: 80          # Pod port
      nodePort: 30080         # Access via localhost:30080
  type: NodePort
```

```bash
# ── Step 2: Apply all manifests ───────────────────────────
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# ── Step 3: Verify deployment ─────────────────────────────
kubectl rollout status deployment/nginx-demo -n demo
kubectl get pods -n demo -o wide
kubectl get svc -n demo

# ── Step 4: Access the app ────────────────────────────────
# Option A: Port forward
kubectl port-forward svc/nginx-demo-svc 8080:80 -n demo
# Open: http://localhost:8080

# Option B: NodePort (if port mapping was set in kind-config)
curl http://localhost:30080
```

---

## 🔧 Topic 4: Debug Cluster

### Describe what the topic does:
```
User has a problem, error, or failing pod, node, or service in their kind Kubernetes cluster
```

### Conversation Flow:

```
[The agent chooses]
  "User has a Kubernetes or kind cluster error to debug"
       ↓
[Question node]
  "What is the issue you're seeing?"
  Quick replies:
    Pod not starting / CrashLoopBackOff |
    ImagePullBackOff |
    Node NotReady |
    Service not reachable |
    Paste my error log
  Save as: {Topic.IssueType}
       ↓
[Create generative answers]
  Input: Activity.Text
  ✅ Web search ON
  ✅ General knowledge ON
  Custom prompt (paste below)
```

### Customize Your Prompt:

```
The user has a Kubernetes kind cluster issue: {Topic.IssueType}

Provide:
1. ROOT CAUSE — what causes this issue in kind clusters specifically
2. DIAGNOSIS COMMANDS — exact kubectl commands to investigate
3. STEP-BY-STEP FIX — numbered list of commands to resolve it
4. PREVENTION — what to add to manifests or kind config to prevent recurrence
5. QUICK CHECK — one-liner to verify the fix worked

Format with clear markdown headers. All commands in code blocks.
Focus on kind-specific quirks (image loading, port mapping, CNI issues).
```

### Common Debug Scenarios — AI Responses:

#### 🔴 CrashLoopBackOff

```bash
# ── Diagnose ──────────────────────────────────────────────
kubectl get pods -n <namespace>
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> -f

# ── Common Fixes ──────────────────────────────────────────
# 1. Check resource limits (OOMKilled)
kubectl describe pod <pod-name> | grep -A5 "Limits\|OOM"

# 2. Check liveness probe settings
# Increase initialDelaySeconds in your deployment:
#   livenessProbe:
#     initialDelaySeconds: 30   ← increase this

# 3. Check for missing ConfigMap or Secret
kubectl get configmap -n <namespace>
kubectl get secret -n <namespace>
```

#### 🔴 ImagePullBackOff (most common in kind)

```bash
# ── Root Cause (kind-specific) ────────────────────────────
# kind nodes cannot pull local Docker images directly.
# You MUST load local images into kind manually.

# ── Fix ───────────────────────────────────────────────────
# Step 1: Build your image locally
docker build -t my-app:latest .

# Step 2: Load into kind cluster
kind load docker-image my-app:latest --name <cluster-name>

# Step 3: Set imagePullPolicy in deployment to Never
# (prevents Kubernetes from trying to pull from registry)
# In your deployment.yaml:
#   image: my-app:latest
#   imagePullPolicy: Never    ← add this line

# Step 4: Verify image is in kind node
docker exec -it <cluster-name>-control-plane crictl images | grep my-app

# ── Verify Fix ────────────────────────────────────────────
kubectl rollout restart deployment/<name> -n <namespace>
kubectl get pods -n <namespace> -w
```

#### 🔴 Node NotReady

```bash
# ── Diagnose ──────────────────────────────────────────────
kubectl get nodes
kubectl describe node <node-name>
kubectl get events --sort-by='.lastTimestamp' -A

# Check kind containers are running
docker ps | grep kind

# ── Common Fixes ──────────────────────────────────────────
# 1. Docker ran out of memory — increase Docker Desktop memory to 4GB+
# 2. Restart the kind node container
docker restart <cluster-name>-worker

# 3. If CNI is broken, reapply
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# ── Verify ────────────────────────────────────────────────
kubectl get nodes -w
# Wait until STATUS = Ready
```

#### 🔴 Service Not Reachable Locally

```bash
# ── Root Cause (kind-specific) ────────────────────────────
# NodePort services in kind are NOT directly accessible via localhost
# unless extraPortMappings was set in kind-config.yaml at cluster creation.

# ── Fix Option A: Port Forward (always works) ─────────────
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
# Access: http://localhost:8080

# ── Fix Option B: Recreate cluster with port mapping ──────
# Add this to kind-config.yaml before creating:
#   nodes:
#     - role: control-plane
#       extraPortMappings:
#         - containerPort: 30080
#           hostPort: 8080

# ── Fix Option C: Install ingress-nginx for kind ──────────
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

---

## 🔧 Topic 5: Delete Cluster

### Describe what the topic does:
```
User wants to stop, reset, delete, or teardown a kind Kubernetes cluster
```

### Conversation Flow:

```
[The agent chooses]
  "User wants to delete or reset a kind cluster"
       ↓
[Question node]
  "Do you want to delete a specific cluster or all clusters?"
  Quick replies: Delete one cluster | Delete ALL clusters | Just list my clusters
  Save as: {Topic.DeleteType}
       ↓
[Create generative answers]
  Input: Activity.Text
  ✅ General knowledge ON
  Custom prompt (paste below)
```

### Customize Your Prompt:

```
Generate safe teardown commands for kind cluster deletion: {Topic.DeleteType}

Include:
1. List clusters first (so user confirms the right one)
2. Clean up any PersistentVolumes or local mounts first
3. The delete command
4. Verify deletion (cluster no longer in kubeconfig)
5. Optional: prune Docker volumes/images created by kind
```

### Generated Output:

```bash
# ── Step 1: List all kind clusters ────────────────────────
kind get clusters

# ── Step 2: Delete a specific cluster ─────────────────────
kind delete cluster --name my-cluster

# ── Step 3: Delete ALL clusters ───────────────────────────
kind delete clusters --all

# ── Step 4: Verify cluster removed from kubeconfig ────────
kubectl config get-contexts | grep kind
# Should show nothing

# ── Step 5: Optional — clean up Docker resources ──────────
# Remove kind node images (frees disk space)
docker images | grep kindest | awk '{print $3}' | xargs docker rmi -f

# Remove dangling volumes
docker volume prune -f

# ── Verify ────────────────────────────────────────────────
kind get clusters
# Expected: No kind clusters found.
```

---

## ⚙️ Step 3: Configure Agent Instructions (Overview tab)

Go to **Overview → Instructions** and paste this as the agent's base behaviour:

```
You are KubeKind Assistant — a Kubernetes expert specializing in
local development clusters using kind (Kubernetes in Docker).

Your capabilities:
- Guide users to install kind and kubectl on any OS
- Generate kind cluster config YAML for any topology
- Create Kubernetes manifests (Deployment, Service, Ingress, ConfigMap)
- Load Docker images into kind clusters
- Debug CrashLoopBackOff, ImagePullBackOff, NotReady, and networking issues
- Provide teardown and reset commands

Always:
- Output commands in code blocks with # comments
- Mention kind-specific quirks (image loading, port mapping)
- Include verification commands after every action
- Prefer kubectl over k8s shorthand for clarity
- Keep responses focused on kind local clusters, not cloud providers

Never:
- Suggest paid cloud services as primary solutions
- Hardcode sensitive values — use ConfigMaps and Secrets
- Skip the "verify" step after any cluster operation
```

---

## 🧪 Step 4: Test Your Agent

Use the **"Test your agent"** panel on the right side of Copilot Studio:

### Test Script

| Type This | Expected Agent Response |
|-----------|------------------------|
| *"Install kind on Linux"* | curl install commands for kind + kubectl |
| *"Create a 3-node kind cluster called dev"* | kind-config.yaml + create command |
| *"Deploy nginx to my cluster"* | namespace + deployment + service YAMLs |
| *"My pod is in CrashLoopBackOff"* | kubectl describe + logs + fix steps |
| *"Load my local Docker image into kind"* | kind load docker-image command |
| *"Delete my dev cluster"* | kind delete cluster --name dev |

---

## 📊 Agent Topic Summary

| Topic | Trigger Description | Key Output |
|-------|--------------------|-----------| 
| Install kind | Install kind or kubectl on any OS | OS-specific install commands |
| Create Cluster | Create a new kind cluster | kind-config.yaml + create cmd |
| Deploy Application | Deploy app to kind cluster | Deployment + Service YAMLs |
| Debug Cluster | Fix pod/node/service errors | Diagnosis + fix commands |
| Delete Cluster | Remove kind clusters | Safe teardown sequence |

---

## 🔗 Reference Commands Cheat Sheet

```bash
# ── Cluster Management ────────────────────────────────────
kind create cluster --name <name>                    # Create default cluster
kind create cluster --config config.yaml             # Create from config
kind get clusters                                    # List all clusters
kind delete cluster --name <name>                    # Delete cluster
kind export logs --name <name> ./kind-logs           # Export logs for debug

# ── Image Management ──────────────────────────────────────
kind load docker-image <image>:<tag> --name <name>   # Load local image
docker exec <node> crictl images                     # List images in node

# ── kubectl Context ───────────────────────────────────────
kubectl config get-contexts                          # List all contexts
kubectl config use-context kind-<name>               # Switch to kind context
kubectl cluster-info --context kind-<name>           # Cluster details

# ── Health Checks ─────────────────────────────────────────
kubectl get nodes -o wide                            # Node status
kubectl get pods -A                                  # All pods all namespaces
kubectl get events --sort-by='.lastTimestamp' -A     # Recent events
kubectl top nodes                                    # Resource usage
```

---

## 🔗 Resources

- **kind Official Docs:** https://kind.sigs.k8s.io/
- **kind Config Reference:** https://kind.sigs.k8s.io/docs/user/configuration/
- **kubectl Cheat Sheet:** https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **Copilot Studio Docs:** https://learn.microsoft.com/en-us/microsoft-copilot-studio/

---

*KubeKind Assistant — AI-powered local Kubernetes cluster management.*
*All tools used are free and open-source.*
