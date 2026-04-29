# Microsoft Copilot Studio — Workshop Lab Guide

**Duration:** 3 hours (self-paced)
**Level:** Beginner — no coding required
**Requirement:** Microsoft account (work or school account recommended)
**URL:** [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)

---

## Before You Begin — Setup Checklist

Complete these steps before the lab starts.

- [ ] Microsoft account ready (work, school, or personal Microsoft account)
- [ ] Chrome or Edge browser installed
- [ ] Go to [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com) and confirm you can sign in
- [ ] You can see the **"What would you like to build?"** home screen
- [ ] Check the top right — confirm your **Environment** is set to **Default Environment**
- [ ] Optional: Have a document (PDF or Word file) or a website URL ready to use as knowledge

> **Free Trial Note:** Copilot Studio offers a free trial. You do not need a paid licence to complete this lab. If prompted, start the free trial when you sign in.

---

## What You Will Build

By the end of this lab you will have built and published a working **HR FAQ Bot** — an AI agent that:

- Answers employee questions about leave, benefits, and company policy
- Reads answers from your own documents or websites
- Handles guided conversations using Topics
- Is published and accessible inside Microsoft Teams

---

## Key Concepts — Read This First

Before starting, understand these four terms. They come up throughout the lab.

| Term | What it means |
|------|--------------|
| **Agent** | The AI chatbot you build. It has a name, a purpose, and instructions. |
| **Knowledge** | Documents, websites, or SharePoint pages the agent reads to find answers. |
| **Topic** | A scripted conversation flow. When a user says X, the bot asks Y, then does Z. |
| **Action** | Something the bot does automatically — send an email, create a task, fill a form. |

---

## Lab 1 — Sign In and Explore the Interface

**Goal:** Get comfortable with the Copilot Studio interface before building anything.

### Step 1 — Sign In

1. Go to [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Click **Sign in** and use your Microsoft account
3. If prompted to start a free trial, click **Start free trial**
4. Wait for the home screen to load — you should see **"What would you like to build?"**

### Step 2 — Explore the Home Screen

You will see this layout on the home screen:

- **"What would you like to build?"** — the main prompt bar in the centre
- **Agent / Workflow toggle** — two buttons just below the heading:
  - **Agent** (selected by default, shown in teal/blue)
  - **Workflow**
- **Start building from scratch** — three cards below the toggle:

| Card | What it builds |
|------|---------------|
| **Create workflow** | Automations that run in the background (like Power Automate) |
| **Agent** | An AI chatbot that answers questions, shares knowledge, and handles tasks |
| **Computer-using agent** | An agent that can operate apps and websites directly on screen |

> **For this lab, we will use the Agent card.**

### Step 3 — Understand the Left Navigation Bar

The left sidebar shows these icons (top to bottom):

| Icon | Label | What it does |
|------|-------|-------------|
| 🏠 | **Home** | Returns to the "What would you like to build?" screen |
| 🤖 | **Agents** | Lists all the agents (bots) you have created |
| ⚡ | **Flows** | Lists all the workflows and automations you have built |
| 🔧 | **Tools** | Connectors, plugins, and tools you can add to agents |
| **...** | **More** | Additional settings and options |

### Step 4 — Check Your Environment

1. Look at the **top right corner** of the screen
2. You will see **Environment: Default Environment**
3. This tells you which Microsoft Power Platform environment you are working in
4. For this lab, leave it as **Default Environment** — do not change it

> **What you just learned:** The home screen separates Agents (chatbots) from Workflows (automations). The left sidebar — Home, Agents, Flows, Tools — is your main navigation throughout the platform.

---

## Lab 2 — Create Your First Agent

**Goal:** Create a new agent with a name and clear purpose in under 5 minutes.

### Step 1 — Open the Agent Builder

From the home screen, you have two options:

**Option A — Use the prompt bar:**
1. Make sure **Agent** is selected in the toggle (it shows in teal/blue when active)
2. Click inside the large text box that says **"Start building by describing what your agent needs to do"**
3. Type:
   ```
   An HR assistant that answers employee questions about leave, benefits, and company policy.
   ```
4. Click the **→ arrow** on the right side of the bar

**Option B — Use the Agent card (recommended for beginners):**
1. Scroll down to **Start building from scratch**
2. Click the **Agent** card — the middle card that says *"Flexible solutions that take action, share knowledge, and handle tasks"*
3. This opens the agent configuration screen directly

### Step 2 — Fill In the Agent Details

On the configuration screen, fill in these fields:

| Field | What to type |
|-------|-------------|
| **Name** | `HR FAQ Bot` |
| **Description** | `Answers employee questions about leave, benefits, payroll, and company policy.` |
| **Instructions** | `You are a helpful HR assistant. Only answer questions using the knowledge provided. If you do not know the answer, say: Please contact the HR team directly at hr@yourcompany.com` |

### Step 3 — Create the Agent

1. Review the name and instructions — make sure they look right
2. Click **Create**
3. Copilot Studio builds your agent in a few seconds
4. You will land on the agent's main configuration page

### Step 4 — Test the Empty Agent

1. Find the **Test** button — usually at the top right corner of the screen
2. Click it to open the test chat panel on the right
3. Type: `What is the leave policy?`
4. The agent will give a vague or empty answer — this is expected because no knowledge has been added yet
5. Close the test panel for now

> **What you just learned:** An agent with no Knowledge gives generic or empty answers. The next lab adds real content for it to learn from.

---

## Lab 3 — Add Knowledge

**Goal:** Give your agent a document or website to read answers from.

### Step 1 — Find the Knowledge Section

Inside your agent, look at the left panel or the tabs along the top of the editor. Click **Knowledge**.

### Step 2 — Add a Knowledge Source

1. Click **Add knowledge**
2. Choose the type of source:

| Option | When to use |
|--------|-------------|
| **Public website** | You have a URL to a public webpage — easiest for this lab |
| **Files** | You have a PDF, Word, or text file to upload |
| **SharePoint** | You have content in SharePoint (work/school account required) |
| **Dataverse** | You have structured data in Microsoft Dataverse |

### Step 3 — Add Your Source (pick one)

**Option A — Public website:**
1. Select **Public website**
2. Paste a URL — for example:
   `https://www.acas.org.uk/holiday-entitlement`
   *(or any relevant HR or policy page from your organisation)*
3. Click **Add**

**Option B — Upload a file:**
1. Select **Files**
2. Click **Upload file**
3. Choose a PDF or Word document from your computer
4. Click **Add**

### Step 4 — Wait for Indexing

- Copilot Studio reads and processes your content automatically
- This takes **1–2 minutes**
- Watch for the status indicator — wait until it shows **Ready**
- Do not test the agent until the status shows Ready

### Step 5 — Test With Knowledge

1. Click **Test** (top right)
2. Ask a question that your document or website covers
3. The agent should now reply with a proper, sourced answer
4. It may show a citation or reference to the source it used

> **What you just learned:** Knowledge is the most important part of a helpful agent. It only answers from what you add here — it will not guess or invent information outside of the provided content.

---

## Lab 4 — Build a Topic (Guided Conversation)

**Goal:** Create a scripted conversation flow for booking annual leave.

Knowledge handles open questions. Topics handle guided step-by-step tasks where you collect information from the user.

### Step 1 — Go to Topics

1. Inside your agent, click **Topics** in the left panel or top tabs
2. You will see system topics already created (Greeting, Goodbye, Fallback, etc.)
3. Click **Add a topic** → **From blank**

### Step 2 — Name the Topic

1. At the top of the canvas, click on **Untitled** and type: `Book Annual Leave`
2. Press Enter or click away to save

### Step 3 — Add Trigger Phrases

Trigger phrases are what the user types to activate this topic.

1. Click inside the **Trigger** node at the top of the canvas
2. Add these phrases one by one (press Enter after each):
   - `I want to book leave`
   - `How do I request annual leave`
   - `Book time off`
   - `Request holiday`
   - `I need to take some leave`

### Step 4 — Add a Welcome Message

1. Click the **+** button below the Trigger node
2. Select **Send a message**
3. Type: `I can help you book annual leave. I just need a few details from you.`

### Step 5 — Ask for the Start Date

1. Click **+** → **Ask a question**
2. In the question field, type: `What is your start date for the leave?`
3. Under **Identify**, select **Date**
4. Under **Save response as**, name the variable: `LeaveStartDate`

### Step 6 — Ask for the End Date

1. Click **+** → **Ask a question**
2. Type: `And what is your end date?`
3. Under **Identify**, select **Date**
4. Name the variable: `LeaveEndDate`

### Step 7 — Add a Confirmation Message

1. Click **+** → **Send a message**
2. Use the **variable picker** (the `{}` curly brace icon inside the editor) to insert your variables
3. Build this message:

```
Thank you! Your leave request has been noted.

Start date: {LeaveStartDate}
End date: {LeaveEndDate}

Your manager will receive a notification to approve this. You will hear back within 2 business days.
```

> **Important:** Always use the variable picker `{}` button inside the message editor to insert variables. Do not type curly braces manually — they will not be replaced with real values.

### Step 8 — Save and Test

1. Click **Save** (top right)
2. Click **Test**
3. Type: `I want to book leave`
4. Answer the date questions when prompted
5. Confirm the summary message appears correctly at the end

> **What you just learned:** Topics let you collect information step by step using a visual drag-and-drop canvas. Saved answers become variables you can use in later messages.

---

## Lab 5 — Edit the Greeting Topic

**Goal:** Personalise the agent's greeting so it feels professional and sets clear expectations.

### Step 1 — Find the Greeting Topic

1. Click **Topics** in the left panel
2. Look for **Greeting** in the System topics list
3. Click it to open the canvas

### Step 2 — Edit the Greeting Message

1. Find the **Send a message** node in the canvas
2. Clear the default text
3. Replace it with:

```
Hello! I am the HR FAQ Bot.

I can help you with:
- Leave and holiday questions
- Benefits and payroll queries
- Company policy information
- Booking annual leave

Just type your question and I will do my best to help.
For urgent matters, please contact hr@yourcompany.com
```

### Step 3 — Save and Test

1. Click **Save**
2. Click **Test** and type `Hello`
3. Confirm the new greeting appears correctly

---

## Lab 6 — Edit the Fallback Topic

**Goal:** Make sure the agent handles unknown questions gracefully instead of going silent.

### Step 1 — Find the Fallback Topic

1. In **Topics**, find **Fallback** in the System topics list
2. Click to open it

### Step 2 — Edit the Fallback Message

Replace the default text with:

```
I'm sorry, I don't have the answer to that question yet.

Here is what you can do:
- Try rephrasing your question
- Email the HR team at hr@yourcompany.com
- Visit the HR portal for more information

Is there anything else I can help you with?
```

### Step 3 — Save and Test

1. Click **Save**
2. Click **Test** and type something completely unrelated — like `What is the best pizza topping?`
3. Confirm the fallback message appears

> **What you just learned:** A clear fallback message stops users hitting a dead end. Always tell them what to do next when the agent cannot help.

---

## Lab 7 — Publish to Microsoft Teams

**Goal:** Make your agent available inside Microsoft Teams so your colleagues can use it.

### Step 1 — Publish the Agent

1. Click the **Publish** button at the top right of the agent editor
2. A confirmation dialog will appear — click **Publish** to confirm
3. Wait 30–60 seconds for publishing to complete
4. You will see a success message

> **Remember:** You must publish every time you make changes. Changes only go live after publishing.

### Step 2 — Go to Channels

1. Inside your agent, click **Channels** in the left panel or top tabs
2. You will see the available channels listed

### Step 3 — Connect to Teams

1. Click **Microsoft Teams**
2. Click **Add to Teams** (or **Turn on Teams** depending on what you see)
3. Copilot Studio automatically creates a Teams app — no developer steps needed
4. You will see two options:

| Option | What it means |
|--------|--------------|
| **Open in Teams** | Opens Teams immediately with your agent as a personal app |
| **Copy link** | Gives you a shareable link for colleagues |

### Step 4 — Test in Teams

1. Click **Open in Teams**
2. Microsoft Teams opens with your agent as a chat
3. Type `Hello` — confirm the greeting appears
4. Ask a question your knowledge source covers
5. Type `I want to book leave` — go through the full topic flow

> **What you just learned:** Publishing to Teams takes one click. Your agent is now live and accessible to anyone you share the link with.

---

## Lab 8 — View Analytics

**Goal:** Learn how to monitor your agent after it goes live.

### Step 1 — Open Analytics

1. Inside your agent, click **Analytics** in the left panel or top tabs
2. The analytics dashboard loads

> **Note:** If you have just published, the dashboard may show no data yet. Analytics populate after real conversations happen.

### Step 2 — Key Metrics to Watch

| Metric | What it tells you |
|--------|------------------|
| **Total sessions** | How many conversations the agent has had |
| **Engagement rate** | How many users continued after the first message |
| **Resolution rate** | How often conversations ended with the question answered |
| **Escalation rate** | How often the agent could not help |
| **Top topics** | Which topics users trigger most often |
| **Unrecognised phrases** | What users asked that the agent could not understand |

### Step 3 — Improve Using Unrecognised Phrases

1. Click **Unrecognised phrases**
2. Review what users asked that the agent could not handle
3. For each entry, decide:
   - Add it as a trigger phrase to an existing topic?
   - Create a brand new topic for it?
   - Add a new knowledge source to cover it?

> **What you just learned:** Analytics shows exactly where your agent is failing. Check it weekly and make small improvements — this is how a good bot becomes a great one.

---

## Capstone — Build a Complete IT Helpdesk Bot

**Goal:** Apply everything from this workshop to build a second, more complete agent from scratch.

### The Brief

Build an **IT Helpdesk Bot** with these requirements:

- Answers common IT questions (password resets, VPN, software access, account lockouts)
- Has a guided topic for raising a support ticket
- Uses at least one knowledge source
- Has a professional greeting and a clear fallback message
- Is published to at least one channel

### Step 1 — Write Your Instructions First

Before creating anything, plan what your bot should do:

```
You are an IT Helpdesk assistant. Help employees with common IT issues.

You can answer questions about:
- Password resets and account lockouts
- VPN setup and connection issues
- Software installation and access requests
- Hardware faults and replacements

If you cannot solve the issue, direct the user to raise a support ticket.
Always be clear, calm, and professional.
```

### Step 2 — Create the Agent

1. From the home screen, scroll to **Start building from scratch**
2. Click the **Agent** card
3. Fill in the Name and Instructions using your text above
4. Click **Create**

### Step 3 — Add Knowledge

Add at least one IT-related knowledge source:
- Your company's IT support intranet page
- A public Microsoft support page (e.g., `https://support.microsoft.com`)
- An IT policy or procedures document you upload as a file

### Step 4 — Build the Raise a Support Ticket Topic

Create a new topic called `Raise a Support Ticket` with:

1. **Trigger phrases:** `I need help`, `raise a ticket`, `log an issue`, `report a problem`, `something is broken`
2. **Message:** `I will help you log a support ticket. I just need a few details.`
3. **Question 1:** `What type of issue is this?` — give options: Hardware / Software / Access / Other — save as `IssueType`
4. **Question 2:** `Please describe the issue in a sentence.` — save as `IssueDescription`
5. **Question 3:** `What is your employee email address?` — save as `UserEmail`
6. **Confirmation message:**
```
Thank you! Your ticket has been logged.

Issue type: {IssueType}
Description: {IssueDescription}
Your email: {UserEmail}

The IT team will contact you within 4 business hours.
```

### Step 5 — Update System Topics

Edit both **Greeting** and **Fallback** topics to match the IT Helpdesk context.

### Step 6 — Publish and Test

1. Click **Publish** and confirm
2. Connect to Teams or copy a web chat link from **Channels**
3. Test all three scenarios:
   - General IT question → verify knowledge answer
   - `raise a ticket` → go through the full topic flow
   - Unrelated question → verify fallback message

### Step 7 — Capstone Checklist

```
CAPSTONE CHECKLIST

[ ] Agent has a clear name and description
[ ] Instructions are specific about what the bot covers
[ ] At least one knowledge source added and status shows Ready
[ ] Agent answers knowledge questions correctly in the test panel
[ ] Raise a Support Ticket topic works from start to finish
[ ] Greeting message lists what the bot can help with
[ ] Fallback message tells users what to do next
[ ] Agent is published
[ ] Agent connected to at least one channel (Teams or web chat)
[ ] Opened Analytics after testing
[ ] Found at least one unrecognised phrase to improve
```

---

## Quick Reference Card

```
COPILOT STUDIO — QUICK REFERENCE (Current UI)

HOME SCREEN
  URL             → copilotstudio.microsoft.com
  Build an agent  → Agent card under "Start building from scratch"
  Build a flow    → Create workflow card
  Top right       → Environment selector (use Default Environment)

LEFT NAVIGATION
  Home    → "What would you like to build?" screen
  Agents  → All your agents (bots)
  Flows   → All your workflows / automations
  Tools   → Connectors, plugins, tools for agents

INSIDE AN AGENT (left panel or tabs)
  Knowledge  → Add documents, websites, SharePoint pages
  Topics     → Build conversation flows (guided tasks)
  Actions    → Connect to external apps
  Channels   → Publish to Teams, website, Slack, etc.
  Analytics  → Usage data, top topics, unrecognised phrases

CREATING AN AGENT
  Home → Agent card → Fill in Name + Instructions → Create
  Then: Add Knowledge → wait for Ready → Test → Publish → Channels

BUILDING A TOPIC
  Topics → Add a topic → From blank
  → Name it
  → Add trigger phrases in the Trigger node (at least 5 phrases)
  → Add nodes: Send a message / Ask a question / Condition
  → Use the {} variable picker to insert saved answers
  → Save → Test

PUBLISHING
  Publish button (top right) → Confirm → Channels → Teams or web chat

KNOWLEDGE SOURCES
  Public website → paste a public URL
  Files          → upload PDF or Word document
  SharePoint     → requires work/school Microsoft account
  Dataverse      → structured business data

GOOD INSTRUCTIONS FORMULA
  1. Who the bot is (role + name)
  2. What topics it covers
  3. What to say when it does not know
  4. Tone (professional, friendly, brief)
```

---

## Troubleshooting

**I cannot see the "What would you like to build?" screen**
→ Make sure you are signed in. Go directly to [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com). Complete the free trial registration if prompted.

**I cannot find the Agent card**
→ Scroll down to **Start building from scratch** on the home screen. The Agent card is the middle one. Refresh the page if needed.

**Agent gives generic answers instead of using my knowledge**
→ The knowledge source status must show **Ready** before testing. If it shows Failed, delete and re-add the source. Always wait at least 2 minutes after adding.

**Topic does not trigger when I type the phrase**
→ Add more trigger phrase variations — at least 5 different ways a user might ask. Use full sentences, not single words.

**Variable shows as plain text `{LeaveStartDate}` in the chat**
→ Use the **variable picker** (the `{}` button inside the message node editor) — do not type curly braces by hand.

**Agent loops or keeps repeating the same question**
→ Check the topic canvas for nodes with no outgoing connection. Every node must link to the next step or end the conversation.

**Published agent not showing in Teams**
→ You must click Publish first. After publishing, wait 60 seconds and reload Teams. Check the Channels section and confirm Teams is enabled.

**Cannot add SharePoint as a knowledge source**
→ SharePoint requires a work or school Microsoft account. Personal accounts cannot access SharePoint. Use a file upload or public URL instead.

**Free trial expired**
→ The free trial lasts 30 days. After that, a Microsoft 365 licence that includes Copilot Studio is needed. Contact your IT admin to check your licence.

---

*Microsoft Copilot Studio Workshop Lab Guide*
*Updated to match current UI — For training use*
*Visit [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com) to get started*
