# AI Agent Creation Lab Guide
## Google Antigravity (Free Tier) — Hands-On Training

---

> **Platform:** Google Antigravity (antigravity.google)
> **Cost:** Free (public preview — personal Gmail required)
> **Models available on free tier:** Gemini 3 Flash · Claude Sonnet 4.5 · GPT-OSS
> **Duration:** ~3 hours (self-paced)

---

## Before You Begin — Setup Checklist

Complete these steps before the lab starts. Each one is required.

- [ ] Personal Gmail account ready (work/enterprise accounts are NOT supported on free tier)
- [ ] Chrome browser installed (required for browser-in-the-loop features)
- [ ] Operating system is compatible:
  - macOS 12 Monterey or later
  - Windows 10 64-bit or later
  - Ubuntu 20+ / Debian 10+ / Fedora 36+ / RHEL 8+
- [ ] Download Antigravity from: **https://antigravity.google**
- [ ] Install and sign in with your personal Gmail
- [ ] Confirm you can see the **Editor View** and **Manager View** in the interface

---

## Lab 1 — Install and Explore the Interface

**Goal:** Get comfortable with Antigravity's two main surfaces before running any agent.

### Step 1 — Download and Install

1. Go to **https://antigravity.google**
2. Click **Download** and select your operating system
3. Run the installer and launch Antigravity
4. Sign in with your **personal Gmail account** when prompted
5. Wait for the welcome screen to appear — this confirms you are on the free tier

### Step 2 — Open a Workspace

1. Click **Open Folder** on the welcome screen
2. Create a new folder on your computer called `antigravity-lab`
3. Select that folder as your workspace
4. Antigravity will load your workspace — it looks and feels like VS Code

### Step 3 — Explore the Editor View

The Editor View is your standard coding environment. Take 5 minutes to locate:

- **File explorer** — left sidebar, shows your workspace files
- **Terminal panel** — bottom panel, open it with `Ctrl+`` (backtick) or `Cmd+``
- **Agent chat panel** — right sidebar, where you give instructions to the agent
- **Tab completion** — start typing in any file to see AI suggestions appear inline

### Step 4 — Switch to Manager View

1. Look for the **Manager** icon in the left activity bar (or use the View menu)
2. Click it to switch to Manager View
3. Notice the difference — this is your mission control for running multiple agents

**Manager View key areas:**
- **Agent slots** — up to 5 agents can run in parallel on the free tier (rate limited)
- **Artifacts panel** — shows task plans, implementation plans, and screenshots the agent generates
- **Status indicators** — shows each agent's current step and progress

### Step 5 — Check Your Model

1. Click the **model selector** (usually shown in the bottom status bar or agent panel header)
2. On the free tier you should see: **Gemini 3 Flash** as default
3. You can switch to **Claude Sonnet 4.5** or **GPT-OSS** for any task
4. For this lab, keep **Gemini 3 Flash** as your primary model to conserve free quota

> **Free Tier Tip:** The free tier includes Gemini 3 Flash with generous limits and basic access to Claude Sonnet 4.5 and GPT-OSS. Heavy use (2–3 hours of intensive sessions) can hit daily rate limits. If you hit a limit, wait 1 hour and resume.

---

## Lab 2 — Your First Agent Task (Editor View)

**Goal:** Run a simple single-agent task and read its output artifacts.

### Step 1 — Create a Starter File

1. In your `antigravity-lab` workspace, create a new file called `task.txt`
2. Type the following into it and save:

```
I want to build a simple to-do list web app.
It should let users add, complete, and delete tasks.
Use plain HTML, CSS, and JavaScript — no frameworks.
```

### Step 2 — Give the Agent Its First Instruction

1. Open the **Agent chat panel** on the right
2. Type the following instruction exactly:

```
Read task.txt and build the to-do list app described in it.
Create the files in this workspace. After you finish, 
open the app in the browser and take a screenshot to verify it works.
```

3. Press Enter to send

### Step 3 — Watch the Agent Work

Do not type anything while the agent is running. Observe:

- The agent will show a **Thought** step — its plan before acting
- It will create files in your workspace (you will see them appear in the file explorer)
- It will open the terminal and may run commands
- It will launch a browser preview and capture a screenshot

### Step 4 — Review the Artifacts

When the agent finishes, click the **Artifacts** panel to see what it produced:

- **Task plan** — the steps the agent decided to take
- **Implementation plan** — how it structured the code
- **Browser screenshot** — visual proof the app loaded correctly

### Step 5 — Give Feedback on an Artifact

1. In the Artifacts panel, find the **Task plan** artifact
2. Click the comment icon (like Google Docs commenting)
3. Type a feedback comment, for example:

```
Add a "Clear all completed" button to the feature list.
```

4. The agent will read your comment and adjust its plan — without you restarting the task

> **What you just learned:** Antigravity agents produce human-readable artifacts so you can verify what they are doing before accepting their output. Feedback on artifacts guides the agent in real time.

---

## Lab 3 — Configure Agent Permissions

**Goal:** Understand and set the terminal permission policy before running more powerful tasks.

The agent can execute terminal commands automatically. You control how much autonomy it has.

### Step 1 — Open Settings

1. Press `Cmd + ,` (Mac) or `Ctrl + ,` (Windows/Linux) to open Settings
2. Navigate to the **Terminal** section

### Step 2 — Choose Your Permission Policy

You will see three options. For this lab, select **Request Review**:

| Policy | What it means | When to use |
|--------|--------------|-------------|
| Always Proceed | Agent runs all terminal commands automatically | Experienced users, trusted tasks |
| Request Review | Agent asks before each terminal command | **Recommended for this lab** |
| Never Execute | Agent cannot run any terminal commands | Read-only tasks only |

### Step 3 — Set an Allow List (Optional)

If you want the agent to auto-run specific safe commands without asking:

1. In Settings → Terminal → Allow List, add:
```
npm install
npm run dev
python -m http.server
```

2. These commands will run automatically. All others will require your approval.

### Step 4 — Set a Deny List

Add commands you never want the agent to run without explicit confirmation:

```
rm -rf
sudo
git push
```

> **Security note:** Giving an agent terminal access is powerful but has risks. On the free tier, always use **Request Review** until you are comfortable reading the agent's task plans before approving actions.

---

## Lab 4 — Run a Multi-Step Agent Task

**Goal:** Give the agent a complex, multi-step mission and use artifacts to verify each stage.

### Step 1 — Create a New Task File

Create a file called `mission.txt` in your workspace with this content:

```
Build a personal expense tracker web app with these features:
1. Add expenses with a name, amount, and category (Food, Transport, Other)
2. Show a running total at the top
3. List all expenses with a delete button for each
4. Use localStorage to save data between page refreshes
5. Style it cleanly with a dark theme

After building it, test that adding and deleting expenses works correctly.
```

### Step 2 — Launch the Agent from Manager View

1. Switch to **Manager View**
2. Click **New Agent** (or the + button) to open an agent slot
3. In the agent input, type:

```
Read mission.txt. Build everything described in it, step by step.
Show me your task plan before writing any code.
```

4. Press Enter

### Step 3 — Review the Task Plan Before Proceeding

The agent will generate a **Task Plan artifact** first. Read it carefully:

- Does the plan match what you asked for?
- Are the steps in a logical order?
- Is anything missing?

If you want to adjust it, **leave a comment on the artifact** before the agent proceeds. For example:

```
Step 3 is correct but also add input validation — 
the amount field should only accept positive numbers.
```

### Step 4 — Monitor Each Stage

As the agent works through the plan, it will produce artifacts at each stage:

- **Implementation Plan** — which files it will create and why
- **Code diffs** — changes made to each file
- **Browser screenshots** — visual verification at the end

Watch for the agent requesting terminal approval if you set **Request Review**. For each request:
- Read the command carefully
- Click **Approve** if it looks correct
- Click **Reject** and leave a comment if something looks wrong

### Step 5 — Test the Final Output

When the agent finishes:

1. Open the app in your browser (the agent will provide a local URL)
2. Add 3 expenses manually
3. Refresh the page — confirm they are still there (localStorage test)
4. Delete one expense — confirm it disappears
5. Check the total updates correctly

If something is broken, go back to the agent panel and type:

```
The total is not updating when I delete an expense. Fix this.
```

The agent will locate the bug and fix it without rebuilding everything from scratch.

---

## Lab 5 — Run Two Agents in Parallel (Manager View)

**Goal:** Use the Manager View to dispatch two agents on different tasks simultaneously.

This is one of Antigravity's most unique features on the free tier.

### Step 1 — Prepare Two Task Files

Create `task-a.txt`:
```
Write a JavaScript utility file called utils.js that contains 
these functions: formatCurrency(amount), formatDate(dateString), 
and truncateText(text, maxLength). Add JSDoc comments to each.
```

Create `task-b.txt`:
```
Write a README.md file for a personal expense tracker web app.
Include: project description, features list, how to run locally,
and a technologies section. Make it clear and professional.
```

### Step 2 — Open Two Agent Slots

1. In Manager View, click **New Agent** — this is Agent A
2. Click **New Agent** again — this is Agent B

You will see two agent panels side by side (or stacked, depending on your screen size).

### Step 3 — Assign Tasks to Each Agent

In Agent A's input, type:
```
Read task-a.txt and complete the task described in it.
```

In Agent B's input, type:
```
Read task-b.txt and complete the task described in it.
```

Start both agents by pressing Enter in each panel.

### Step 4 — Monitor Both Agents

Watch the Manager View as both agents work simultaneously:

- Each agent has its own status indicator showing its current step
- Each agent produces its own artifacts independently
- If one agent finishes early, it will show a completion status while the other continues

### Step 5 — Merge the Outputs

When both agents are done:

1. Open `utils.js` — review the functions Agent A created
2. Open `README.md` — review what Agent B wrote
3. Open a third agent slot and type:

```
Review utils.js and README.md in this workspace. 
Update the README to include a "Utility Functions" section 
that documents each function from utils.js with its purpose and parameters.
```

> **What you just learned:** The Manager View lets you decompose a large project into parallel workstreams. Each agent works independently and you review and merge their outputs — exactly how a team of developers works.

---

## Lab 6 — Use the Knowledge Base

**Goal:** Save useful patterns to Antigravity's knowledge base so agents reuse them in future tasks.

### Step 1 — Understand the Knowledge Base

Antigravity agents learn from your sessions. When you mark something as worth remembering, it goes into a project knowledge base that future agents can access.

### Step 2 — Save a Code Pattern

After Lab 4, you have a working localStorage pattern. Let's save it:

1. Open the agent chat panel
2. Type:

```
Save the localStorage pattern used in the expense tracker 
to the knowledge base. Label it "localStorage: save and load array of objects".
```

3. The agent will extract the pattern and store it

### Step 3 — Save a Project Convention

Tell the agent your preferred conventions:

```
Save these conventions to the knowledge base for this project:
- Always use dark theme (#1a1a2e background, #e0e0e0 text)
- Always validate form inputs before processing
- Always add JSDoc comments to utility functions
```

### Step 4 — Test That the Knowledge Is Used

Open a new agent slot and give it a fresh task:

```
Build a simple bookmark manager app — let users save a title and URL, 
list all bookmarks, and delete individual ones.
```

After it completes, check:
- Did it use dark theme styling automatically?
- Did it validate the URL input?
- Did it use localStorage for persistence?

If yes — the knowledge base is working correctly.

---

## Lab 7 — Capstone Mission

**Goal:** Build a complete mini-project from scratch using everything you have learned.

### The Brief

Build a **Daily Habit Tracker** using Google Antigravity's free tier.

**Requirements:**
- Users can add habits with a name and daily goal (number of times per day)
- Users can log a completion for each habit
- Progress bar shows completion percentage for each habit today
- Data persists via localStorage
- Works correctly on mobile screen sizes
- Includes a "Reset today" button that clears today's completions only

### Step 1 — Write a Clear Mission File

Create `capstone.txt` and write the full brief in your own words. Be specific about every feature. A well-written mission file produces better agent output.

### Step 2 — Plan Before You Build

Open an agent and ask it to produce a plan only — no code yet:

```
Read capstone.txt. Do not write any code yet.
Produce a detailed task plan and implementation plan only.
List every file you will create and what each one will contain.
```

Review the plan carefully. Leave comments on anything that needs adjustment.

### Step 3 — Build in Stages

Once the plan looks correct, tell the agent:

```
The plan looks good. Now implement Stage 1 only: 
the HTML structure and CSS styling. Do not add any JavaScript yet.
Show me a browser screenshot when Stage 1 is done.
```

After Stage 1 is verified:

```
Stage 1 looks correct. Now implement Stage 2: 
all the JavaScript functionality. Use the localStorage pattern 
from the knowledge base.
```

After Stage 2:

```
Run a final check. Open the app in the browser, 
add 3 habits, log completions for each, then refresh the page 
and confirm the data is still there. Take a screenshot of the final result.
```

### Step 4 — Self-Evaluate

After your agent finishes, fill in this checklist:

```
CAPSTONE CHECKLIST

[ ] All 6 features from the brief are working
[ ] App loads without console errors
[ ] localStorage persists data across page refresh
[ ] Reset today button works correctly
[ ] Looks reasonable on a narrow (mobile) screen
[ ] Agent task plan matched what was actually built
[ ] I reviewed at least one artifact before approving agent actions
[ ] I used at least one parallel agent during the project
```

---

## Free Tier — Limits and Tips

### What the Free Tier Includes

| Feature | Free Tier |
|---------|-----------|
| Models | Gemini 3 Flash (primary), Claude Sonnet 4.5, GPT-OSS |
| Agents in parallel | Up to 5 (rate limited) |
| Browser-in-the-loop | Yes |
| Agent Manager | Yes |
| Knowledge base | Yes |
| Editor (VS Code extensions) | Yes |
| Daily quota | Limited — heavy use hits limits in 2–3 hours |

### Tips to Stay Within Free Quota

1. **Use Gemini 3 Flash** as your default model — it is the most quota-efficient on the free tier
2. **Write clear, specific mission files** — vague instructions waste tokens on clarification rounds
3. **Review the task plan first** before the agent writes any code — catching mistakes early saves quota
4. **Use parallel agents efficiently** — don't spawn 5 agents for tasks that can be done sequentially
5. **Save patterns to the knowledge base** — reusing saved knowledge costs less than the agent rediscovering it
6. **If you hit the rate limit**, wait one hour and resume — the daily quota resets automatically

### When to Switch Models

| Use case | Recommended model |
|---------|-----------------|
| Most coding tasks | Gemini 3 Flash |
| Complex reasoning, architecture planning | Claude Sonnet 4.5 |
| Quick prototypes, scripting | GPT-OSS |

---

## Troubleshooting

**Agent loops or repeats steps**
→ Open the agent panel and type: `Stop. Summarize what you have done so far and what still needs to be done.` Then restart from where it left off.

**Browser screenshot is blank or shows an error**
→ The agent's local server may not have started. Type: `Check if the development server is running. If not, start it and retry the browser check.`

**Rate limit hit mid-task**
→ Save your current state by typing: `Save a summary of what has been completed and what remains to a file called progress.txt.` Resume after one hour.

**Agent creates files in the wrong location**
→ Type: `Move all created files into the /src folder and update any file paths accordingly.`

**Terminal command rejected**
→ Read the command carefully in the approval dialog. If it looks safe, approve it. If unsure, type: `Explain what this terminal command does before I approve it.`

**VS Code extension not working**
→ Antigravity is built on VS Code — your existing extensions should import. If one is missing, install it from the Extensions panel exactly as you would in VS Code.

---

## Quick Reference Card

```
ANTIGRAVITY FREE TIER — LAB REFERENCE

VIEWS
  Editor View   → Code, terminal, inline agent assistance
  Manager View  → Spawn and monitor multiple agents in parallel

KEY ACTIONS
  New agent      → Manager View → click + New Agent
  Leave feedback → Click comment icon on any artifact
  Save knowledge → Tell agent: "Save [pattern] to the knowledge base"
  Check quota    → Bottom status bar shows model and usage indicator

PERMISSION POLICIES (Settings → Terminal)
  Always Proceed   → Full autonomy (not recommended for beginners)
  Request Review   → Approve each terminal command (recommended)
  Never Execute    → No terminal access

MODELS (free tier)
  Gemini 3 Flash  → Default, most quota-efficient
  Claude Sonnet 4.5 → Better for complex reasoning
  GPT-OSS          → Good for quick scripting

ARTIFACT TYPES
  Task plan           → Agent's step-by-step intention before acting
  Implementation plan → File structure and code approach
  Code diffs          → What changed in each file
  Browser screenshot  → Visual verification of running app

GOOD MISSION FILE FORMULA
  1. What to build (specific, not vague)
  2. Every feature, listed clearly
  3. Any technical constraints (no frameworks, use localStorage, etc.)
  4. What "done" looks like (how to verify it works)
```

---

*Google Antigravity AI Agent Creation Lab Guide*
*Free Tier Edition — April 2025*
*For training use. Visit https://antigravity.google to get started.*
