# Mastering Prompt Engineering for DevOps

Prompt engineering is the art of crafting instructions to get the most accurate and helpful responses from AI models. In this guide, you'll learn how to master prompt engineering using the following techniques, with DevOps-related examples.

---

## 1. Zero-Shot Prompting

In zero-shot prompting, the AI is asked to generate a response without any prior examples or context. It relies entirely on the AI's general understanding of the task.

### Example 1: Generate a Shell Script to Backup Logs

**Prompt:**
```
Write a shell script to back up log files from /var/logs to /backup.
```

**Response:**
```bash
#!/bin/bash
# Backup logs from /var/logs to /backup
mkdir -p /backup
cp -r /var/logs/* /backup/
echo "Backup completed!"
```

---

### Example 2: Explain the Purpose of a `kubectl get pods` Command

**Prompt:**
```
Explain the purpose of the kubectl get pods command.
```

**Response:**

The `kubectl get pods` command lists all the running and pending pods in the current Kubernetes namespace. It provides information about the pod name, status, restart count, and age.

---

## 2. Few-Shot Prompting

In few-shot prompting, you provide the AI with a few examples to guide its response.

### Example 1: Write a Shell Script to Create and Archive Logs

**Prompt:**
```
Here are some shell scripts to manage logs:

Example 1:
# Create a log file
touch /var/logs/app.log
echo "Application started" > /var/logs/app.log

Example 2:
# Archive old logs
tar -czf /backup/app_logs.tar.gz /var/logs/*

Now write a script to delete logs older than 7 days.
```

**Response:**
```bash
#!/bin/bash
# Delete logs older than 7 days
find /var/logs -type f -mtime +7 -exec rm {} \;
echo "Old logs deleted!"
```

---

## 3. Multi-Shot Prompting

Multi-shot prompting is an extension of few-shot prompting where you provide multiple examples and progressively refine the task.

### Example 1: Create a Kubernetes Deployment and Describe It

**Prompt:**
```
Example 1:
kubectl create deployment nginx --image=nginx

Example 2:
kubectl get deployment nginx

Example 3:
kubectl describe deployment nginx

Now write a command to scale the nginx deployment to 5 replicas.
```

**Response:**
```bash
kubectl scale deployment nginx --replicas=5
```

---

## 4. Chain of Thought (CoT) Prompting

CoT prompting encourages the AI to "think step-by-step" to improve the accuracy of complex responses.

### Example 1: Debug a Failing Shell Script Step-by-Step

**Prompt:**
```
My shell script is failing. Here is the script:

#!/bin/bash
mkdir -p /backup
cp /var/logs/* /backup/

It shows a "permission denied" error. How can I fix it? Please explain step-by-step.
```

**Response:**

**Step 1** — Check the permissions of `/var/logs`:
```bash
ls -ld /var/logs
```

**Step 2** — If permissions are restricted, use `sudo`:
```bash
sudo cp /var/logs/* /backup/
```

**Step 3** — If SELinux is enabled, adjust the security context:
```bash
sudo chcon -R -t user_home_t /backup
```

---

### Example 2: Troubleshoot a Kubernetes Pod in CrashLoopBackOff

**Prompt:**
```
My Kubernetes pod is in CrashLoopBackOff. Help me debug step-by-step.
```

**Response:**

**Step 1** — Check pod status:
```bash
kubectl get pod <pod-name>
```

**Step 2** — Get detailed pod logs:
```bash
kubectl logs <pod-name>
```

**Step 3** — Describe the pod to check for readiness/liveness probe issues:
```bash
kubectl describe pod <pod-name>
```

**Step 4** — If the container is failing to start, fix the image or entry point issue:
```bash
kubectl set image deployment <deployment-name> <container-name>=<new-image>
```

---

##  Best Practices for Prompt Engineering

| Practice | Description |
|---|---|
| **Be clear and specific** | The more specific the prompt, the better the output. |
| **Use context** | Provide background information or examples when needed. |
| **Iterate and refine** | If the output isn't ideal, adjust the prompt. |
| **Use CoT for complex tasks** | Step-by-step reasoning improves accuracy. |
