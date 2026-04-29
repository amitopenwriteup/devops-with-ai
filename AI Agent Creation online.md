# AI Agent Creation Lab Guide
## Online Mode — No Installation Required

---

> **Mode:** Fully browser-based — no downloads, no installs
> **Cost:** Free (personal Google account required)
> **Tools used:** Google AI Studio · StackBlitz · Claude.ai
> **Duration:** ~3 hours (self-paced)

---

## Why Online Mode?

The original lab guide uses **Google Antigravity**, which requires a local desktop installation. This version replaces every local tool with a browser-based equivalent so you can complete the full workshop on any device — including school computers, Chromebooks, or shared machines.

| Original Tool (Antigravity) | Online Replacement | Purpose |
|---|---|---|
| Antigravity Editor View | StackBlitz | Code editor + live preview |
| Antigravity Manager View | Google AI Studio | Agent/prompt management |
| Antigravity Agent Chat | Google AI Studio (Gemini) or Claude.ai | AI agent instructions |
| Terminal panel | StackBlitz built-in terminal | Run commands in browser |
| Knowledge Base | AI Studio System Prompt | Reusable context |
| Artifacts panel | AI Studio conversation history | Review agent output |

---

## Before You Begin — Setup Checklist

Complete all steps before starting Lab 1.

- [ ] Personal **Google account** ready (Gmail) — work/enterprise accounts may have restrictions
- [ ] Modern browser: **Chrome** or **Edge** recommended
- [ ] Open **Google AI Studio** → [aistudio.google.com](https://aistudio.google.com) and sign in
- [ ] Open **StackBlitz** → [stackblitz.com](https://stackblitz.com) and sign in with GitHub or Google
- [ ] Optional: Open **Claude.ai** → [claude.ai](https://claude.ai) for complex reasoning tasks
- [ ] Confirm you can see both tabs open and working before starting

> **No GitHub account?** StackBlitz also supports Google sign-in. If you can't access either, use **CodeSandbox** ([codesandbox.io](https://codesandbox.io)) as an alternative — the steps are nearly identical.

---

## Understanding the Online Toolset

### Google AI Studio
Your **agent control panel**. This is where you write instructions (prompts), choose your model, and review what the AI produces. Think of it as the Antigravity Manager View + Agent Chat combined.

**Key features you will use:**
- **New prompt** — start a conversation with an AI model
- **System prompt** — persistent instructions the AI always follows (replaces the Knowledge Base)
- **Model selector** — choose between Gemini models (free quota available)
- **Run** button — sends your instruction to the AI
- **Conversation history** — acts as your Artifacts panel

### StackBlitz
Your **code editor and live preview environment**. StackBlitz runs entirely in the browser, gives you a file explorer, an editor, a terminal, and a live preview URL — exactly like Antigravity's Editor View.

**Key features you will use:**
- **File explorer** — left sidebar
- **Editor** — centre panel
- **Terminal** — bottom panel (supports `npm`, `node`, basic shell commands)
- **Preview** — right panel shows your app live as you edit

### Claude.ai (Optional but Recommended)
Use Claude for tasks that need deeper reasoning — architecture planning, debugging complex logic, or writing technical documentation. In the original guide, this maps to switching to **Claude Sonnet** for complex tasks.

---

## Lab 1 — Set Up Your Online Workspace

**Goal:** Create your working environment in StackBlitz and verify Google AI Studio is ready.

### Step 1 — Create a StackBlitz Workspace

1. Go to [stackblitz.com](https://stackblitz.com)
2. Click **+ New Project**
3. Select **Static HTML** (or search for "HTML/CSS/JS" in the template list)
4. Your workspace will open — you will see:
   - A file explorer on the left
   - An editor in the centre
   - A live preview on the right
5. Rename the project: click the project name at the top and type `antigravity-lab-online`

### Step 2 — Explore the StackBlitz Interface

Take 5 minutes to locate these panels:

- **File explorer** — left sidebar, shows `index.html`, `style.css`, etc.
- **Editor** — click any file to open and edit it
- **Terminal** — click the Terminal tab at the bottom. Try typing `ls` and pressing Enter
- **Preview panel** — live view of your app. Click the external link icon to open in a new browser tab

> **Tip:** If the preview panel doesn't appear automatically, click **Preview** in the top toolbar.

### Step 3 — Set Up Google AI Studio

1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in with your personal Google account
3. Click **+ Create new prompt** to open a blank session
4. In the model selector (top right), choose **Gemini 2.0 Flash** — this is your quota-efficient default, equivalent to Gemini 3 Flash in the original guide
5. Leave this tab open — you will use it throughout the labs

### Step 4 — Set Up Your System Prompt (Replaces Knowledge Base)

The **System Prompt** in AI Studio is persistent context the AI always reads — it replaces Antigravity's Knowledge Base.

1. In your AI Studio session, click **System instructions** (usually a collapsible panel above the conversation)
2. Paste the following:

```
You are a helpful coding agent. You write clean, working code.
When asked to build a web app, always:
- Use plain HTML, CSS, and JavaScript (no frameworks unless asked)
- Include comments in the code
- Validate form inputs before processing
- Use dark theme styling (#1a1a2e background, #e0e0e0 text) unless told otherwise
- Add JSDoc comments to all utility functions
When asked to plan, produce a numbered task plan before writing any code.
```

3. Click outside the panel to save it

> **What you just did:** You set up the equivalent of Antigravity's Knowledge Base. Every new prompt in this session will follow these instructions automatically.

---

## Lab 2 — Your First Agent Task

**Goal:** Use Google AI Studio to generate a working web app, then deploy it in StackBlitz.

### Step 1 — Write a Task Description

In your AI Studio conversation, type the following and click **Run**:

```
I want to build a simple to-do list web app.
It should let users add, complete, and delete tasks.
Use plain HTML, CSS, and JavaScript — no frameworks.

First, produce a numbered task plan showing the steps you will take.
Do not write any code yet — plan only.
```

### Step 2 — Review the Task Plan (Your Artifact)

The AI will respond with a task plan. This is the equivalent of Antigravity's **Task Plan artifact**. Read it carefully:

- Does it cover all the features you asked for?
- Are the steps logical?
- Is anything missing?

If you want to adjust it, reply with feedback before asking for code. For example:

```
The plan looks good. Also add a step for a "Clear all completed" button.
Now write the full code.
```

### Step 3 — Generate the Code

Once the plan is approved, send:

```
The plan is correct. Now write the complete code:
- index.html (full page including embedded CSS and JS)
That's all — one file only.
```

### Step 4 — Deploy in StackBlitz

1. Copy the HTML code from AI Studio's response
2. Switch to your StackBlitz tab
3. Click `index.html` in the file explorer
4. **Select all** existing content (`Ctrl+A` / `Cmd+A`) and **delete** it
5. **Paste** the copied code
6. StackBlitz auto-saves and the preview refreshes immediately

### Step 5 — Verify it Works (Your Screenshot Equivalent)

1. Click the **external link icon** in the preview panel to open the app in a full browser tab
2. Test it:
   - Add 3 tasks
   - Mark one as complete
   - Delete one
3. Take a browser screenshot (`Windows + Shift + S` on Windows, `Cmd + Shift + 4` on Mac) to document your result

### Step 6 — Give Feedback (Artifact Commenting Equivalent)

If something doesn't work, go back to AI Studio and describe the issue:

```
The delete button is not working. When I click it, nothing happens.
Please fix only the delete function — do not rewrite the whole file.
```

Paste the fix into StackBlitz when the AI responds.

> **What you just learned:** AI Studio's conversation history acts as your Artifacts panel. Each AI response is a reviewable artifact. You review before you deploy — giving you the same human-in-the-loop control as Antigravity.

---

## Lab 3 — Configure Agent Permissions

**Goal:** Understand how to control what the AI agent can and cannot do in an online workflow.

In Antigravity, you set **terminal permission policies**. In an online workflow, you control the AI through your prompt instructions and StackBlitz terminal permissions.

### Step 1 — Set Prompt-Level Constraints (Replaces Terminal Policy)

In AI Studio, you can control agent autonomy through the system prompt. Update your system prompt to add:

```
CONSTRAINTS:
- Never suggest running commands that delete files (rm -rf, del /f, etc.)
- Never suggest pushing to remote repositories without explicit instruction
- Always explain what a terminal command does before suggesting it
- Ask before suggesting any command that installs global packages
```

This is the equivalent of the **Request Review** policy — the AI will explain commands rather than silently include them.

### Step 2 — StackBlitz Terminal Safety

StackBlitz's browser terminal is sandboxed — it cannot affect your actual computer. However, good habits still matter:

- **Safe to run freely:** `npm install`, `node filename.js`, `npm run dev`, `python -m http.server`
- **Review before running:** any command with `sudo`, `rm -rf`, or `curl | bash`
- **Never needed:** `git push` (StackBlitz handles saving automatically)

### Step 3 — Practice the Review Habit

Ask AI Studio:

```
I want to start a local development server in StackBlitz for a plain HTML project.
What terminal command should I run, and what does it do?
```

Read the explanation before running anything in the terminal.

| Antigravity Policy | Online Equivalent |
|---|---|
| Always Proceed | Run all suggested commands without reading them (not recommended) |
| Request Review | Read AI's explanation before running any terminal command **(recommended)** |
| Never Execute | Only edit files manually, never use the terminal |

---

## Lab 4 — Run a Multi-Step Agent Task

**Goal:** Give the AI a complex mission, review the plan, then build stage-by-stage.

### Step 1 — Write the Mission

In AI Studio, start a **new conversation** (`+ Create new prompt`) and type:

```
Build a personal expense tracker web app with these features:
1. Add expenses with a name, amount, and category (Food, Transport, Other)
2. Show a running total at the top
3. List all expenses with a delete button for each
4. Use localStorage to save data between page refreshes
5. Style it cleanly with a dark theme

First, show me only the task plan and implementation plan.
List every section of the app and what it will contain.
Do not write any code yet.
```

### Step 2 — Review the Implementation Plan

The AI will produce a plan. Check:

- Does it mention all 5 features?
- Does it plan for localStorage?
- Does it describe the HTML structure, CSS approach, and JS functions separately?

Leave feedback before proceeding:

```
Add input validation to the plan — amount must be a positive number only.
Otherwise the plan looks correct. Now write the code.
```

### Step 3 — Build in StackBlitz

1. Copy the generated HTML from AI Studio
2. In StackBlitz, create a new file `expense-tracker.html`
3. Paste the code into it
4. Open the preview — your app should appear

### Step 4 — Test Each Feature

Manually test all 5 features:

1. Add 3 expenses with different categories
2. Verify the total updates correctly
3. Delete one expense — confirm it disappears and total updates
4. **Refresh the page** — confirm expenses are still there (localStorage test)
5. Try entering a negative number — confirm validation blocks it

### Step 5 — Fix Bugs via AI Studio

If anything is broken, describe it precisely in AI Studio:

```
The total is not updating when I delete an expense.
Here is the current deleteExpense function: [paste the function]
Fix only this function.
```

Paste only the fixed function into your file — not the whole file. This is faster and easier to review.

---

## Lab 5 — Run Two Agents in Parallel

**Goal:** Simulate Antigravity's parallel agent feature using two AI Studio sessions side by side.

### Step 1 — Open Two AI Studio Sessions

1. Open [aistudio.google.com](https://aistudio.google.com) in **Tab 1** — this is Agent A
2. Open [aistudio.google.com](https://aistudio.google.com) in **Tab 2** (right-click → Open in new tab) — this is Agent B
3. Arrange your browser so both tabs are visible (use Split View on Mac, Snap Assist on Windows, or two browser windows side by side)

### Step 2 — Assign Tasks

**In Tab 1 (Agent A)**, type and run:

```
Write a JavaScript utility file called utils.js that contains 
these functions: formatCurrency(amount), formatDate(dateString), 
and truncateText(text, maxLength). Add JSDoc comments to each function.
Output only the JavaScript code — no explanation needed.
```

**In Tab 2 (Agent B)**, type and run:

```
Write a README.md file for a personal expense tracker web app.
Include: project description, features list, how to run locally,
and a technologies section. Make it clear and professional.
Output only the markdown — no explanation needed.
```

### Step 3 — Monitor Both

Switch between tabs to watch progress. Both agents work independently. One may finish before the other — that's expected.

### Step 4 — Deploy Both Outputs in StackBlitz

1. Copy Agent A's output → Create `utils.js` in StackBlitz → Paste
2. Copy Agent B's output → Create `README.md` in StackBlitz → Paste

### Step 5 — Merge the Outputs with a Third Agent

Open a **Tab 3** in AI Studio and type:

```
I have two files from two different agents:

FILE 1 — utils.js:
[paste the full utils.js content here]

FILE 2 — README.md:
[paste the full README.md content here]

Update the README to include a "Utility Functions" section 
that documents each function from utils.js with its purpose and parameters.
Output only the updated README.md.
```

Replace your `README.md` in StackBlitz with the merged version.

> **What you just learned:** You can run independent AI agents in parallel using browser tabs. Each one works on a separate task. You review and merge their outputs — exactly like a team of developers working in parallel.

---

## Lab 6 — Build and Use a Reusable Knowledge Base

**Goal:** Save useful patterns so you can reuse them in future tasks without re-explaining them.

### Step 1 — Create a Knowledge File in StackBlitz

1. In StackBlitz, create a new file called `knowledge-base.md`
2. This file acts as your persistent knowledge store — you will paste its contents into AI Studio's system prompt for any new session

### Step 2 — Extract the localStorage Pattern

In AI Studio, ask:

```
From the expense tracker we built, extract the localStorage 
save-and-load pattern as a reusable code snippet.
Label it clearly: "localStorage: save and load array of objects"
Format it as a markdown code block with a short explanation.
```

Paste the AI's response into `knowledge-base.md` in StackBlitz.

### Step 3 — Save Your Project Conventions

Add the following to `knowledge-base.md` manually:

```markdown
## Project Conventions

- Dark theme: background #1a1a2e, text #e0e0e0
- Always validate form inputs before processing
- Always add JSDoc comments to utility functions
- Use localStorage for client-side data persistence
- No frameworks — plain HTML, CSS, JavaScript only
```

### Step 4 — Use the Knowledge Base in a New Session

1. Open a new AI Studio conversation
2. Click **System instructions**
3. Paste the full contents of `knowledge-base.md` into the system instructions
4. Now give it a fresh task:

```
Build a simple bookmark manager app.
Let users save a title and URL, list all bookmarks, and delete individual ones.
```

After it generates code, check:
- Did it use dark theme automatically? ✓
- Did it validate the URL input? ✓
- Did it use localStorage for persistence? ✓

---

## Lab 7 — Capstone Mission

**Goal:** Build a complete project from scratch using everything you have learned.

### The Brief

Build a **Daily Habit Tracker** using your online toolset.

**Requirements:**
- Users can add habits with a name and daily goal (number of times per day)
- Users can log a completion for each habit
- Progress bar shows completion percentage for each habit today
- Data persists via localStorage
- Works correctly on mobile screen sizes
- Includes a "Reset today" button that clears today's completions only

---

### Step 1 — Write a Clear Mission in AI Studio

Open a fresh AI Studio conversation. Set your system prompt with your full `knowledge-base.md` content.

Then type your mission — write it yourself in your own words. A well-written mission produces better output. Include:
- What the app does
- Every feature, listed clearly
- Technical constraints
- What "done" looks like

### Step 2 — Plan Before You Build

Send this first:

```
Do not write any code yet.
Produce only:
1. A numbered task plan (steps you will take)
2. An implementation plan (list every HTML section, CSS section, and JS function you will create)
```

Review the plan carefully. Leave feedback in the chat before proceeding.

### Step 3 — Build in Stages

Once the plan is approved, build one stage at a time.

**Stage 1 — Structure and Styling:**

```
The plan is approved. Implement Stage 1 only:
HTML structure and CSS styling. No JavaScript yet.
Output the full index.html file.
```

Paste into StackBlitz. Verify the layout looks correct in the preview before continuing.

**Stage 2 — JavaScript Functionality:**

```
Stage 1 looks correct. Now implement Stage 2:
All JavaScript functionality.
Use the localStorage pattern from the system prompt.
Output only the JavaScript — I will add it inside the <script> tag myself.
```

Add the JS to your `index.html` in StackBlitz.

**Stage 3 — Final Verification:**

```
I have deployed the app. Here is a list of tests I will run:
1. Add 3 habits
2. Log completions for each
3. Refresh the page — confirm data is still there
4. Use the "Reset today" button — confirm completions clear but habits remain

Describe exactly what the correct behaviour should look like for each test
so I can verify the app is working correctly.
```

Run each test manually in the StackBlitz preview.

### Step 4 — Self-Evaluate

After completing the capstone, fill in this checklist:

```
CAPSTONE CHECKLIST — ONLINE MODE

[ ] All 6 features from the brief are working
[ ] App loads without console errors (check browser DevTools → Console)
[ ] localStorage persists data across page refresh
[ ] Reset today button clears only today's completions
[ ] Layout looks reasonable on a narrow screen 
    (DevTools → Toggle device toolbar → iPhone SE)
[ ] I reviewed the task plan before asking for code
[ ] I built in stages (structure first, then JS)
[ ] I ran at least two AI Studio sessions in parallel (Lab 5 technique)
[ ] My knowledge-base.md was used in at least one session
```

---

## Free Tier — Limits and Tips

### What You Get for Free

| Tool | Free Tier |
|---|---|
| Google AI Studio | Generous free quota on Gemini 2.0 Flash |
| StackBlitz | Free with Google/GitHub sign-in, no time limits on public projects |
| Claude.ai | Free tier with daily message limit |
| Parallel sessions | Unlimited browser tabs (AI quota applies per account) |

### Tips to Stay Within Free Quota

1. **Use Gemini 2.0 Flash** as your default in AI Studio — most quota-efficient
2. **Write clear, specific missions** — vague prompts waste tokens on clarification
3. **Ask for plans first** — catching mistakes in the plan stage costs far less than regenerating code
4. **Request only what changed** — when fixing a bug, paste only the broken function, not the whole file
5. **Switch to Claude.ai** for complex reasoning tasks if you hit Gemini quota limits
6. **Save patterns in `knowledge-base.md`** — reusing saved context is cheaper than re-explaining

### When to Switch Tools

| Task | Recommended Tool |
|---|---|
| Most coding tasks | Google AI Studio (Gemini 2.0 Flash) |
| Complex reasoning, architecture | Claude.ai |
| Quick scripting | AI Studio (Gemini 2.0 Flash) |
| Code editing and live preview | StackBlitz |
| Reading and deploying code | StackBlitz |

---

## Troubleshooting

**AI loops or repeats itself**
→ Type: `Stop. Summarize what you have produced so far and what still needs to be done.` Then continue from where it left off.

**Preview in StackBlitz is blank**
→ Check the browser console (F12 → Console tab) for errors. Paste the error into AI Studio and ask for a fix.

**AI produces a framework (React, Vue) when you asked for plain HTML**
→ Add to your prompt: `Important: use only plain HTML, CSS, and vanilla JavaScript. No npm, no build tools, no frameworks.`

**localStorage not working in StackBlitz preview**
→ Open the preview in a separate tab (click the external link icon). Some browsers block localStorage in embedded iframes.

**Hit AI Studio quota limit mid-task**
→ Type: `Save a summary of what has been completed and what remains as a markdown list.` Paste that summary into a new session when quota resets, or switch to Claude.ai.

**StackBlitz project lost**
→ Projects are auto-saved to your account. Go to [stackblitz.com](https://stackblitz.com) → Dashboard to find it. Always keep a copy of important files in `knowledge-base.md` as a backup.

**Code is too long to paste easily**
→ Ask the AI: `Output only the section that changed, with the surrounding 3 lines for context so I know where to paste it.`

---

## Quick Reference Card

```
ONLINE MODE — LAB REFERENCE

TOOLS
  Google AI Studio  → Agent instructions, model selection, conversation history
  StackBlitz        → Code editor, file explorer, terminal, live preview
  Claude.ai         → Complex reasoning, architecture, documentation (optional)

KEY ACTIONS
  New agent session  → Open new AI Studio tab
  Review artifact    → Read AI response before copying code to StackBlitz
  Save knowledge     → Paste pattern into knowledge-base.md in StackBlitz
  Set conventions    → Add to AI Studio System Instructions
  Parallel agents    → Open multiple AI Studio tabs side by side
  Fix a bug          → Paste only the broken function, not the whole file

PERMISSION CONTROL (Prompt-Level)
  Full autonomy      → No constraints in system prompt (not recommended)
  Request Review     → "Always explain what a command does before suggesting it"
  Read-only          → "Do not suggest any terminal commands"

MODELS (free tier)
  Gemini 2.0 Flash  → Default, most quota-efficient (AI Studio)
  Claude             → Better for complex reasoning (Claude.ai)

ARTIFACT EQUIVALENTS
  Task plan          → AI's first response when you ask for plan only
  Implementation plan → File/function breakdown before code
  Code diff          → Ask AI: "Output only the changed function"
  Browser screenshot → Take manually from StackBlitz external preview tab

GOOD MISSION FORMULA
  1. What to build (specific, not vague)
  2. Every feature, clearly listed
  3. Technical constraints (no frameworks, use localStorage, etc.)
  4. What "done" looks like (how to verify it works)
  5. "Show me the plan before writing any code."

STAGE-BY-STAGE BUILD ORDER
  Stage 1 → Plan only (no code)
  Stage 2 → HTML + CSS only (no JS)
  Stage 3 → JavaScript functionality
  Stage 4 → Test and fix
```

---

*AI Agent Creation Lab Guide — Online Mode*
*Adapted from the Google Antigravity Free Tier Workshop*
*Tools: Google AI Studio · StackBlitz · Claude.ai*
*For training use — April 2026*
