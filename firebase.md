# Workshop: "Environment-as-Code" with Firebase Studio

> **Format:** Hands-on lab · **Platform:** Firebase Studio (Project IDX)

---

## Prerequisites

- A Google Account
- Access to [idx.google.com](https://idx.google.com)
- A GitLab account (needed for Steps 3 & 4)
- Basic familiarity with a terminal

---

## Step 1 — Provisioning the DevOps Workspace

Instead of installing local tools, we spin up a cloud-based Linux environment.

1. Go to [idx.google.com](https://idx.google.com) and sign in.
2. Click **New Workspace** → select **Web App** (or "Blank").
3. Name your project: `devops-workshop-2026`.
4. Wait for the environment to provision (~60–90 seconds). You now have a dedicated VM with an integrated terminal.
5. Verify the terminal works — type the following and confirm you see output:

```bash
echo "hello devops"
```

> **Key Concept:** This cloud workspace is ephemeral and reproducible. If it breaks, you can destroy it and provision a fresh one in under 2 minutes — with all tools pre-installed.

---

## Step 2 — Defining the Infrastructure (Nix)

In DevOps, we never manually install tools. We define them in code.

### Part A — Edit dev.nix

1. In the file explorer, locate and open `.idx/dev.nix` (enable "show hidden files" if needed).
2. Replace the contents with the block below and save the file.

```nix
{ pkgs, ... }: {
  channel = "stable-23.11";

  packages = [
    pkgs.nodejs_20     # Runtime for the React app
    pkgs.terraform     # Infrastructure as Code
    pkgs.glab          # GitLab CLI
    pkgs.htop          # Process monitoring
    pkgs.jq            # JSON processor
  ];

  idx.extensions = [
    "hashicorp.terraform"
    "gitlab.gitlab-workflow"
  ];
}
```

### Part B — Rebuild & Verify

3. Click the **"Rebuild Environment"** button that appears in the bottom-right of the editor.
4. Once finished (~30–60 seconds), run the verification commands in the terminal:

```bash
terraform --version   # Terraform v1.7.0
glab --version        # glab version 1.x.x
node --version        # v20.11.0
```

> **Why Nix matters:** You never ran a single `apt install` or `brew install`. The environment is fully described in one file — this is the core of "Environment-as-Code."

---

## Step 3 — AI-Assisted CI/CD Planning

We use the integrated Gemini AI to plan and generate our automation pipeline.

### Part A — Generate the pipeline with Gemini

1. Open the **Gemini Chat** sidebar (sparkle icon on the left activity bar).
2. Send the following prompt:

```
I am building a React app on Firebase.
Generate a GitLab CI/CD YAML file that:
  1. Runs on merge request to 'main' only
  2. Installs dependencies with npm ci
  3. Runs the test suite with npm test
  4. Deploys to Firebase Hosting on test success

Use GitLab CI/CD variables for credentials.
```

### Part B — Create the pipeline file

3. Create a file named `.gitlab-ci.yml` in your project root and paste the AI-generated YAML. Reference template:

```yaml
image: node:20

stages:
  - test
  - deploy

variables:
  NODE_ENV: production

cache:
  paths:
    - node_modules/

test:
  stage: test
  script:
    - npm ci
    - npm test
  only:
    - merge_requests
    - main

deploy:
  stage: deploy
  script:
    - npm ci
    - npm install -g firebase-tools
    - firebase deploy --token "$FIREBASE_TOKEN" --only hosting
  only:
    - main
  environment:
    name: production
    url: https://your-project.web.app
```

### Part C — Discover required CI/CD variables

5. Ask Gemini the follow-up:

```
What CI/CD variables do I need to add to GitLab for this to work?
Walk me through adding FIREBASE_TOKEN step by step.
```

> **Warning:** Never commit the `FIREBASE_TOKEN` value to your repository. Store it in GitLab → Settings → CI/CD → Variables, and mark it as **Masked**.

---

## Step 4 — Reliability & Testing

A key DevOps pillar is "shifting left" — catching bugs as early as possible, before code ever reaches staging or production.

### Part A — Generate a test with Gemini

1. In the Gemini chat, send this prompt:

```
Create a simple Vitest test for a React login component.
The component has:
- An email input (data-testid="email-input")
- A password input (data-testid="password-input")
- A submit button (data-testid="submit-btn")

Test that: (1) the form renders, (2) inputs accept values,
and (3) the submit button is clickable.
```

2. Save the generated test as `src/__tests__/Login.test.jsx`.

### Part B — Automate startup via Nix hooks

3. Reopen `.idx/dev.nix` and add the following `idx` block inside the main `{ }`:

```nix
  idx = {
    # Runs once when the workspace is first created
    workspace.onCreate = {
      install-deps = "npm install";
    };

    # Runs every time the workspace starts
    workspace.onStart = {
      dev-server = "npm run dev -- --port 3000";
    };
  };
```

4. Trigger a **Rebuild Environment** to apply the changes.

> **Team impact:** Any teammate who clones this workspace gets a fully configured environment — dependencies pre-installed and the dev server auto-started — in one click. Zero onboarding friction.

---

## Step 5 — Capacity Planning Simulation

Use Gemini to simulate a DevOps "Incident Command" scenario before you hit production limits.

1. In the Gemini chat, send this prompt:

```
Analyze my current Firebase project structure.

If I expect 5,000 requests per second tomorrow,
which Firebase services will hit their Spark (free)
tier limits first?

Please give me:
1. A ranked list of services by which breaks first
2. The specific limit for each (requests/day, reads/day, etc.)
3. Recommended Blaze plan upgrade thresholds
4. One mitigation strategy per service
```

2. Review the breakdown Gemini returns. Key Spark tier limits to watch:

| Service | Free Limit |
|---|---|
| Cloud Functions | 125,000 invocations/month |
| Firestore reads | 50,000 reads/day |
| Hosting bandwidth | 10 GB/month |
| Authentication | 10,000 users/month |

3. Follow-up: *"How do I add Firebase budget alerts?"* — this teaches you to get notified before you're charged.

---

## Final Deliverable

By the end of this workshop, you have built four production artifacts:

| # | Deliverable | File | What it does |
|---|---|---|---|
| 1 | Reproducible Environment | `.idx/dev.nix` | Any teammate gets identical tooling in one click |
| 2 | Deployment Pipeline | `.gitlab-ci.yml` | Automated test + deploy on every merge to main |
| 3 | Automated Setup | Nix `onCreate` hook | Deps install and dev server starts automatically |
| 4 | Scaling Plan | Gemini analysis | Firebase tier limits mapped against 5K req/s traffic |

---

## Complete dev.nix Template

Copy-paste ready for the full workshop configuration:

```nix
{ pkgs, ... }: {
  channel = "stable-23.11";

  packages = [
    pkgs.nodejs_20
    pkgs.terraform
    pkgs.glab
    pkgs.htop
    pkgs.jq
  ];

  idx = {
    extensions = [
      "hashicorp.terraform"
      "gitlab.gitlab-workflow"
      "vitest.explorer"
    ];

    workspace.onCreate = {
      install-deps = "npm install";
    };

    workspace.onStart = {
      dev-server = "npm run dev -- --port 3000";
    };

    previews = {
      enable = true;
      previews = {
        web = {
          command = ["npm" "run" "dev"];
          manager = "web";
          env = {
            PORT = "$PORT";
          };
        };
      };
    };
  };
}
```

---

## Cleanup

- **Keep the workspace:** It stays saved in your IDX dashboard indefinitely.
- **Delete the workspace:** Go to [idx.google.com](https://idx.google.com) → three-dot menu on the workspace card → **Delete workspace**.

---

## Next Steps

- Add a staging environment using GitLab Environments and protected branches
- Configure the Firebase Emulator Suite inside your Nix environment
- Use Terraform to provision Firebase resources declaratively
- Explore GitLab's built-in container registry for Docker-based deployments
