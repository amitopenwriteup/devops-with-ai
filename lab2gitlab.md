# GitLab CI/CD Workshop: From Zero to Secure Container

> **Who is this for?** Complete beginners to GitLab CI/CD. No prior experience needed.
> **What you'll build:** A Java 11 app → Dockerized → Published to Docker Hub → Scanned with Trivy.

---

## Table of Contents

1. [Prompting Concepts: Zero-shot, Few-shot, Chain-of-Thought](#1-prompting-concepts)
2. [GitLab CI Fundamentals](#2-gitlab-ci-fundamentals)
3. [Setting Up Your GitLab Runner](#3-setting-up-your-gitlab-runner)
4. [Importing the Java 11 Project](#4-importing-the-java-11-project)
5. [Writing the Dockerfile](#5-writing-the-dockerfile)
6. [Building & Publishing to Docker Hub](#6-building--publishing-to-docker-hub)
7. [Security Scanning with Trivy](#7-security-scanning-with-trivy)
8. [Full `.gitlab-ci.yml` — All Together](#8-full-gitlab-ciyml--all-together)
9. [Hands-On Exercises](#9-hands-on-exercises)

---

## 1. Prompting Concepts

> **Why does this matter?** When you write a `.gitlab-ci.yml` file, you're giving instructions to a machine — just like prompting an AI. Clear instructions = predictable results.

### Zero-Shot Prompting

**What it is:** You give a command with **no examples**. The system figures it out from context alone.

**In CI terms:** Writing a pipeline stage with no template — you trust the tool to understand your intent.

```yaml
# Zero-shot: "just run the tests" — no example given
test:
 script:
 - mvn test
```

You didn't tell GitLab *how* Maven works. It just does it because Maven is installed. That's zero-shot thinking.

---

### Few-Shot (Multi-Shot) Prompting

**What it is:** You provide **2–3 examples** so the system learns the pattern before doing the real task.

**In CI terms:** Defining multiple similar jobs so GitLab learns your pattern.

```yaml
# Few-shot: "here are examples of how I want stages to look"
build-dev:
 stage: build
 script:
 - mvn package -Pdev

build-staging:
 stage: build
 script:
 - mvn package -Pstaging

# Now GitLab (and your team) understands the pattern for:
build-prod:
 stage: build
 script:
 - mvn package -Pprod
```

Each example reinforces the expected structure.

---

### Chain-of-Thought (CoT)

**What it is:** Breaking a complex task into **step-by-step reasoning**. Instead of jumping to the answer, you walk through each step.

**In CI terms:** Your pipeline stages are a chain of thought — each step depends on the previous one succeeding.

```
Think step by step:
 1. Does the code compile? → compile stage
 2. Do the tests pass? → test stage
 3. Is the image built? → build stage
 4. Is it safe to ship? → trivy scan stage
 5. Push only if all above pass → publish stage
```

This is exactly how a good pipeline is structured — **chain of thought made real**.

---

## 2. GitLab CI Fundamentals

### Key Vocabulary

| Term | Meaning | Real-world Analogy |
|------|---------|-------------------|
| **Pipeline** | The full automated workflow | An assembly line |
| **Stage** | A group of jobs that run together | One station on the line |
| **Job** | A single unit of work | One task a worker does |
| **Runner** | The machine that executes jobs | The worker |
| **Artifact** | Files saved from a job | Output from the station |
| **Variable** | A stored value (like a secret) | A sticky note with info |

### Pipeline Flow

```
[Push Code] → [compile] → [test] → [build image] → [scan] → [publish]
 ↓ fail? ↓ fail? ↓ fail? ↓ fail?
 Pipeline stops here (nothing continues downstream)
```

---

## 3. Setting Up Your GitLab Runner

> A **Runner** is the machine that actually runs your CI jobs. Without it, nothing executes.

### Option A: Use GitLab's Shared Runners (Easiest for beginners)

GitLab.com provides free shared runners. Just enable them:

1. Go to your project → **Settings** → **CI/CD**
2. Expand **Runners**
3. Under "Shared runners", click **Enable shared runners for this project**

 Done! No setup needed.

---

### Option B: Register Your Own Runner (Recommended for production)

**Step 1 — Install GitLab Runner on your machine:**

```bash
# Linux (Debian/Ubuntu)
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# macOS
brew install gitlab-runner
brew services start gitlab-runner
```

**Step 2 — Get your registration token:**

Go to **Project → Settings → CI/CD → Runners → Registration token** and copy it.

**Step 3 — Register the runner:**

```bash
sudo gitlab-runner register
```

You'll be prompted:

```
Enter the GitLab instance URL:
> https://gitlab.com

Enter the registration token:
> YOUR_TOKEN_HERE

Enter a description for the runner:
> my-workshop-runner

Enter tags for the runner (comma-separated):
> docker, java

Enter the executor:
> docker

Enter the default Docker image:
> maven:3.8-openjdk-11
```

**Step 4 — Verify it's online:**

```bash
sudo gitlab-runner status
# → gitlab-runner: Service is running!
```

In GitLab UI → Settings → CI/CD → Runners → you should see a **green dot** next to your runner.

### Runner Tags — Why They Matter

Tags let you route specific jobs to specific runners:

```yaml
# This job will ONLY run on runners tagged "docker"
build:
 tags:
 - docker
 script:
 - docker build .
```

---

## 4. Importing the Java 11 Project

> We'll use the `biezhi/java11` project structure you saw — a Maven-based Java 11 demo.

### Option A: Fork from GitHub into GitLab

1. In GitLab, click **New Project** → **Import Project** → **GitHub**
2. Authenticate with GitHub
3. Search for `biezhi/java11` and import

### Option B: Manual Import

```bash
# Clone the GitHub repo
git clone https://github.com/biezhi/java11.git
cd java11

# Add GitLab as a new remote
git remote add gitlab https://gitlab.com/YOUR_USERNAME/java11.git

# Push to GitLab
git push gitlab main
```

### Project Structure You Should See

```
java11/
 src/
 main/java/io/github/biezhi/java11/ ← Java source files
 .gitignore
 LICENSE
 README.md
 pom.xml ← Maven build config
```

---

## 5. Writing the Dockerfile

> A **Dockerfile** defines how to package your app into a container image.

### Chain-of-Thought Approach to Writing a Dockerfile

**Step 1 — What base do I need?**
→ Java 11 app built with Maven → need JDK 11 to build, JRE 11 to run.

**Step 2 — How do I build it?**
→ `mvn package` produces a JAR file in `target/`

**Step 3 — What do I actually ship?**
→ Only the JAR. We don't need Maven or source code in production.

**Step 4 — How do I run it?**
→ `java -jar app.jar`

### The Dockerfile (Multi-stage — Best Practice)

Create a file named `Dockerfile` in your project root:

```dockerfile
# ============================================================
# STAGE 1: BUILD
# Few-shot hint: This is the "builder" stage — compile only
# ============================================================
FROM maven:3.8-openjdk-11 AS builder

# Set working directory inside the container
WORKDIR /app

# Copy dependency list first (caching optimization)
# Zero-shot: Maven knows what to do with pom.xml
COPY pom.xml .
RUN mvn dependency:go-offline -B

# Copy source code and build
COPY src ./src
RUN mvn package -DskipTests -B

# ============================================================
# STAGE 2: RUNTIME
# Chain-of-thought: We proved it builds → now ship lean image
# ============================================================
FROM eclipse-temurin:11-jre-alpine

# Create a non-root user (security best practice)
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy ONLY the built JAR from the builder stage
COPY --from=builder /app/target/*.jar app.jar

# Switch to non-root user
USER appuser

# Expose port (adjust if your app uses a different port)
EXPOSE 8080

# Start the application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Why Multi-stage?

| Single-stage Image | Multi-stage Image |
|-------------------|------------------|
| ~650 MB (includes Maven, JDK, source) | ~85 MB (only JRE + JAR) |
| Attack surface: large | Attack surface: minimal |
| Build tools exposed in prod | Build tools gone |

---

## 6. Building & Publishing to Docker Hub

### Prerequisites

1. Create a free account at [hub.docker.com](https://hub.docker.com)
2. Create a repository (e.g., `yourusername/java11-demo`)

### Store Secrets in GitLab (Never in code!)

Go to **Project → Settings → CI/CD → Variables** and add:

| Variable Name | Value | Protected | Masked |
|--------------|-------|-----------|--------|
| `DOCKER_USERNAME` | your Docker Hub username | | |
| `DOCKER_PASSWORD` | your Docker Hub password/token | | |

> **Tip:** Use a Docker Hub **Access Token** instead of your password. Go to Docker Hub → Account Settings → Security → New Access Token.

### CI Job: Build and Push

```yaml
build-and-push:
 stage: build
 image: docker:24
 services:
 - docker:24-dind # Docker-in-Docker: allows running docker commands
 variables:
 DOCKER_TLS_CERTDIR: "/certs"
 IMAGE_TAG: $DOCKER_USERNAME/java11-demo:$CI_COMMIT_SHORT_SHA
 before_script:
 # Chain-of-thought step 1: authenticate before pushing
 - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
 script:
 # Chain-of-thought step 2: build the image
 - docker build -t $IMAGE_TAG .
 # Chain-of-thought step 3: also tag as latest
 - docker tag $IMAGE_TAG $DOCKER_USERNAME/java11-demo:latest
 # Chain-of-thought step 4: push both tags
 - docker push $IMAGE_TAG
 - docker push $DOCKER_USERNAME/java11-demo:latest
 tags:
 - docker
```

### Understanding `$CI_COMMIT_SHORT_SHA`

GitLab provides built-in variables automatically:

| Variable | Example Value | Use |
|----------|--------------|-----|
| `$CI_COMMIT_SHORT_SHA` | `a1b2c3d4` | Unique tag per commit |
| `$CI_COMMIT_REF_NAME` | `main` | Branch name |
| `$CI_PROJECT_NAME` | `java11` | Repo name |
| `$CI_PIPELINE_ID` | `12345` | Pipeline number |

---

## 7. Security Scanning with Trivy

> **Trivy** by Aqua Security scans your container image for known vulnerabilities (CVEs) before it reaches production.

### What Trivy Checks

- OS packages (Alpine, Ubuntu, Debian)
- Application dependencies (Maven, npm, pip)
- Misconfigurations in Dockerfiles
- Secrets accidentally baked into images

### CI Job: Trivy Scan

```yaml
trivy-scan:
 stage: scan
 image:
 name: aquasec/trivy:latest
 entrypoint: [""] # Override entrypoint so we can run custom commands
 variables:
 IMAGE_TAG: $DOCKER_USERNAME/java11-demo:$CI_COMMIT_SHORT_SHA
 # Fail pipeline if CRITICAL or HIGH vulnerabilities found
 TRIVY_EXIT_CODE: "1"
 TRIVY_SEVERITY: "CRITICAL,HIGH"
 script:
 # Chain-of-thought: Pull the image we just built, then scan it
 - trivy image
 --exit-code $TRIVY_EXIT_CODE
 --severity $TRIVY_SEVERITY
 --no-progress
 --format table
 $IMAGE_TAG
 allow_failure: false # This stage MUST pass before publish
```

### Trivy Output Example

```
java11-demo:a1b2c3d4 (alpine 3.18.4)

Total: 2 (HIGH: 2, CRITICAL: 0)

 Library Vulnerability Severity Installed Version Fixed Version Title 

 libssl3 CVE-2023-xxxx HIGH 3.1.1-r1 3.1.2-r0 OpenSSL: buffer overflow 
 libcrypto3 CVE-2023-xxxx HIGH 3.1.1-r1 3.1.2-r0 OpenSSL: information disclosure 

```

If vulnerabilities are found → **pipeline fails** → image is NOT published. 

### Save Trivy Report as Artifact

```yaml
trivy-scan:
 # ... (same as above, add these lines)
 artifacts:
 when: always # Save report even if job fails
 reports:
 junit: trivy-report.xml # Shows in GitLab's test results UI
 paths:
 - trivy-report.html # Also save HTML for humans
 expire_in: 7 days
 script:
 # Generate both table output and HTML report
 - trivy image --format template --template "@contrib/html.tpl"
 -o trivy-report.html $IMAGE_TAG
 - trivy image --format junit -o trivy-report.xml $IMAGE_TAG
 - trivy image --exit-code 1 --severity CRITICAL,HIGH $IMAGE_TAG
```

---

## 8. Full `.gitlab-ci.yml` — All Together

Create `.gitlab-ci.yml` in your project root:

```yaml
# ============================================================
# GitLab CI/CD Pipeline — Java 11 Workshop
# Chain-of-thought: compile → test → build → scan → publish
# ============================================================

# Define the order of stages (chain of thought)
stages:
 - compile # Step 1: Does it build?
 - test # Step 2: Do tests pass?
 - build # Step 3: Package into container
 - scan # Step 4: Is it safe?
 - publish # Step 5: Ship it!

# 
# Global variables (reused across all jobs)
# 
variables:
 MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
 IMAGE_NAME: "$DOCKER_USERNAME/java11-demo"
 IMAGE_TAG: "$IMAGE_NAME:$CI_COMMIT_SHORT_SHA"

# 
# Cache Maven dependencies between jobs
# 
cache:
 paths:
 - .m2/repository/

# 
# STAGE: compile
# Zero-shot: Maven knows how to compile
# 
compile:
 stage: compile
 image: maven:3.8-openjdk-11
 script:
 - mvn compile -B
 tags:
 - docker

# 
# STAGE: test
# Few-shot: same pattern as compile, different goal
# 
test:
 stage: test
 image: maven:3.8-openjdk-11
 script:
 - mvn test -B
 artifacts:
 when: always
 reports:
 junit:
 - target/surefire-reports/*.xml # GitLab shows test results natively
 expire_in: 1 week
 tags:
 - docker

# 
# STAGE: build
# Build Docker image and push to registry
# 
build-image:
 stage: build
 image: docker:24
 services:
 - docker:24-dind
 variables:
 DOCKER_TLS_CERTDIR: "/certs"
 before_script:
 - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
 script:
 - docker build -t $IMAGE_TAG .
 - docker tag $IMAGE_TAG $IMAGE_NAME:latest
 - docker push $IMAGE_TAG
 - docker push $IMAGE_NAME:latest
 tags:
 - docker
 only:
 - main # Only build images from the main branch

# 
# STAGE: scan
# Chain-of-thought: prove it's safe BEFORE publishing
# 
trivy-scan:
 stage: scan
 image:
 name: aquasec/trivy:latest
 entrypoint: [""]
 before_script:
 - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
 script:
 # Generate HTML report for humans
 - trivy image
 --format template
 --template "@contrib/html.tpl"
 -o trivy-report.html
 $IMAGE_TAG
 # Fail pipeline on CRITICAL or HIGH findings
 - trivy image
 --exit-code 1
 --severity CRITICAL,HIGH
 --no-progress
 $IMAGE_TAG
 artifacts:
 when: always
 paths:
 - trivy-report.html
 expire_in: 7 days
 tags:
 - docker
 only:
 - main

# 
# STAGE: publish
# Only runs if ALL previous stages passed
# 
publish:
 stage: publish
 image: docker:24
 services:
 - docker:24-dind
 variables:
 DOCKER_TLS_CERTDIR: "/certs"
 before_script:
 - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
 script:
 # Re-tag with version from git tag (if available) or use SHA
 - |
 if [ -n "$CI_COMMIT_TAG" ]; then
 docker pull $IMAGE_TAG
 docker tag $IMAGE_TAG $IMAGE_NAME:$CI_COMMIT_TAG
 docker push $IMAGE_NAME:$CI_COMMIT_TAG
 echo "Published as $IMAGE_NAME:$CI_COMMIT_TAG"
 else
 echo "No git tag — image already published as :latest and :$CI_COMMIT_SHORT_SHA"
 fi
 tags:
 - docker
 only:
 - main
 - tags
```

---

## 9. Hands-On Exercises

### Exercise 1 — Zero-Shot (Beginner)

Add a `package` job between `test` and `build-image` that runs `mvn package -DskipTests`.

**Hint:** Copy the `compile` job and change the stage and script.

<details>
<summary> Show Answer</summary>

```yaml
package:
 stage: test # runs in parallel with test stage
 image: maven:3.8-openjdk-11
 script:
 - mvn package -DskipTests -B
 artifacts:
 paths:
 - target/*.jar
 expire_in: 1 hour
 tags:
 - docker
```

</details>

---

### Exercise 2 — Few-Shot (Intermediate)

Add a second scan job that scans for **MEDIUM** vulnerabilities but does **not** fail the pipeline (advisory only).

**Hint:** Use `allow_failure: true` and `TRIVY_SEVERITY: "MEDIUM"`.

<details>
<summary> Show Answer</summary>

```yaml
trivy-scan-advisory:
 stage: scan
 image:
 name: aquasec/trivy:latest
 entrypoint: [""]
 script:
 - trivy image
 --exit-code 0 # Never fail — advisory only
 --severity MEDIUM
 --no-progress
 $IMAGE_TAG
 allow_failure: true
 tags:
 - docker
 only:
 - main
```

</details>

---

### Exercise 3 — Chain-of-Thought (Advanced)

Think through and add a **deploy** stage that runs after `publish` and sends a Slack notification using `curl`. 

**Chain-of-thought questions to answer before coding:**
1. What stage does it go in?
2. What image do I need (`curl` is in `alpine`)?
3. Where do I store the Slack webhook URL?
4. What should the message say?
5. Should it run on all branches or only `main`?

<details>
<summary> Show Answer</summary>

First add `deploy` to your `stages:` list, then:

```yaml
notify-slack:
 stage: deploy
 image: alpine:latest
 before_script:
 - apk add --no-cache curl
 script:
 - |
 curl -X POST -H 'Content-type: application/json' \
 --data "{
 \"text\": \" *$CI_PROJECT_NAME* deployed!\n• Commit: \`$CI_COMMIT_SHORT_SHA\` by $GITLAB_USER_LOGIN\n• Image: \`$IMAGE_TAG\`\n• Pipeline: $CI_PIPELINE_URL\"
 }" \
 $SLACK_WEBHOOK_URL
 tags:
 - docker
 only:
 - main
```

Add `SLACK_WEBHOOK_URL` as a masked GitLab CI variable.

</details>

---

## Workshop Checklist

- [ ] GitLab Runner registered and showing green online
- [ ] Java 11 project imported into GitLab
- [ ] `Dockerfile` created with multi-stage build
- [ ] `DOCKER_USERNAME` and `DOCKER_PASSWORD` set as CI variables
- [ ] `.gitlab-ci.yml` committed and pipeline runs
- [ ] Docker Hub shows your image with commit SHA tag
- [ ] Trivy scan report downloadable from pipeline artifacts
- [ ] All 5 stages shown green in GitLab pipeline view

---

## Quick Reference Card

```
Pipeline Trigger: git push
Stages order: compile → test → build → scan → publish
Image naming: username/repo:SHORT_SHA
Secrets location: Settings → CI/CD → Variables
Artifacts: Download from pipeline → job → Browse
Runner tags: match job tags: [docker] to runner tags
Trivy severity: CRITICAL,HIGH = fail | MEDIUM = warn only
```

---

*Workshop created for GitLab CI/CD beginners — covering zero-shot, few-shot, and chain-of-thought prompting patterns as applied to pipeline design.*
