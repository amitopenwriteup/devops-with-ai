# DevOps AI Workshop — Day 1 Lab Guide

**Duration:** 4 hours (hands-on labs only)  
**Environment:** Linux · GitLab CI · Claude API  
**Prerequisite:** GitLab project with CI enabled, `ANTHROPIC_API_KEY` set as a masked CI/CD variable

---

## Setup Checklist (10 min)

Run these before starting Lab 1. Every participant must confirm all checks pass.

```bash
# 1. Verify tools
curl --version && jq --version && git --version && docker --version

# 2. Confirm GitLab token works
curl --silent --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/user" | jq '.username'

# 3. Confirm Anthropic API key works
curl --silent https://api.anthropic.com/v1/messages \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data '{
    "model": "claude-opus-4-20250514",
    "max_tokens": 50,
    "messages": [{"role": "user", "content": "Reply OK"}]
  }' | jq '.content[0].text'

# 4. Clone the workshop repo
git clone https://gitlab.com/<your-namespace>/devops-ai-workshop.git
cd devops-ai-workshop
```

> **Expected output for check 3:** `"OK"`  
> If any check fails, raise your hand before proceeding.

---

## Lab 1 — AI Code Review Gate (55 min)

### Goal

Add a CI job that reviews every merge request diff with Claude and posts the result as an MR comment. If Claude flags a `high` severity issue, the pipeline fails.

### Concepts covered

- Calling the Claude API from a shell script
- Parsing structured JSON with `jq`
- Using the GitLab MR API to post comments
- Failing a job on an AI-determined condition

---

### Step 1 — Write the review script (15 min)

Create `scripts/ai_review.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Collect the diff for this MR
DIFF=$(git diff origin/main --unified=5)

if [[ -z "$DIFF" ]]; then
  echo "No diff found. Skipping review."
  exit 0
fi

# Build the JSON payload (jq handles escaping safely)
PAYLOAD=$(jq -n \
  --arg diff "$DIFF" \
  '{
    model: "claude-opus-4-20250514",
    max_tokens: 1024,
    messages: [{
      role: "user",
      content: ("Review this git diff. Reply ONLY with valid JSON, no markdown fences:\n{\n  \"severity\": \"low|medium|high\",\n  \"summary\": \"one sentence\",\n  \"issues\": [\"issue 1\", \"issue 2\"]\n}\n\nDiff:\n" + $diff)
    }]
  }')

# Call Claude
RESPONSE=$(curl --silent \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data "$PAYLOAD" \
  https://api.anthropic.com/v1/messages)

# Extract the text content
AI_JSON=$(echo "$RESPONSE" | jq -r '.content[0].text')

echo "--- AI Review Output ---"
echo "$AI_JSON"

# Validate it is real JSON before proceeding
echo "$AI_JSON" | jq . > /dev/null 2>&1 || {
  echo "ERROR: AI did not return valid JSON. Raw response:"
  echo "$RESPONSE"
  exit 1
}

# Parse fields
SEVERITY=$(echo "$AI_JSON" | jq -r '.severity')
SUMMARY=$(echo "$AI_JSON" | jq -r '.summary')
ISSUES=$(echo "$AI_JSON" | jq -r '.issues[]' | sed 's/^/- /')

# Post comment to the MR via GitLab API
COMMENT_BODY="## AI Code Review\n\n**Severity:** \`$SEVERITY\`\n\n**Summary:** $SUMMARY\n\n**Issues:**\n$ISSUES"

curl --silent --request POST \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --header "content-type: application/json" \
  --data "$(jq -n --arg body "$COMMENT_BODY" '{body: $body}')" \
  "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/merge_requests/$CI_MERGE_REQUEST_IID/notes" \
  | jq '.id'

# Fail the pipeline if severity is high
if [[ "$SEVERITY" == "high" ]]; then
  echo "FAIL: High severity issues detected. Fix before merging."
  exit 1
fi

echo "PASS: Severity is $SEVERITY."
exit 0
```

```bash
chmod +x scripts/ai_review.sh
```

---

### Step 2 — Add the CI job (10 min)

Add this stage to `.gitlab-ci.yml`:

```yaml
stages:
  - review
  - test
  - build
  - deploy

ai-code-review:
  stage: review
  image: alpine:3.19
  before_script:
    - apk add --no-cache curl jq git
  script:
    - bash scripts/ai_review.sh
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  variables:
    GIT_DEPTH: 0        # needed for full diff against main
```

---

### Step 3 — Test it (20 min)

```bash
# Create a feature branch with an intentional issue
git checkout -b feature/lab1-review-test

# Add a file with a clear code smell
cat > src/example.py << 'EOF'
import os

def get_user(id):
    query = "SELECT * FROM users WHERE id = " + str(id)   # SQL injection
    password = "hardcoded_secret_123"                      # hardcoded secret
    return query
EOF

git add src/example.py
git commit -m "feat: add user lookup (lab 1 test)"
git push origin feature/lab1-review-test

# Open an MR on GitLab UI — watch the pipeline run
```

**Expected result:** Pipeline fails, MR comment appears with `severity: high`.

---

### Step 4 — Fix and re-test (10 min)

```bash
# Fix the issues
cat > src/example.py << 'EOF'
import os

def get_user(user_id: int) -> str:
    query = "SELECT * FROM users WHERE id = %s"
    return query
EOF

git add src/example.py
git commit -m "fix: parameterize query, remove hardcoded secret"
git push origin feature/lab1-review-test
```

**Expected result:** Pipeline passes, MR comment shows `severity: low` or `medium`.

---

### ✅ Lab 1 Done Criteria

- [ ] CI job runs only on MR pipelines
- [ ] MR receives an AI-generated comment
- [ ] Pipeline fails on `high` severity
- [ ] Pipeline passes after the fix commit

---

## Break (10 min)

---

## Lab 2 — AI Test Generation (55 min)

### Goal

Add a CI job that reads changed Python files and asks Claude to generate `pytest` test stubs. The generated tests are saved as a CI artifact and run by a downstream job.

### Concepts covered

- Using `git diff --name-only` to scope AI input
- Saving AI output as a CI artifact
- Chaining jobs with `needs:`
- Running AI-generated tests in the pipeline

---

### Step 1 — Write the generation script (15 min)

Create `scripts/generate_tests.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

mkdir -p tests/ai_generated

# Get list of changed Python source files (exclude test files)
CHANGED_FILES=$(git diff origin/main --name-only | grep '\.py$' | grep -v 'test_' || true)

if [[ -z "$CHANGED_FILES" ]]; then
  echo "No Python files changed. Skipping test generation."
  exit 0
fi

echo "Generating tests for:"
echo "$CHANGED_FILES"

for FILE in $CHANGED_FILES; do
  [[ -f "$FILE" ]] || continue

  FILE_CONTENT=$(cat "$FILE")
  BASENAME=$(basename "$FILE" .py)
  OUTPUT="tests/ai_generated/test_${BASENAME}_ai.py"

  PAYLOAD=$(jq -n \
    --arg code "$FILE_CONTENT" \
    --arg filename "$FILE" \
    '{
      model: "claude-opus-4-20250514",
      max_tokens: 2048,
      messages: [{
        role: "user",
        content: ("Generate pytest test cases for this Python file.\nRules:\n- Output ONLY valid Python code\n- No markdown fences, no explanation\n- Use parametrize for edge cases\n- Import the module correctly\n- Cover: happy path, empty input, boundary values, invalid types\n\nFile: " + $filename + "\n\n" + $code)
      }]
    }')

  TEST_CODE=$(curl --silent \
    --header "x-api-key: $ANTHROPIC_API_KEY" \
    --header "anthropic-version: 2023-06-01" \
    --header "content-type: application/json" \
    --data "$PAYLOAD" \
    https://api.anthropic.com/v1/messages | jq -r '.content[0].text')

  # Strip markdown fences if model added them anyway
  TEST_CODE=$(echo "$TEST_CODE" | sed '/^```/d')

  echo "$TEST_CODE" > "$OUTPUT"
  echo "Written: $OUTPUT"
done
```

```bash
chmod +x scripts/generate_tests.sh
```

---

### Step 2 — Add CI jobs (10 min)

```yaml
generate-tests:
  stage: test
  image: alpine:3.19
  before_script:
    - apk add --no-cache curl jq git python3 py3-pip
  script:
    - bash scripts/generate_tests.sh
  artifacts:
    paths:
      - tests/ai_generated/
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  variables:
    GIT_DEPTH: 0

run-ai-tests:
  stage: test
  image: python:3.12-slim
  needs:
    - job: generate-tests
      artifacts: true
  before_script:
    - pip install pytest --quiet
  script:
    - |
      if ls tests/ai_generated/test_*.py 1>/dev/null 2>&1; then
        pytest tests/ai_generated/ -v --tb=short
      else
        echo "No AI-generated tests found. Skipping."
      fi
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

---

### Step 3 — Test it (20 min)

```bash
git checkout -b feature/lab2-test-gen

cat > src/calculator.py << 'EOF'
def add(a: float, b: float) -> float:
    return a + b

def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def clamp(value: float, min_val: float, max_val: float) -> float:
    return max(min_val, min(max_val, value))
EOF

git add src/calculator.py
git commit -m "feat: add calculator module (lab 2 test)"
git push origin feature/lab2-test-gen
```

**Expected result:**

- `generate-tests` job creates `tests/ai_generated/test_calculator_ai.py`
- `run-ai-tests` job downloads the artifact and runs it
- Browse to the job in GitLab → **Browse** artifacts to read the generated file

---

### Step 4 — Inspect and tune (10 min)

Download the artifact and review it locally:

```bash
# Via GitLab UI: CI/CD → Jobs → generate-tests → Browse
# Or via API:
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/jobs/artifacts/\
feature%2Flab2-test-gen/download?job=generate-tests" \
  --output ai_tests.zip
unzip ai_tests.zip
cat tests/ai_generated/test_calculator_ai.py
```

**Discussion:** What did Claude get right? What did it miss? What would you add to the prompt?

---

### ✅ Lab 2 Done Criteria

- [ ] `generate-tests` job runs and creates artifact
- [ ] `run-ai-tests` job consumes artifact via `needs:`
- [ ] At least one generated test file is visible in artifacts
- [ ] Tests execute (pass or fail — both are valid outcomes)

---

## Break (10 min)

---

## Lab 3 — AI Failure Triage (50 min)

### Goal

Add a job that triggers only when a pipeline fails. It collects the failed job's log, sends it to Claude for root-cause analysis, and creates a GitLab issue with the AI diagnosis.

### Concepts covered

- `when: on_failure` jobs
- Fetching job logs from the GitLab Jobs API
- Creating GitLab issues programmatically
- Designing reliable structured output contracts

---

### Step 1 — Write the triage script (15 min)

Create `scripts/triage_failure.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Fetch the log of the most recently failed job in this pipeline
FAILED_JOB=$(curl --silent \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/pipelines/$CI_PIPELINE_ID/jobs?scope[]=failed" \
  | jq -r '.[0] | "\(.id) \(.name)"')

JOB_ID=$(echo "$FAILED_JOB" | awk '{print $1}')
JOB_NAME=$(echo "$FAILED_JOB" | awk '{print $2}')

echo "Triaging failed job: $JOB_NAME (ID: $JOB_ID)"

# Get last 150 lines of the job log
JOB_LOG=$(curl --silent \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/jobs/$JOB_ID/trace" \
  | tail -150)

# Send to Claude for diagnosis
PAYLOAD=$(jq -n \
  --arg log "$JOB_LOG" \
  --arg job "$JOB_NAME" \
  '{
    model: "claude-opus-4-20250514",
    max_tokens: 1024,
    messages: [{
      role: "user",
      content: ("Analyze this CI job failure log. Reply ONLY with valid JSON, no markdown:\n{\n  \"root_cause\": \"one clear sentence\",\n  \"affected_service\": \"service or component name\",\n  \"fix_suggestion\": \"concrete actionable fix in 1-2 sentences\",\n  \"confidence\": \"high|medium|low\"\n}\n\nJob name: " + $job + "\n\nLog tail:\n" + $log)
    }]
  }')

AI_JSON=$(curl --silent \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data "$PAYLOAD" \
  https://api.anthropic.com/v1/messages | jq -r '.content[0].text')

# Strip fences if present
AI_JSON=$(echo "$AI_JSON" | sed '/^```/d')

echo "--- AI Diagnosis ---"
echo "$AI_JSON"

# Validate JSON
echo "$AI_JSON" | jq . > /dev/null 2>&1 || {
  echo "ERROR: AI response is not valid JSON"
  exit 1
}

ROOT_CAUSE=$(echo "$AI_JSON"    | jq -r '.root_cause')
FIX=$(echo "$AI_JSON"           | jq -r '.fix_suggestion')
SERVICE=$(echo "$AI_JSON"       | jq -r '.affected_service')
CONFIDENCE=$(echo "$AI_JSON"    | jq -r '.confidence')
PIPELINE_URL="$CI_PROJECT_URL/-/pipelines/$CI_PIPELINE_ID"

# Create a GitLab issue
ISSUE_BODY="## Pipeline Failure — AI Triage Report\n\n\
**Failed job:** \`$JOB_NAME\`\n\
**Pipeline:** $PIPELINE_URL\n\
**Affected service:** $SERVICE\n\
**AI confidence:** $CONFIDENCE\n\n\
### Root cause\n$ROOT_CAUSE\n\n\
### Suggested fix\n$FIX\n\n\
---\n*Generated automatically by the AI triage job.*"

curl --silent --request POST \
  --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
  --header "content-type: application/json" \
  --data "$(jq -n \
    --arg title "Pipeline failure: $JOB_NAME — $ROOT_CAUSE" \
    --arg description "$ISSUE_BODY" \
    --arg label "ai-triage" \
    '{title: $title, description: $description, labels: $label}')" \
  "https://gitlab.com/api/v4/projects/$CI_PROJECT_ID/issues" \
  | jq '{id: .iid, url: .web_url}'

echo "Issue created."
```

```bash
chmod +x scripts/triage_failure.sh
```

---

### Step 2 — Add the CI job (10 min)

```yaml
ai-failure-triage:
  stage: .post
  image: alpine:3.19
  before_script:
    - apk add --no-cache curl jq
  script:
    - bash scripts/triage_failure.sh
  when: on_failure
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

> **Note:** `.post` is a built-in GitLab stage that always runs last, after all other stages.

---

### Step 3 — Test it (15 min)

```bash
git checkout -b feature/lab3-triage-test

# Introduce a build failure
cat > Dockerfile << 'EOF'
FROM python:3.12-slim
RUN pip install nonexistent-package-xyz==99.0.0
COPY . /app
EOF

# Add a build job to gitlab-ci.yml
cat >> .gitlab-ci.yml << 'EOF'

build-docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t test-image .
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
EOF

git add Dockerfile .gitlab-ci.yml
git commit -m "test: intentional docker failure for lab 3"
git push origin feature/lab3-triage-test

# Merge to main via GitLab UI to trigger the pipeline on main
```

**Expected result:**

- `build-docker` job fails
- `ai-failure-triage` job runs automatically
- A GitLab issue is created with the label `ai-triage`
- Issue contains root cause and fix suggestion from Claude

---

### ✅ Lab 3 Done Criteria

- [ ] Triage job uses `when: on_failure`
- [ ] Job fetches the real log from the GitLab Jobs API
- [ ] GitLab issue is created automatically on failure
- [ ] Issue body contains `root_cause` and `fix_suggestion` fields

---

## Break (5 min)

---

## Lab 4 — AI Deployment Risk Gate (50 min)

### Goal

Build a pre-deploy stage that scores the risk of a deployment using Claude. If the risk score is 70 or higher, the pipeline stops before production deploy and explains why.

### Concepts covered

- Feeding multiple signals to the AI (diff stats + commit log)
- Using `temperature: 0` for deterministic scoring
- `rules:` with `if:` to target production branches
- Failing with a human-readable message

---

### Step 1 — Write the risk scorer (15 min)

Create `scripts/risk_score.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Collect deployment signals
DIFF_STAT=$(git diff origin/main --stat | tail -5)
COMMIT_LOG=$(git log origin/main..HEAD --oneline)
CHANGED_FILES=$(git diff origin/main --name-only)
FILE_COUNT=$(echo "$CHANGED_FILES" | wc -l)

echo "=== Deployment signals ==="
echo "Files changed: $FILE_COUNT"
echo "$DIFF_STAT"
echo "Commits:"
echo "$COMMIT_LOG"

# Ask Claude to score the risk
PAYLOAD=$(jq -n \
  --arg stat "$DIFF_STAT" \
  --arg log "$COMMIT_LOG" \
  --arg files "$CHANGED_FILES" \
  --argjson count "$FILE_COUNT" \
  '{
    model: "claude-opus-4-20250514",
    max_tokens: 512,
    temperature: 0,
    messages: [{
      role: "user",
      content: ("Score the deployment risk for this change set. Reply ONLY with valid JSON:\n{\n  \"risk_score\": <integer 0-100>,\n  \"risk_level\": \"low|medium|high|critical\",\n  \"risk_reason\": \"one sentence explaining the main risk factor\",\n  \"risky_files\": [\"file1\", \"file2\"]\n}\n\nScoring guide:\n- 0-30: Low — small isolated changes\n- 31-69: Medium — moderate scope, some complexity\n- 70-89: High — large diff, critical files, or risky patterns\n- 90-100: Critical — DB migrations, auth changes, or mass deletions\n\nChange set:\nFiles changed: " + ($count | tostring) + "\nDiff stat:\n" + $stat + "\nCommit messages:\n" + $log + "\nChanged files:\n" + $files)
    }]
  }')

AI_JSON=$(curl --silent \
  --header "x-api-key: $ANTHROPIC_API_KEY" \
  --header "anthropic-version: 2023-06-01" \
  --header "content-type: application/json" \
  --data "$PAYLOAD" \
  https://api.anthropic.com/v1/messages | jq -r '.content[0].text')

AI_JSON=$(echo "$AI_JSON" | sed '/^```/d')

echo ""
echo "=== AI Risk Assessment ==="
echo "$AI_JSON" | jq .

# Validate
echo "$AI_JSON" | jq . > /dev/null 2>&1 || {
  echo "ERROR: AI response is not valid JSON. Failing safe."
  exit 1
}

SCORE=$(echo "$AI_JSON"  | jq -r '.risk_score')
LEVEL=$(echo "$AI_JSON"  | jq -r '.risk_level')
REASON=$(echo "$AI_JSON" | jq -r '.risk_reason')

echo ""
echo "Risk score : $SCORE / 100"
echo "Risk level : $LEVEL"
echo "Reason     : $REASON"

# Gate on threshold
THRESHOLD=${RISK_THRESHOLD:-70}

if [[ "$SCORE" -ge "$THRESHOLD" ]]; then
  echo ""
  echo "BLOCKED: Risk score $SCORE >= threshold $THRESHOLD"
  echo "Reason: $REASON"
  echo "Review the changes, get a second approval, or lower scope before deploying to production."
  exit 1
fi

echo ""
echo "APPROVED: Risk score $SCORE < threshold $THRESHOLD. Proceeding to deploy."
exit 0
```

```bash
chmod +x scripts/risk_score.sh
```

---

### Step 2 — Add CI jobs (10 min)

```yaml
risk-gate:
  stage: deploy
  image: alpine:3.19
  before_script:
    - apk add --no-cache curl jq git
  script:
    - bash scripts/risk_score.sh
  variables:
    GIT_DEPTH: 0
    RISK_THRESHOLD: "70"     # adjust per team
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  environment:
    name: production
    action: prepare

deploy-production:
  stage: deploy
  image: alpine:3.19
  script:
    - echo "Deploying to production..."
    - echo "Deploy steps go here"
  needs:
    - job: risk-gate
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  environment:
    name: production
```

---

### Step 3 — Test with a risky change (15 min)

```bash
# Test 1: risky change (should be BLOCKED)
git checkout -b feature/lab4-risky-deploy

# Simulate a large risky change
cat > database/migrations/0042_drop_user_table.sql << 'EOF'
-- Migration: restructure user data
DROP TABLE IF EXISTS users CASCADE;
DROP TABLE IF EXISTS sessions CASCADE;
ALTER TABLE accounts DROP COLUMN password_hash;
EOF

cat >> src/auth.py << 'EOF'

# Temporary: disable auth for testing
AUTH_BYPASS = True

def verify_token(token):
    if AUTH_BYPASS:
        return True
    return _verify_jwt(token)
EOF

git add .
git commit -m "refactor: drop legacy user tables and simplify auth"
git push origin feature/lab4-risky-deploy

# Merge to main and watch the pipeline
```

**Expected result:** `risk-gate` blocks with score ≥ 70. `deploy-production` never runs.

```bash
# Test 2: safe change (should PASS)
git checkout -b feature/lab4-safe-deploy

cat > docs/CHANGELOG.md << 'EOF'
## v1.2.1 — patch
- Fix typo in error message
- Update README link
EOF

git add docs/CHANGELOG.md
git commit -m "docs: update changelog"
git push origin feature/lab4-safe-deploy
```

**Expected result:** `risk-gate` passes with low score. `deploy-production` runs.

---

### Step 4 — Tune the threshold (10 min)

Experiment by changing `RISK_THRESHOLD` as a CI/CD variable in **Settings → CI/CD → Variables** (no pipeline change needed).

| Threshold | Behaviour |
|-----------|-----------|
| `90` | Only blocks critical changes (DB drops, auth bypasses) |
| `70` | Blocks high and critical (recommended default) |
| `50` | Blocks anything medium or above |

**Discussion:** What threshold makes sense for your team? Who should be able to override a blocked risk gate?

---

### ✅ Lab 4 Done Criteria

- [ ] `risk-gate` job runs before `deploy-production`
- [ ] Risky commit is blocked with a clear message
- [ ] Safe commit passes through to deploy
- [ ] `RISK_THRESHOLD` is configurable as a CI/CD variable

---

## Day 1 Wrap-up (15 min)

### Patterns built today

| Lab | CI Pattern | AI role |
|-----|-----------|---------|
| 1 | MR code review gate | Reviewer |
| 2 | Automated test generation | Developer |
| 3 | On-failure incident triage | SRE |
| 4 | Pre-deploy risk scoring | Risk officer |

### Key rules for AI in CI pipelines

1. **Always validate JSON** — use `jq .` before accessing any field. Exit on parse failure.
2. **Use `temperature: 0`** for scoring/classification jobs. Use higher temperature only for generation tasks.
3. **Set `max_tokens`** on every call — never leave it unbounded.
4. **Log every AI call** — save the raw response as a CI artifact for auditability.
5. **Design for failure** — if the AI call fails (network, quota), the job should fail safely, not silently continue.

### Bonus challenges

```bash
# Bonus 1 — Lab 1: add a severity emoji to the MR comment
#   low=✅  medium=⚠️  high=❌

# Bonus 2 — Lab 2: add pytest --cov and fail if coverage drops below 60%
pip install pytest-cov
pytest tests/ --cov=src --cov-fail-under=60

# Bonus 3 — Lab 3: also post the triage result to Slack
curl -X POST -H 'Content-type: application/json' \
  --data "{\"text\": \"Pipeline failed: $ROOT_CAUSE\"}" \
  "$SLACK_WEBHOOK_URL"

# Bonus 4 — Lab 4: add a manual override job that lets a senior engineer
#   bypass the risk gate with a justification comment
```

### Day 2 preview

- AI-assisted Dockerfile optimization
- Semantic changelog generation from commits
- Building a custom GitLab bot that responds to MR slash commands
- Cost tracking: measuring and capping AI token spend per pipeline

---

*Workshop materials: https://gitlab.com/your-namespace/devops-ai-workshop*  
*Questions → create an issue with label `workshop-question`*
