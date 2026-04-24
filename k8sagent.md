#  KubeKind Assistant — Full Beginner Setup Guide
### *Step-by-step: Build an AI Agent for Kubernetes kind Clusters in Copilot Studio*

---

##  What You Will Build

A **Copilot Studio AI Agent** called `KubeKind Assistant` that helps you:

- Install kind and kubectl on any OS
- Create single-node and multi-node kind clusters
- Deploy apps to your cluster
- Debug crashed pods and errors
- Safely delete clusters

>  **You do NOT need coding experience.** Every click, every field, every paste is shown below.

---

## 🗺️ Full Agent Map

```
KubeKind Assistant
│
├── Topic 1 → Install kind          "How do I install kind?"
├── Topic 2 → Create Cluster        "Create a kind cluster"
├── Topic 3 → Deploy Application    "Deploy nginx to my cluster"
├── Topic 4 → Debug Cluster         "My pod is crashing"
└── Topic 5 → Delete Cluster        "Delete my cluster"
```

Each Topic follows this **same 3-node pattern**:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
│   TRIGGER    │ ──► │   QUESTION   │ ──► │  GENERATIVE ANSWERS  │
│ (auto-made)  │     │ (you fill)   │     │  (AI generates output│
└──────────────┘     └──────────────┘     └──────────────────────┘
```

---

##  PART 0 — Create the Agent

### Step 0.1 — Open Copilot Studio

1. Go to  [https://copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Sign in with your **free Microsoft account** (Outlook / Hotmail)
3. You will see: *"What would you like to build?"*

### Step 0.2 — Create the Agent

1. Click the blue **`Agent`** button
2. In the big text box type exactly:

```
You are a Kubernetes expert assistant that helps users install,
configure, and troubleshoot kind (Kubernetes in Docker) clusters.
You generate kubectl commands, kind config YAML files, and debug
pod and node issues.
```

3. Click the **→ arrow** button
4. Your agent **`KubeKind Assistant`** is created ✅

### Step 0.3 — Set Agent Instructions

1. Click **Overview** tab at the top
2. Scroll down to find **Instructions**
3. Clear whatever is there and paste this:

```
You are KubeKind Assistant — a Kubernetes expert for local kind clusters.

Always:
- Output commands in code blocks with # comments
- Mention kind-specific tips (image loading, port mapping)
- Include verify commands after every action
- Keep responses focused on kind local clusters

Never:
- Suggest paid cloud services
- Hardcode passwords or secrets
- Skip the verify step
```

4. Click **Save** (top right) 

---

## 📂 PART 1 — Topic 1: Install kind

### What this topic does:
User asks how to install kind or kubectl → Agent asks which OS → Generates install commands

---

### Step 1.1 — Create the Topic

1. Click **Topics** in the top navigation bar
2. Click **`+ New Topic`** → **`From blank`**
3. At the top, click the topic name and rename it to:
   ```
   Install kind
   ```

---

### Step 1.2 — Configure the Trigger Node

You will see a **Trigger** node already on the canvas.

1. Click **Edit** inside the Trigger node
2. In the **"Describe what the topic does"** box, paste:

```
This tool can handle queries like these: install kind,
how do I install kind, setup kind tool,
kind installation steps, guide to installing kind,
install kubectl, setup kubernetes tools
```

---

### Step 1.3 — Configure the Question Node

A **Question** node already exists below the Trigger. Click on it to edit.

**Field 1 — Question text:**
Click inside the text area and type:
```
Which operating system are you using?
```

**Field 2 — Identify:**
Click the dropdown → select:
```
Multiple choice options
```

**Field 3 — Options for user:**
Add exactly 3 options (click **+ New option** to add more):
```
Option 1: Linux
Option 2: Mac
Option 3: Windows
```

**Field 4 — Save user response as:**
Click the variable box at the bottom → rename it to:
```
OSChoice
```

Your Question node should look like this:
```
┌─────────────────────────────────────────────┐
│  ❓ Question                                 │
│                                             │
│  Which operating system are you using?      │
│                                             │
│  Identify: Multiple choice options          │
│                                             │
│  Options:  [ Linux ]  [ Mac ]  [ Windows ] │
│                                             │
│  Save as:  {x} OSChoice   choice           │
└─────────────────────────────────────────────┘
```

Click **Save** (top right) 💾

---

### Step 1.4 — Add the Generative Answers Node

1. Click the **`+`** blue circle button at the bottom of the Question node
2. A menu appears → click **`Advanced`**
3. Click **`Generative answers`**
4. A new **"Create generative answers"** node appears

**Inside the new node — fix the Input field:**

1. Click inside the **"Enter or select a value"** Input box
2. Click the **`···`** three-dot button on the right
3. Select **`System variable`**
4. Select **`Activity.Text`**

The input box should now show:
```
Input
┌───────────────────────┐
│  Activity.Text        │
└───────────────────────┘
```

**Configure Data Sources:**

1. Click **`Edit`** under Data sources (globe icon)
2. A properties panel opens on the right side
3. Set these toggles:

```
Search only selected sources     ⚫ OFF  (leave off)
Web search                       🔵 ON   (toggle blue)
Allow AI general knowledge       🔵 ON   (toggle blue)  ← very important!
```

4. Scroll down to the **"Customize your prompt"** box (shows 0/8000)
5. Paste this prompt:

```
The user wants to install kind on {Topic.OSChoice}.

Generate step-by-step installation commands for:
1. Docker (prerequisite) — show how to verify Docker is running
2. kind — latest stable version from kind.sigs.k8s.io
3. kubectl — latest stable version from kubernetes.io

Rules:
- For Linux: use curl method
- For Mac: use Homebrew (brew install kind)
- For Windows: use Chocolatey AND manual PowerShell method
- Include version check command after each install
- End with: kind version && kubectl version --client
- Wrap all commands in code blocks with # comments
```

6. Click **Save** (top right) 

---

### Step 1.5 — Test Topic 1

In the **"Test your agent"** panel on the right side, type:
```
Install kind on Linux
```

 Expected: Step-by-step install commands with code blocks

---

###  Topic 1 Complete — Generated Output Examples

#### Linux Install Commands (Agent Output):

```bash
# ── Step 1: Verify Docker is running ──────────────────────
docker --version
# Expected: Docker version 24.x.x or later

# ── Step 2: Install kind ──────────────────────────────────
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Verify
kind version

# ── Step 3: Install kubectl ───────────────────────────────
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# Verify everything
kind version && kubectl version --client
```

#### macOS Install Commands (Agent Output):

```bash
# ── Install via Homebrew ───────────────────────────────────
brew update
brew install kind
brew install kubectl

# Verify
kind version
kubectl version --client
```

#### Windows Install Commands (Agent Output):

```powershell
# ── Option A: Chocolatey (run as Admin) ───────────────────
choco install kind -y
choco install kubernetes-cli -y

# ── Option B: Manual PowerShell ───────────────────────────
curl.exe -Lo kind-windows-amd64.exe `
  https://kind.sigs.k8s.io/dl/v0.23.0/kind-windows-amd64
Move-Item .\kind-windows-amd64.exe C:\Windows\kind.exe

# Verify
kind version
kubectl version --client
```

---

## PART 2 — Topic 2: Create Cluster

### What this topic does:
User asks to create a cluster → Agent asks what type → Generates kind-config.yaml + commands

---

### Step 2.1 — Create the Topic

1. Click **Topics** → **`+ New Topic`** → **`From blank`**
2. Rename it to:
   ```
   Create Cluster
   ```

---

### Step 2.2 — Configure the Trigger Node

In the **"Describe what the topic does"** box, paste:

```
User wants to create a new kind Kubernetes cluster,
set up a cluster, make a multi-node cluster,
create a single node cluster, or create an HA cluster
```

---

### Step 2.3 — Configure Question Node 1 (Cluster Type)

Edit the Question node:

**Question text:**
```
What type of cluster do you need?
```

**Identify:** Multiple choice options

**Options for user:**
```
Option 1: Single Node
Option 2: Multi-Node (1 control + 2 workers)
Option 3: HA (3 control plane nodes)
```

**Save user response as:** `ClusterType`

---

### Step 2.4 — Add Question Node 2 (Cluster Name)

Click **`+`** below Question 1 → **`Ask a question`**

**Question text:**
```
What would you like to name your cluster?
(press Enter to use the default name: kind)
```

**Identify:** User's entire response

**Save user response as:** `ClusterName`

---

### Step 2.5 — Add Generative Answers Node

Click **`+`** → **`Advanced`** → **`Generative answers`**

**Input:** `Activity.Text`

**Data Sources → Edit:**
```
Web search           ON
General knowledge    ON
```

**Customize your prompt (paste into 0/8000 box):**

```
Generate a complete kind cluster setup for:
- Cluster type: {Topic.ClusterType}
- Cluster name: {Topic.ClusterName}

Output in this order:
1. The complete kind-config.yaml file with comments on every line
2. The exact terminal command to create the cluster
3. Commands to verify the cluster is healthy:
   - kind get clusters
   - kubectl cluster-info --context kind-{Topic.ClusterName}
   - kubectl get nodes -o wide
4. How to switch kubectl to use this cluster

Always include port mappings (80, 443) and node labels in the config.
```

Click **Save** 💾

---

### 📦 Topic 2 Complete — Generated Output Examples

#### Single Node (Agent Output):

```bash
# Create default single-node cluster
kind create cluster --name my-cluster

# Verify
kind get clusters
kubectl cluster-info --context kind-my-cluster
kubectl get nodes
```

#### Multi-Node — kind-config.yaml (Agent Output):

```yaml
# kind Cluster Config — 1 control-plane + 2 workers
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster

name: my-cluster

nodes:
  # Control plane — manages the cluster
  - role: control-plane
    extraPortMappings:
      - containerPort: 80    # HTTP traffic
        hostPort: 80
        protocol: TCP
      - containerPort: 443   # HTTPS traffic
        hostPort: 443
        protocol: TCP

  # Worker node 1 — runs your applications
  - role: worker
    labels:
      node-type: worker
      zone: a

  # Worker node 2 — runs your applications
  - role: worker
    labels:
      node-type: worker
      zone: b

networking:
  podSubnet: "10.244.0.0/16"      # IP range for pods
  serviceSubnet: "10.96.0.0/12"   # IP range for services
```

```bash
# Create cluster from config file
kind create cluster --config kind-config.yaml

# Verify all 3 nodes are Ready
kubectl get nodes -o wide

# Switch context to this cluster
kubectl config use-context kind-my-cluster
```

---

##  PART 3 — Topic 3: Deploy Application

### What this topic does:
User wants to deploy an app → Agent asks what type → Generates Kubernetes YAML manifests

---

### Step 3.1 — Create the Topic

1. Click **Topics** → **`+ New Topic`** → **`From blank`**
2. Rename it to:
   ```
   Deploy Application
   ```

---

### Step 3.2 — Configure the Trigger Node

In the **"Describe what the topic does"** box, paste:

```
User wants to deploy an application, pod, container,
or service to their kind Kubernetes cluster,
run nginx on kind, deploy docker image to kind
```

---

### Step 3.3 — Configure the Question Node

**Question text:**
```
What would you like to deploy?
```

**Identify:** Multiple choice options

**Options for user:**
```
Option 1: Sample NGINX app
Option 2: My own Docker image
Option 3: App with Ingress
Option 4: Helm chart
```

**Save user response as:** `DeployType`

---

### Step 3.4 — Add Generative Answers Node

Click **`+`** → **`Advanced`** → **`Generative answers`**

**Input:** `Activity.Text`

**Data Sources → Edit:**
```
Web search          🔵 ON
General knowledge   🔵 ON
```

**Customize your prompt (paste into 0/8000 box):**

```
Generate Kubernetes manifests to deploy {Topic.DeployType} on kind cluster.

Output in this exact order:
1. Command to load local Docker image into kind (kind load docker-image)
2. namespace.yaml — with labels
3. deployment.yaml — with resource limits, readiness probe, liveness probe
4. service.yaml — NodePort type for local access
5. kubectl apply commands for each file
6. kubectl port-forward command to test in browser
7. kubectl rollout status to verify it worked

Add # comments on every YAML field explaining what it does.
Mention the kind-specific step: imagePullPolicy: Never for local images.
```

Click **Save** 

---

###  Topic 3 Complete — Generated Output Example

```bash
# ── Step 1: Load your image into kind ─────────────────────
# (only needed for locally-built images, skip for Docker Hub images)
docker build -t my-app:latest .
kind load docker-image my-app:latest --name my-cluster
```

**namespace.yaml**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo          # All resources go in this namespace
  labels:
    env: workshop
```

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: demo
spec:
  replicas: 2                   # Run 2 copies of the pod
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
          image: nginx:1.25-alpine
          imagePullPolicy: Never  # ← kind-specific: use local image
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"    # Minimum memory needed
              cpu: "100m"       # 0.1 CPU core
            limits:
              memory: "128Mi"   # Maximum memory allowed
              cpu: "200m"       # 0.2 CPU core
          readinessProbe:       # Is pod ready to receive traffic?
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
          livenessProbe:        # Is pod still alive?
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
```

**service.yaml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  namespace: demo
spec:
  selector:
    app: nginx-demo       # Connect to pods with this label
  ports:
    - port: 80            # Service listens on port 80
      targetPort: 80      # Forwards to pod port 80
      nodePort: 30080     # Access from localhost:30080
  type: NodePort
```

```bash
# ── Apply all files ───────────────────────────────────────
kubectl apply -f namespace.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# ── Verify deployment is running ──────────────────────────
kubectl rollout status deployment/nginx-demo -n demo
kubectl get pods -n demo

# ── Access the app in your browser ───────────────────────
kubectl port-forward svc/nginx-svc 8080:80 -n demo
# Open: http://localhost:8080
```

---

## 📂 PART 4 — Topic 4: Debug Cluster

### What this topic does:
User has an error → Agent asks what type of error → Generates diagnosis + fix commands

---

### Step 4.1 — Create the Topic

1. Click **Topics** → **`+ New Topic`** → **`From blank`**
2. Rename it to:
   ```
   Debug Cluster
   ```

---

### Step 4.2 — Configure the Trigger Node

In the **"Describe what the topic does"** box, paste:

```
User has a problem with their kind cluster,
pod is crashing, CrashLoopBackOff, ImagePullBackOff,
node is NotReady, service not reachable,
pod not starting, kubernetes error
```

---

### Step 4.3 — Configure the Question Node

**Question text:**
```
What issue are you seeing in your cluster?
```

**Identify:** Multiple choice options

**Options for user:**
```
Option 1: CrashLoopBackOff
Option 2: ImagePullBackOff
Option 3: Node NotReady
Option 4: Service not reachable
Option 5: Paste my error log
```

**Save user response as:** `IssueType`

---

### Step 4.4 — Add Generative Answers Node

Click **`+`** → **`Advanced`** → **`Generative answers`**

**Input:** `Activity.Text`

**Data Sources → Edit:**
```
Web search          ON
General knowledge   ON
```

**Customize your prompt (paste into 0/8000 box):**

```
The user has a Kubernetes kind cluster issue: {Topic.IssueType}

Provide a debug report with these sections:

## Root Cause
What causes this in kind clusters specifically (1-2 sentences)

## Diagnosis Commands
Exact kubectl commands to investigate the problem

## Step-by-Step Fix
Numbered list of commands to resolve it

## Prevention
What to add to the manifest or kind config to prevent this

## Quick Verify
One command to confirm the fix worked

Format with markdown headers. All commands in code blocks.
Focus on kind-specific causes (image loading, port mapping, Docker memory).
```

Click **Save** 

---

###  Topic 4 Complete — Debug Output Examples

####  CrashLoopBackOff (Agent Output):

```bash
## Root Cause
The pod starts, crashes, and Kubernetes keeps restarting it.
Usually caused by a bad command, missing env variable, or wrong config.

## Diagnosis Commands
# See pod status
kubectl get pods -n <your-namespace>

# See why it crashed
kubectl describe pod <pod-name> -n <your-namespace>

# See crash logs
kubectl logs <pod-name> -n <your-namespace> --previous

## Step-by-Step Fix
# 1. Check if it ran out of memory (OOMKilled)
kubectl describe pod <pod-name> | grep -i "OOM\|killed\|limits"

# 2. If OOMKilled — increase memory limit in deployment.yaml:
#    resources:
#      limits:
#        memory: "256Mi"   ← increase this value

# 3. Check liveness probe is not too aggressive
#    livenessProbe:
#      initialDelaySeconds: 30   ← increase if app starts slowly

# 4. Restart the deployment after fixing
kubectl rollout restart deployment/<name> -n <namespace>

## Quick Verify
kubectl get pods -n <namespace> -w
# Watch until STATUS = Running (not CrashLoopBackOff)
```

#### 🔴 ImagePullBackOff — kind specific! (Agent Output):

```bash
## Root Cause
kind cluster nodes CANNOT pull local Docker images automatically.
You must manually load local images into kind using kind load.

## Diagnosis Commands
kubectl describe pod <pod-name> -n <namespace>
# Look for: "Failed to pull image" or "not found"

## Step-by-Step Fix
# 1. Build your image locally
docker build -t my-app:latest .

# 2. Load it into your kind cluster  ← this is the key step!
kind load docker-image my-app:latest --name <cluster-name>

# 3. Add this to your deployment.yaml  ← prevents pull attempts
#    image: my-app:latest
#    imagePullPolicy: Never

# 4. Reapply your deployment
kubectl apply -f deployment.yaml

## Quick Verify
kubectl get pods -n <namespace>
# STATUS should change from ImagePullBackOff → Running
```

#### 🔴 Service Not Reachable (Agent Output):

```bash
## Root Cause
In kind, NodePort services are NOT reachable on localhost
unless extraPortMappings was set when the cluster was created.

## Fix Option A — Port Forward (always works, no cluster change needed)
kubectl port-forward svc/<service-name> 8080:80 -n <namespace>
# Open: http://localhost:8080

## Fix Option B — Recreate cluster with port mapping
# Add to kind-config.yaml BEFORE creating cluster:
# nodes:
#   - role: control-plane
#     extraPortMappings:
#       - containerPort: 30080
#         hostPort: 8080

## Fix Option C — Install Ingress for kind
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

## Quick Verify
curl http://localhost:8080
# Should return your app's response
```

---

##  PART 5 — Topic 5: Delete Cluster

### What this topic does:
User wants to remove a cluster → Agent confirms which one → Generates safe delete commands

---

### Step 5.1 — Create the Topic

1. Click **Topics** → **`+ New Topic`** → **`From blank`**
2. Rename it to:
   ```
   Delete Cluster
   ```

---

### Step 5.2 — Configure the Trigger Node

In the **"Describe what the topic does"** box, paste:

```
User wants to delete, remove, stop, teardown,
or reset a kind Kubernetes cluster
```

---

### Step 5.3 — Configure the Question Node

**Question text:**
```
What would you like to do?
```

**Identify:** Multiple choice options

**Options for user:**
```
Option 1: Delete one specific cluster
Option 2: Delete ALL clusters
Option 3: Just show me my clusters
```

**Save user response as:** `DeleteType`

---

### Step 5.4 — Add Generative Answers Node

Click **`+`** → **`Advanced`** → **`Generative answers`**

**Input:** `Activity.Text`

**Data Sources → Edit:**
```
Web search           ON
General knowledge    ON
```

**Customize your prompt (paste into 0/8000 box):**

```
Generate kind cluster teardown commands for: {Topic.DeleteType}

Include steps in this order:
1. List all clusters first (confirm before deleting)
2. Clean up command
3. Verify cluster is gone from kubeconfig
4. Optional Docker cleanup to free disk space

Add safety warnings. Use code blocks.
```

Click **Save** 💾

---

### 📦 Topic 5 Complete — Generated Output:

```bash
# ── Step 1: See all your clusters first ───────────────────
kind get clusters
# Example output:
# dev-cluster
# my-cluster

# ── Step 2A: Delete ONE specific cluster ──────────────────
kind delete cluster --name dev-cluster

# ── Step 2B: Delete ALL clusters ──────────────────────────
kind delete clusters --all

# ── Step 3: Verify it is gone ─────────────────────────────
kind get clusters
# Expected: No kind clusters found.

kubectl config get-contexts | grep kind
# Should show nothing

# ── Step 4: Optional — free up disk space ─────────────────
# Remove kind Docker images
docker images | grep kindest | awk '{print $3}' | xargs docker rmi -f

# Remove unused Docker volumes
docker volume prune -f
```

---

## PART 6 — Final Checklist

After building all 5 topics, verify everything:

### Agent Checklist

| Item | Done? |
|------|-------|
| Agent created with description | ☐ |
| Agent Instructions set in Overview | ☐ |
| Topic 1: Install kind — 3 nodes built | ☐ |
| Topic 2: Create Cluster — 4 nodes built | ☐ |
| Topic 3: Deploy Application — 3 nodes built | ☐ |
| Topic 4: Debug Cluster — 3 nodes built | ☐ |
| Topic 5: Delete Cluster — 3 nodes built | ☐ |
| All Generative Answers nodes have Activity.Text input | ☐ |
| All Generative Answers nodes have General knowledge ON | ☐ |
| All Generative Answers nodes have Web search ON | ☐ |
| All topics saved | ☐ |

---

## 🧪 PART 7 — Test Script (Copy & Paste into Test Panel)

Test each topic by typing these into the **"Test your agent"** panel:

```
Test 1 → "How do I install kind on Linux?"
Test 2 → "Create a 2-worker kind cluster called dev"
Test 3 → "Deploy nginx to my kind cluster"
Test 4 → "My pod is in CrashLoopBackOff"
Test 5 → "My image won't pull — ImagePullBackOff error"
Test 6 → "I can't reach my service on localhost"
Test 7 → "Delete my cluster named dev"
```

---

## 📋 PART 8 — Quick Command Cheat Sheet

```bash
# ── Install ───────────────────────────────────────────────
kind version                              # Check kind installed
kubectl version --client                  # Check kubectl installed

# ── Cluster ───────────────────────────────────────────────
kind create cluster --name dev            # Create simple cluster
kind create cluster --config config.yaml  # Create from config file
kind get clusters                         # List all clusters
kind delete cluster --name dev            # Delete a cluster

# ── Images (kind-specific!) ───────────────────────────────
kind load docker-image my-app:v1 --name dev   # Load local image

# ── kubectl Basics ────────────────────────────────────────
kubectl get nodes                         # See cluster nodes
kubectl get pods -A                       # See all pods
kubectl get svc -A                        # See all services
kubectl describe pod <name> -n <ns>       # Debug a pod
kubectl logs <pod-name> -n <ns>           # See pod logs
kubectl logs <pod-name> -n <ns> --previous # Logs from crashed pod

# ── Access Apps ───────────────────────────────────────────
kubectl port-forward svc/<name> 8080:80 -n <ns>  # Access locally

# ── Context Switching ─────────────────────────────────────
kubectl config get-contexts               # List all clusters
kubectl config use-context kind-dev       # Switch to kind cluster
```

---

##  Useful Links

| Resource | Link |
|----------|------|
| Copilot Studio | https://copilotstudio.microsoft.com |
| kind Official Docs | https://kind.sigs.k8s.io |
| kind Config Reference | https://kind.sigs.k8s.io/docs/user/configuration |
| kubectl Cheat Sheet | https://kubernetes.io/docs/reference/kubectl/cheatsheet |
| kind Releases | https://github.com/kubernetes-sigs/kind/releases |

---

*KubeKind Assistant — Beginner-friendly AI Agent for local Kubernetes.*
*Built with Microsoft Copilot Studio free tier. No credit card needed.*
