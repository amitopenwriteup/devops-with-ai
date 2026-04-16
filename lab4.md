# GitHub Copilot for Shell Scripting
## Hands-On Lab Workshop

> **Workshop Level:** Beginner to Intermediate
> **Duration:** 2–3 Hours
> **Prerequisites:** VS Code installed, GitHub account, basic Linux knowledge

---

## Workshop Objectives

By the end of this workshop, you will be able to:

- Set up GitHub Copilot in VS Code
- Use Copilot to generate shell scripts from comments
- Use Copilot Chat to debug and explain shell scripts
- Complete 5 progressive hands-on labs using Copilot assistance

---

## 🛠️ Environment Setup

### Step 1: Install VS Code

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y code

# Or download from https://code.visualstudio.com
```

### Step 2: Install GitHub Copilot Extension

1. Open VS Code
2. Go to **Extensions** (`Ctrl+Shift+X`)
3. Search for **GitHub Copilot**
4. Click **Install**
5. Sign in with your GitHub account when prompted

### Step 3: Verify Copilot is Active

- Look for the **Copilot icon** in the bottom status bar of VS Code
- It should show ✅ active (not grey/disabled)

### Step 4: Install GitHub Copilot Chat (Optional but Recommended)

1. Search **GitHub Copilot Chat** in Extensions
2. Click **Install**
3. Access it via the chat icon in the left sidebar

---

## 📖 How GitHub Copilot Works in Shell Scripting

| Copilot Feature | How to Trigger | Use Case |
|---|---|---|
| **Inline suggestion** | Start typing or write a comment | Auto-complete next lines |
| **Accept suggestion** | Press `Tab` | Accept the full suggestion |
| **Next suggestion** | `Alt + ]` | Cycle through options |
| **Reject suggestion** | Press `Esc` | Dismiss and type manually |
| **Copilot Chat** | `Ctrl+Shift+I` | Ask questions, debug, explain |
| **Inline Chat** | `Ctrl+I` | Fix or explain selected code |

> 💡 **Pro Tip:** The more descriptive your comments, the better Copilot's suggestions will be.

---

## 🧪 Lab 1: Your First Copilot-Generated Script

### Objective
Use GitHub Copilot to generate a system information script purely from comments.

### Instructions

**Step 1** — Create a new file in VS Code:
```bash
touch lab1_sysinfo.sh
```

**Step 2** — Open the file and type the following comments **one line at a time**, pressing `Enter` after each and accepting Copilot suggestions with `Tab`:

```bash
#!/bin/bash
# Script to display system information

# Print the current hostname

# Print the operating system and kernel version

# Print CPU details

# Print total and used memory in MB

# Print disk usage for root partition

# Print current logged-in users

# Print system uptime
```

**Step 3** — Accept Copilot's suggestions for each section.

### Expected Output (Copilot should generate something like):
```bash
#!/bin/bash
# Script to display system information

# Print the current hostname
echo "Hostname: $(hostname)"

# Print the operating system and kernel version
echo "OS: $(uname -o)"
echo "Kernel: $(uname -r)"

# Print CPU details
echo "CPU: $(grep 'model name' /proc/cpuinfo | head -1 | cut -d: -f2 | xargs)"

# Print total and used memory in MB
TOTAL=$(free -m | awk '/Mem:/ {print $2}')
USED=$(free -m | awk '/Mem:/ {print $3}')
echo "Memory: ${USED}MB used / ${TOTAL}MB total"

# Print disk usage for root partition
echo "Disk Usage: $(df -h / | awk 'NR==2 {print $3 "/" $2 " (" $5 " used)"}')"

# Print current logged-in users
echo "Logged-in Users: $(who | awk '{print $1}' | sort -u | tr '\n' ' ')"

# Print system uptime
echo "Uptime: $(uptime -p)"
```

### ✅ Lab 1 Checkpoint
- [ ] Copilot suggested code for each comment
- [ ] Script runs without errors: `bash lab1_sysinfo.sh`
- [ ] Output displays all system information correctly

---

## 🧪 Lab 2: Automated Backup Script

### Objective
Use Copilot to build a full backup automation script with error handling.

### Instructions

**Step 1** — Create the file:
```bash
touch lab2_backup.sh
```

**Step 2** — Type these comments and let Copilot fill in the code:

```bash
#!/bin/bash
# Automated backup script with error handling and logging

# Define source directory, backup directory, and log file

# Create backup directory if it doesn't exist

# Function to log messages with timestamp

# Check if source directory exists, exit if not

# Create a timestamped compressed archive of the source directory

# Check if backup was successful and log the result

# Delete backups older than 7 days

# Print completion message
```

**Step 3** — After Copilot generates the script, use **Copilot Chat** to ask:

```
/explain  What does the find command in this script do?
```

**Step 4** — Make the script executable and test it:
```bash
chmod +x lab2_backup.sh
mkdir -p /tmp/test_source
echo "test file" > /tmp/test_source/test.txt
bash lab2_backup.sh
```

### ✅ Lab 2 Checkpoint
- [ ] Script creates a `.tar.gz` backup file
- [ ] Log file is created with timestamps
- [ ] Old backup deletion logic is present
- [ ] You used Copilot Chat to explain part of the script

---

## 🧪 Lab 3: Debugging with Copilot Chat

### Objective
Use Copilot Chat to identify and fix bugs in broken shell scripts.

### Instructions

**Step 1** — Create a file with intentional bugs:
```bash
touch lab3_buggy.sh
```

**Step 2** — Paste this buggy script:

```bash
#!/bin/bash
# Buggy script - find and fix all errors using Copilot Chat

THRESHOLD=80
LOGFILE=/var/log/disk_monitor.log

check_disk() {
  USAGE=df -h / | awk 'NR==2 {print $5}' | cut -d'%' -f1
  if [ $USAGE -gt $THRESHOLD ]
  then
    echo "$(date) WARNING: Disk usage is ${USAGE}%" >> $LOGFILE
    echo "Alert sent!"
  fi
}

check_disk
echoo "Disk check complete"
```

**Step 3** — Select all the code, press `Ctrl+I` (Inline Chat) and type:

```
Fix all the bugs in this shell script
```

**Step 4** — Review Copilot's fixes. It should identify:

| Bug | Fix |
|---|---|
| Missing `$()` around `df` command | `USAGE=$(df -h / ...)` |
| Missing `then` on same line or correct syntax | Fix `if` statement |
| `echoo` typo | Replace with `echo` |
| Unquoted variables | Add quotes around `"$USAGE"` |

**Step 5** — After fixing, open **Copilot Chat** and ask:

```
What best practices should I follow to avoid these bugs in future shell scripts?
```

### ✅ Lab 3 Checkpoint
- [ ] All bugs identified using Copilot Chat
- [ ] Fixed script runs without errors
- [ ] Copilot explained best practices for clean scripting

---

## 🧪 Lab 4: Kubernetes Pod Monitor Script

### Objective
Use Copilot to write a script that monitors Kubernetes pods and alerts on failures.

### Instructions

**Step 1** — Create the file:
```bash
touch lab4_k8s_monitor.sh
```

**Step 2** — Type the following comments and accept Copilot suggestions:

```bash
#!/bin/bash
# Kubernetes Pod Health Monitor
# Checks for pods in CrashLoopBackOff or Error state and logs alerts

# Define namespace variable (default to "default" if not set)

# Define log file path

# Function to log messages with timestamp

# Function to check pod status in the namespace

# Get all pods and filter those in CrashLoopBackOff or Error state

# For each unhealthy pod, log the alert and print pod logs for debugging

# Function to check if kubectl is installed

# Main execution: check kubectl, then monitor pods

# Run the monitor every 60 seconds in a loop
```

**Step 3** — Use Copilot Chat to enhance the script:

```
Add a feature to send a Slack webhook notification when a pod is in CrashLoopBackOff
```

**Step 4** — Copilot should add something like:

```bash
# Send Slack alert
send_slack_alert() {
  local POD_NAME=$1
  local NAMESPACE=$2
  local WEBHOOK_URL="${SLACK_WEBHOOK_URL:-}"

  if [ -n "$WEBHOOK_URL" ]; then
    curl -s -X POST "$WEBHOOK_URL" \
      -H 'Content-type: application/json' \
      --data "{\"text\":\"🚨 Alert: Pod *${POD_NAME}* in namespace *${NAMESPACE}* is in CrashLoopBackOff!\"}"
  fi
}
```

### ✅ Lab 4 Checkpoint
- [ ] Script detects unhealthy pod states
- [ ] Logging is implemented with timestamps
- [ ] Slack webhook notification added via Copilot Chat
- [ ] Loop monitoring runs every 60 seconds

---

## 🧪 Lab 5: CI/CD Pipeline Helper Script

### Objective
Use Copilot to build a deployment validation script used in CI/CD pipelines.

### Instructions

**Step 1** — Create the file:
```bash
touch lab5_deploy_validate.sh
```

**Step 2** — Type these comments and accept Copilot suggestions:

```bash
#!/bin/bash
# CI/CD Deployment Validation Script
# Validates environment before and after deployment

# Accept environment argument (dev/staging/prod), exit if not provided

# Define expected services array (nginx, docker, kubectl)

# Function to check if a required service is running

# Function to validate all required services

# Function to check if a URL is reachable (health check)

# Function to validate environment variables are set

# Function to print a summary report of all checks

# Run pre-deployment checks

# Run post-deployment health check on the application URL

# Exit with appropriate code based on validation results
```

**Step 3** — Ask Copilot Chat to add a feature:

```
Add a rollback function that triggers if post-deployment health check fails
```

**Step 4** — Test the script:
```bash
chmod +x lab5_deploy_validate.sh
bash lab5_deploy_validate.sh dev
```

### ✅ Lab 5 Checkpoint
- [ ] Script accepts environment argument
- [ ] All service checks work correctly
- [ ] Health check URL validation is present
- [ ] Rollback function added via Copilot Chat
- [ ] Exit codes are correctly set (0 = success, 1 = failure)

---

## 💬 Copilot Chat — Useful Prompts for Shell Scripting

```
# Explain what a script does
/explain What does this script do line by line?

# Fix errors
Fix the bug in this script

# Optimize
Optimize this script for better performance

# Add error handling
Add error handling and logging to this script

# Security check
Are there any security vulnerabilities in this script?

# Add comments
Add clear comments to explain each section of this script

# Convert
Convert this bash script to be compatible with both bash and zsh

# Unit test
Write a test script to validate the functions in this script
```

---

## 🏆 Best Practices: Using Copilot for Shell Scripting

| Practice | Description |
|---|---|
| **Write descriptive comments** | Copilot generates better code with clear intent comments |
| **Review every suggestion** | Never blindly accept — always read and understand the code |
| **Use Copilot Chat for explanation** | Ask Copilot to explain unfamiliar commands before using them |
| **Iterate with Copilot** | Refine suggestions by providing follow-up prompts |
| **Test all generated scripts** | Run in a safe environment before using in production |
| **Use `set -euo pipefail`** | Ask Copilot to add this to every script for safety |
| **Ask for security review** | Prompt Copilot to check for vulnerabilities |

---

## 📚 Summary

| Lab | Topic | Key Copilot Feature Used |
|---|---|---|
| Lab 1 | System Info Script | Inline suggestion from comments |
| Lab 2 | Backup Automation | Full script generation + Chat explain |
| Lab 3 | Debugging | Inline Chat fix + Chat best practices |
| Lab 4 | Kubernetes Monitor | Chat feature enhancement |
| Lab 5 | CI/CD Validation | Chat rollback generation |

---

> 🚀 **Next Steps:** Explore GitHub Copilot for Python, Terraform, and Kubernetes YAML automation in upcoming workshops.
