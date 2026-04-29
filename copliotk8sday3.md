# Kubernetes Troubleshooting Agent — Build Guide

Based on the lab guide you shared, here's a step-by-step walkthrough tailored specifically for a Kubernetes agent.

---

## Step 1 — Create the Agent

1. Go to [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Click the **Agent** card under *Start building from scratch*
3. Fill in the fields:

| Field | What to enter |
|-------|--------------|
| **Name** | `Kubernetes Support Bot` |
| **Description** | `Helps engineers troubleshoot Kubernetes clusters, debug workloads, and follow best practices for setup and operations.` |
| **Instructions** | *(see below)* |

**Instructions to paste:**
```
You are a Kubernetes support assistant for platform and DevOps engineers.

You can help with:
- Cluster setup and configuration
- Pod, Deployment, and Service troubleshooting
- Networking, ingress, and DNS issues
- Persistent volumes and storage problems
- RBAC, namespaces, and access control
- Resource limits, autoscaling, and scheduling
- Best practices for production Kubernetes

Always answer using the knowledge provided from kubernetes.io.
If a question is beyond the provided knowledge, say:
"I recommend checking https://kubernetes.io/docs or raising this with your platform team."

Be concise, technical, and precise. Use kubectl commands where relevant.
```

4. Click **Create**

---

## Step 2 — Add Knowledge from kubernetes.io

Go to **Knowledge** → **Add knowledge** → **Public website**, then add these URLs one by one:

| URL | What it covers |
|-----|---------------|
| `https://kubernetes.io/docs/concepts/` | Core concepts |
| `https://kubernetes.io/docs/tasks/debug/` | Debugging and troubleshooting |
| `https://kubernetes.io/docs/setup/` | Cluster setup |
| `https://kubernetes.io/docs/concepts/configuration/` | Config best practices |
| `https://kubernetes.io/docs/concepts/workloads/` | Pods, Deployments, Jobs |

> ⏳ Wait for **all sources** to show **Ready** before testing — takes 1–3 minutes each.

---

## Step 3 — Build a "Debug a Pod" Topic

Go to **Topics** → **Add a topic** → **From blank**

**Name it:** `Debug a Pod`

**Trigger phrases** (add all five):
```
My pod is not starting
Pod is in CrashLoopBackOff
Pod stuck in Pending
Container keeps restarting
Pod is not running
```

**Then add these nodes in order:**

**Node 1 — Send a message:**
```
I can help you debug your pod. Let me gather a few details.
```

**Node 2 — Ask a question:**
```
What is the pod name and namespace?
(Example: my-app-pod in the production namespace)
```
- Identify: **String** — Save as: `PodName`

**Node 3 — Ask a question:**
```
What status is the pod showing? 
(e.g. Pending, CrashLoopBackOff, OOMKilled, ImagePullBackOff, Error)
```
- Identify: **String** — Save as: `PodStatus`

**Node 4 — Ask a question:**
```
Have you already run kubectl describe pod or kubectl logs? 
What did the output show?
```
- Identify: **String** — Save as: `ErrorOutput`

**Node 5 — Send a message (use variable picker `{}` for each variable):**
```
Thanks! Here is what to check for {PodName} showing {PodStatus}:

Run these commands to investigate:
  kubectl describe pod {PodName}
  kubectl logs {PodName} --previous
  kubectl get events --sort-by=.metadata.creationTimestamp

Based on your status ({PodStatus}), common causes are:
  - CrashLoopBackOff → application error or bad startup command, check logs
  - Pending → insufficient resources or no matching node, check describe output
  - ImagePullBackOff → wrong image name or missing registry credentials
  - OOMKilled → container exceeded memory limit, increase resources

Share the output of describe with your platform team if the issue persists.
```

4. Click **Save** → **Test** with `My pod is not starting`

---

## Step 4 — Build a "Raise a Platform Ticket" Topic

**Name it:** `Raise a Platform Ticket`

**Trigger phrases:**
```
I need help from the platform team
Raise a support ticket
Log a Kubernetes issue
Escalate my problem
I cannot fix this myself
```

**Nodes:**

1. *Send a message:* `I will log a ticket with the platform team. Just a few quick questions.`
2. *Ask a question:* `What cluster is affected?` → String → `ClusterName`
3. *Ask a question:* `Briefly describe the issue.` → String → `IssueDescription`
4. *Ask a question:* `What is your team or squad name?` → String → `TeamName`
5. *Send a message:*
```
Ticket logged. Here is your summary:

Cluster: {ClusterName}
Issue: {IssueDescription}
Raised by: {TeamName}

The platform team will respond within 4 business hours.
For urgent P1 issues, contact the on-call engineer directly.
```

---

## Step 5 — Update System Topics

**Greeting topic — replace default text with:**
```
Hello! I am the Kubernetes Support Bot.

I can help you with:
- Pod and Deployment troubleshooting
- Cluster setup and configuration
- Networking, storage, and RBAC
- kubectl commands and best practices
- Production readiness guidance

Type your question or say "raise a ticket" to escalate to the platform team.
```

**Fallback topic — replace default text with:**
```
I'm sorry, I couldn't find an answer to that.

Try one of these:
- Rephrase your question with more detail
- Check https://kubernetes.io/docs directly
- Say "raise a ticket" to contact the platform team

Is there anything else I can help with?
```

---

## Step 6 — Publish and Connect

1. Click **Publish** (top right) → confirm
2. Go to **Channels** → **Microsoft Teams** → **Add to Teams**
3. Click **Open in Teams** to test live

---

## Quick Test Checklist

Run these in the test panel before sharing with your team:

```
[ ] Type "Hello"                     → greeting appears
[ ] Ask "What is a DaemonSet?"       → knowledge answer from kubernetes.io
[ ] Type "My pod is not starting"    → Debug a Pod topic triggers
[ ] Type "raise a ticket"            → Raise a Platform Ticket topic triggers
[ ] Ask "What is the best pizza?"    → fallback message appears
[ ] All kubectl commands display correctly in messages
```

---

The main thing to watch after go-live is **Unrecognised Phrases** under Analytics — that's where you'll see what engineers are actually asking that the bot can't handle yet, and you can keep adding topics or knowledge sources from there.
