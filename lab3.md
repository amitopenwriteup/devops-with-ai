# GitLab CI/CD Workshop: Copilot-Guided Participant Guide

**Format:** You drive. Copilot assists. The facilitator does not give you the answer.
**Goal:** By the end, you will have a working pipeline that compiles, tests, builds, scans, and publishes a Java 11 container — written by you, with Copilot as your pair programmer.

---

## How This Guide Works

Each section gives you:

- **What you need to understand** — the concept in plain language
- **What to build** — the task
- **How to prompt Copilot** — example prompts you can use or adapt
- **How to know it worked** — the verification check

There are no answers in this document. If you are stuck, ask your facilitator for a nudge, not a solution.

---

## Before You Start

Make sure you have:

- A GitLab account (gitlab.com)
- A Docker Hub account (hub.docker.com)
- GitHub Copilot active in your editor (VS Code recommended)
- Git installed and configured locally

---

## Table of Contents

1. [Understanding the Three Prompting Styles](#1-understanding-the-three-prompting-styles)
2. [GitLab CI Vocabulary](#2-gitlab-ci-vocabulary)
3. [Set Up Your GitLab Runner](#3-set-up-your-gitlab-runner)
4. [Import the Java 11 Project](#4-import-the-java-11-project)
5. [Write Your Dockerfile](#5-write-your-dockerfile)
6. [Configure Docker Hub Publishing](#6-configure-docker-hub-publishing)
7. [Add Trivy Security Scanning](#7-add-trivy-security-scanning)
8. [Build Your Full Pipeline](#8-build-your-full-pipeline)
9. [Final Checklist](#9-final-checklist)

---

## 1. Understanding the Three Prompting Styles

This section is reading only. No code yet.

When you talk to Copilot (or any AI), the way you phrase your request changes the quality of the output. There are three patterns you will practice today.

### Zero-Shot

You give one instruction with no examples. You trust that Copilot understands the context.

When to use it: simple, well-known tasks where the intent is unambiguous.

Example of a zero-shot prompt to Copilot:
> "Write a GitLab CI job that runs Maven tests."

Copilot infers everything else — which image, which command, which stage.

### Few-Shot (Multi-Shot)

You give Copilot two or three examples of the pattern you want, then ask it to continue.

When to use it: when you want Copilot to match a specific structure or style you have already established.

Example: You write two jobs following the same pattern, then add a comment like:
> "// follow the same pattern for the production build"

Copilot reads your examples and continues in the same shape.

### Chain-of-Thought

You break the problem into steps before asking for code. You reason out loud, then ask Copilot to implement each step.

When to use it: complex tasks where jumping straight to code would skip important logic.

Example:
> "I need a pipeline that: first checks if the code compiles, then runs tests, then builds a Docker image only if tests pass, then scans for vulnerabilities, then publishes only if the scan is clean. Start with the stages definition."

You are walking Copilot through your reasoning before asking it to write anything.

**Reflection question:** Look at the pipeline you will build today. Which of your five stages maps to which prompting style? Write your answer in the space below before moving on.

```
compile  →
test     →
build    →
scan     →
publish  →
```

---

## 2. GitLab CI Vocabulary

Read this section. You will need these terms to write accurate Copilot prompts. Vague language produces vague code.

| Term | What it means |
|------|---------------|
| Pipeline | The full sequence of automated steps triggered by a git push |
| Stage | A named phase. Jobs in the same stage run in parallel. Stages run in order. |
| Job | One unit of work inside a stage. It has a name, an image, and a script. |
| Runner | The machine that executes a job. It must be online for the job to run. |
| Image | The Docker image the runner uses to run your job (e.g. `maven:3.8-openjdk-11`) |
| Service | A sidecar container your job needs (e.g. `docker:dind` for Docker-in-Docker) |
| Artifact | A file or folder saved from a job so later jobs or humans can access it |
| Variable | A key-value pair. Use for secrets, image names, environment settings. |
| Tags | Labels that match a job to a specific runner |
| `only` / `rules` | Conditions that control when a job runs (e.g. only on the main branch) |

**Before writing any Copilot prompt today, make sure your prompt uses the correct term from this table.** "Run the build thing" is harder for Copilot to act on than "create a GitLab CI job in the build stage."

---

## 3. Set Up Your GitLab Runner

### What you need to understand

A Runner is the compute that executes your jobs. Without one online, your pipeline will hang forever waiting. Runners can be shared (provided by GitLab) or self-hosted.

For this workshop you have two options. Choose one.

**Option A — Shared Runner (quickest)**

GitLab.com provides free shared runners. Enable them in your project settings.

Where to look: Project → Settings → CI/CD → Runners → Shared runners

**Option B — Self-hosted Runner**

You register your own machine as a runner. This is more realistic for production environments.

### How to prompt Copilot

If you choose Option B, open a terminal and use Copilot to help you understand the registration command:

> "Explain what `gitlab-runner register` does and what each prompt means."

Then use Copilot to generate the install commands for your operating system:

> "Give me the commands to install gitlab-runner on [your OS]."

### What to configure

When registering, you will be asked for:

- The GitLab instance URL
- A registration token (find it in Settings → CI/CD → Runners)
- A description for your runner
- Tags — use `docker` and `java`
- An executor — use `docker`
- A default image — think about what language this project uses

### How to know it worked

Go to Project → Settings → CI/CD → Runners. Your runner should appear in the list with a status indicator showing it is online.

### Key concept: Runner Tags

Tags are how you tell a job which runner to use. If your job has `tags: [docker]` and your runner is tagged `docker`, they are matched. If there is no match, the job waits indefinitely.

You will use this in every job you write today.

---

## 4. Import the Java 11 Project

### What you need to understand

The project is a Maven-based Java 11 application hosted on GitHub at `biezhi/java11`. You need it in GitLab so your pipeline can act on it.

### Task

Get the project into a GitLab repository under your account.

### How to prompt Copilot

> "Give me the git commands to clone a GitHub repo and push it to a new GitLab remote."

After running the commands, confirm you can see the following files in your GitLab project:

```
pom.xml
src/
README.md
.gitignore
```

If `pom.xml` is missing, the Maven stages will fail. Do not continue until it is there.

### Understand `pom.xml` before writing pipeline code

Before you write a single CI line, ask Copilot:

> "Read this pom.xml and tell me: what Java version does it target, what does `mvn package` produce, and where does the output go?"

This is chain-of-thought preparation. You are building understanding before writing instructions.

---

## 5. Write Your Dockerfile

### What you need to understand

A Dockerfile is the recipe for turning your source code into a container image. You want the image to be as small and secure as possible, which means using a multi-stage build: one stage to compile, a separate and smaller stage to run.

### Think before you prompt

Answer these questions yourself first. Write the answers down. Then use your answers to build the Copilot prompt.

1. Which Docker base image has Maven and Java 11 available for building?
2. What command do you run to build the JAR without running tests?
3. Where does Maven put the output JAR by default?
4. Which base image is appropriate for running a Java 11 app (not building it)?
5. Should the container run as root? Why or why not?
6. What command starts a Java application from a JAR?

### How to prompt Copilot

Once you have answered the six questions above, construct a chain-of-thought prompt:

> "Write a multi-stage Dockerfile for a Maven Java 11 project. Stage 1 should [your answer to Q1, Q2, Q3]. Stage 2 should [your answer to Q4, Q5, Q6]."

The more specific your prompt, the less you will have to correct afterward.

### What Copilot might miss

After Copilot generates the Dockerfile, check it yourself:

- Is there a non-root user created and switched to before the ENTRYPOINT?
- Does Stage 2 copy from Stage 1 using `--from=`?
- Is the base image for Stage 2 a runtime image, not a full JDK?

If any of these are missing, prompt Copilot to fix the specific issue rather than regenerating the whole file.

### How to know it worked

Run this locally (if Docker is installed):

```bash
docker build -t java11-test .
docker run --rm java11-test
```

If the container starts without errors, your Dockerfile is correct.

---

## 6. Configure Docker Hub Publishing

### What you need to understand

You never store credentials in code. GitLab CI variables are the secure place for secrets. Your pipeline will read them at runtime using `$VARIABLE_NAME` syntax.

### Task A: Create a Docker Hub access token

Do not use your Docker Hub password directly. Create an access token instead.

Where: Docker Hub → your account → Account Settings → Security → New Access Token

Give it a name like `gitlab-workshop` and note down the token value. You will not see it again.

### Task B: Store the credentials in GitLab

Where: Project → Settings → CI/CD → Variables → Add variable

You need two variables. Decide on the names yourself — you will reference them by those names in your pipeline. Make the password/token masked so it never appears in logs.

### Task C: Understand image naming

Docker Hub images follow this format: `username/repository:tag`

For traceability, your tag should change with every commit. GitLab provides built-in variables for this. Ask Copilot:

> "What GitLab CI built-in variable gives me a short unique identifier for the current commit?"

Use that variable to build your image tag. Write down the full image reference format you will use before moving to the next section.

### How to know it worked

You will verify this in Section 8 when the pipeline runs end-to-end.

---

## 7. Add Trivy Security Scanning

### What you need to understand

Trivy is an open-source vulnerability scanner from Aqua Security. It inspects your container image for known CVEs (Common Vulnerabilities and Exposures) in OS packages and application dependencies.

The key design decision: the scan stage runs after the image is built but before it is published. If vulnerabilities are found above your threshold, the pipeline stops and nothing reaches Docker Hub.

### Think before you prompt

Answer these questions:

1. What Trivy Docker image would you use to run Trivy in a CI job?
2. What command scans a Docker image by name?
3. What flag tells Trivy to exit with a non-zero code when vulnerabilities are found?
4. What flag sets the minimum severity that causes a failure?
5. How do you save the scan output as a file so it can be downloaded from GitLab?

### How to prompt Copilot

Use your answers to construct a specific prompt. Example structure:

> "Write a GitLab CI job in the scan stage that uses the Trivy Docker image to scan [image reference]. It should fail the pipeline if [severity level] vulnerabilities are found, and save the report as an artifact."

### After Copilot responds

Review the output and ask yourself:

- Does the job have `entrypoint: [""]` set? (Required when overriding a Docker image's default entrypoint)
- Is `allow_failure` set to false so the pipeline actually stops on findings?
- Is the artifact configured with `when: always` so the report is saved even on failure?

If any are missing, ask Copilot to add them individually.

### How to know it worked

A passing scan means no HIGH or CRITICAL vulnerabilities. A failing scan means the pipeline correctly caught something. Both outcomes are correct behavior — one just needs you to update the base image.

---

## 8. Build Your Full Pipeline

### What you need to understand

Everything you have built so far becomes one file: `.gitlab-ci.yml`. This file lives at the root of your repository. GitLab reads it on every push.

### The pipeline shape

Your pipeline must have exactly these five stages, in this order:

```
compile → test → build → scan → publish
```

Each stage must only run if the previous one passed. This is enforced automatically by GitLab when you define stages in order.

### How to approach this with Copilot

Do not ask Copilot to write the whole file at once. Use chain-of-thought:

**Step 1 — Define the stages**

> "Write the stages block for a GitLab CI pipeline with these five stages in order: compile, test, build, scan, publish."

**Step 2 — Add global variables**

> "Add a variables block that defines the Maven local repository path, the Docker image name using my Docker Hub username variable, and the full image tag using the commit SHA variable."

**Step 3 — Add Maven caching**

> "Add a cache block that caches the Maven local repository between jobs."

**Step 4 — Add each job one at a time**

For each job, tell Copilot the stage, the Docker image to use, and the command to run. Reference the work you have already done in sections 3 through 7.

Use few-shot prompting here: write the compile job yourself (or with Copilot), then say:

> "Follow the same structure to write the test job, but save the JUnit XML reports as artifacts."

**Step 5 — Restrict jobs to the main branch**

> "Add an `only` block to the build, scan, and publish jobs so they only run on the main branch."

### Things to check before committing

Walk through this list yourself without asking Copilot:

- Does every job have a `tags` entry that matches your runner?
- Does the build job have `docker:dind` as a service?
- Is `DOCKER_TLS_CERTDIR` set to `/certs` in the build job?
- Does the scan job run after the build job?
- Does the publish job run after the scan job?
- Are your secret variable names consistent across the file?

### How to know it worked

Push the file to your main branch. Go to your project in GitLab → CI/CD → Pipelines. You should see a pipeline appear with five stages. Watch each one turn green in order.

If a job fails, click into it to read the log. Use that log output as your next Copilot prompt:

> "This GitLab CI job failed with the following error: [paste error]. What is the likely cause and how do I fix it?"

---

## 9. Final Checklist

Work through this list yourself. Do not tick a box until you have personally verified it — not just assumed it.

**Runner**
- [ ] Runner is visible in Project → Settings → CI/CD → Runners with an online status
- [ ] Runner has the `docker` tag
- [ ] Runner executor is set to `docker`

**Project**
- [ ] Repository contains `pom.xml` at the root
- [ ] Repository contains `Dockerfile` at the root
- [ ] Repository contains `.gitlab-ci.yml` at the root

**Variables**
- [ ] Docker Hub username is stored as a CI variable
- [ ] Docker Hub access token is stored as a masked CI variable
- [ ] No credentials appear anywhere in any committed file

**Pipeline**
- [ ] Pipeline triggers automatically on push to main
- [ ] All five stages appear: compile, test, build, scan, publish
- [ ] All five stages pass (green)
- [ ] A failed scan correctly stops the pipeline before publish

**Docker Hub**
- [ ] Image appears in your Docker Hub repository
- [ ] Image is tagged with the commit SHA
- [ ] Image is also tagged as `latest`

**Trivy**
- [ ] Scan report is downloadable from the pipeline artifacts
- [ ] You can explain what the severity levels mean and why you chose your threshold

---

## Copilot Prompting Reference

Use these as starting points. Adapt them to your specific context — generic prompts give generic results.

| What you want | Starting prompt |
|---------------|----------------|
| Understand a YAML key | "In a .gitlab-ci.yml file, what does [key] do and when would I use it?" |
| Debug a pipeline error | "This GitLab CI job failed with: [paste log]. What is the cause?" |
| Understand a Docker concept | "Explain what [concept] means in the context of a multi-stage Dockerfile." |
| Fix a specific line | "This line in my Dockerfile: [paste line]. What is wrong with it and how do I fix it?" |
| Understand a Trivy flag | "What does the Trivy flag [flag] do and what value should I use for a production scan?" |
| Add a feature to an existing job | "Here is my GitLab CI job: [paste job]. Add [feature] to it." |

---

## If You Are Stuck

In order, try:

1. Read the error message carefully. The answer is usually in the first line.
2. Ask Copilot with the exact error text pasted in.
3. Check the GitLab CI/CD documentation at docs.gitlab.com/ee/ci/
4. Ask your facilitator — describe what you tried before asking.

---

