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
- [ ] You can see the **Home** screen with a **Create** button
- [ ] Optional: Have a document (PDF or Word file) or a website URL ready to use as knowledge

> **Free Trial Note:** Copilot Studio offers a free trial. You do not need a paid licence to complete this lab. If prompted, start the free trial when you sign in.

---

## What You Will Build

By the end of this lab you will have built and published a working **HR FAQ Bot** — an AI chatbot that:

- Answers employee questions about leave, benefits, and company policy
- Reads answers from your own documents or websites
- Handles guided conversations using Topics
- Is published and accessible inside Microsoft Teams

---

## Key Concepts — Read This First

Before starting, understand these four terms. They come up throughout the lab.

| Term | What it means |
|------|--------------|
| **Agent (Bot)** | The AI chatbot you build. It has a name, a purpose, and a personality. |
| **Knowledge** | Documents, websites, or SharePoint pages the bot reads to find answers. |
| **Topic** | A scripted conversation flow. When a user says X, the bot asks Y, then does Z. |
| **Action** | Something the bot does automatically — send an email, create a task, fill a form. |

---

## Lab 1 — Sign In and Explore the Interface

**Goal:** Get comfortable with the Copilot Studio interface before building anything.

### Step 1 — Sign In

1. Go to [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com)
2. Click **Sign in** and use your Microsoft account
3. If prompted to start a free trial, click **Start free trial**
4. Wait for the Home screen to load

### Step 2 — Explore the Home Screen

Take 5 minutes to find these areas:

- **Create button** — top left, this is where you make a new bot
- **My agents** — lists all bots you have built
- **Learn section** — guided tutorials from Microsoft
- **Environment selector** — top right dropdown, shows which Microsoft environment you are in

### Step 3 — Understand the Left Navigation

Once inside a bot, the left panel has:

| Section | What it does |
|---------|-------------|
| **Overview** | Summary of your bot — name, description, channels |
| **Knowledge** | Where you add documents and websites |
| **Topics** | Where you build conversation flows |
| **Actions** | Where you connect to other apps |
| **Channels** | Where you publish — Teams, websites, etc. |
| **Analytics** | See how many people use the bot and what they ask |

> **What you just learned:** Copilot Studio organises everything into Knowledge, Topics, Actions, and Channels. Most of what you build happens in Knowledge and Topics.

---

## Lab 2 — Create Your First Bot

**Goal:** Create a new bot with a name and a clear purpose in under 5 minutes.

### Step 1 — Start a New Agent

1. Click **Create** in the left sidebar
2. Select **New agent**
3. You will see a setup screen with two options — choose **Skip to configure** (for this lab)

### Step 2 — Name and Describe Your Bot

Fill in the following fields:

| Field | What to type |
|-------|-------------|
| **Name** | `HR FAQ Bot` |
| **Description** | `Answers employee questions about leave, benefits, payroll, and company policy.` |
| **Instructions** | `You are a helpful HR assistant. Only answer questions using the knowledge provided. If you do not know the answer, say: Please contact the HR team directly.` |

### Step 3 — Set the Language

1. Under **Language**, select **English**
2. Click **Create**

Your bot is now created. You will land on the bot's **Overview** page.

### Step 4 — Test the Empty Bot

1. Click **Test** (top right button) to open the test chat panel
2. Type: `What is the leave policy?`
3. The bot will say it does not have enough information yet — this is expected
4. Close the test panel

> **What you just learned:** A bot with no Knowledge gives generic or empty answers. The next lab fixes this by adding your content.

---

## Lab 3 — Add Knowledge

**Goal:** Give your bot a document or website to read answers from.

### Step 1 — Go to the Knowledge Section

1. In the left panel, click **Knowledge**
2. Click **Add knowledge**

### Step 2 — Choose a Knowledge Source

You will see several options. Choose the one that matches what you have available:

| Option | When to use |
|--------|-------------|
| **Public website** | You have a URL to a public webpage (easiest for this lab) |
| **Files** | You have a PDF, Word, or text document to upload |
| **SharePoint** | You have content in SharePoint (requires work account) |
| **Dataverse** | You have structured data in Microsoft Dataverse |

**For this lab — use one of these:**

**Option A — Public website:**
1. Select **Public website**
2. Paste this example URL: `https://www.acas.org.uk/holiday-entitlement` (or any HR-related public page)
3. Click **Add**

**Option B — Upload a file:**
1. Select **Files**
2. Click **Upload file**
3. Upload any PDF or Word document — even a simple one-page leave policy document
4. Click **Add**

### Step 3 — Wait for Indexing

Copilot Studio will read and index your content. This takes about 1–2 minutes. You will see a **Ready** status when it is done.

### Step 4 — Test With Knowledge

1. Click **Test** (top right)
2. Ask the bot a question that your document or webpage answers
3. The bot should now give a real answer — and show which source it used

> **What you just learned:** Knowledge is the most important part of a helpful bot. The bot only answers from what you add here — it will not guess or make things up.

---

## Lab 4 — Build a Topic (Guided Conversation)

**Goal:** Create a scripted conversation flow for a specific request — booking annual leave.

Knowledge handles open questions. Topics handle step-by-step guided tasks.

### Step 1 — Go to Topics

1. In the left panel, click **Topics**
2. You will see some system topics already listed (Greeting, Goodbye, etc.)
3. Click **Add a topic** → **From blank**

### Step 2 — Name the Topic

1. At the top, replace **Untitled** with: `Book Annual Leave`
2. Press Enter to save the name

### Step 3 — Add Trigger Phrases

Trigger phrases are what a user says to start this topic.

1. Click inside the **Trigger** node at the top of the canvas
2. Add these phrases one by one (press Enter after each):
   - `I want to book leave`
   - `How do I request annual leave`
   - `Book time off`
   - `Request holiday`
   - `I need to take leave`

### Step 4 — Add a Message Node

1. Click the **+** button below the Trigger node
2. Select **Send a message**
3. Type: `I can help you book annual leave. I just need a few details.`

### Step 5 — Ask for Information

1. Click **+** → **Ask a question**
2. In the question box, type: `What is your start date for the leave?`
3. Under **Identify**, select **Date**
4. Under **Save response as**, name the variable: `LeaveStartDate`

Repeat to add a second question:
1. Click **+** → **Ask a question**
2. Type: `And what is your end date?`
3. Under **Identify**, select **Date**
4. Name the variable: `LeaveEndDate`

### Step 6 — Add a Confirmation Message

1. Click **+** → **Send a message**
2. Type the following (use the variable picker to insert variables):

```
Thank you! Your leave request has been noted:
Start date: {LeaveStartDate}
End date: {LeaveEndDate}

Your manager will receive a notification to approve this. You will hear back within 2 business days.
```

### Step 7 — Save and Test

1. Click **Save** (top right)
2. Click **Test**
3. Type: `I want to book leave`
4. Follow the conversation — answer the date questions
5. Confirm the bot gives a proper summary at the end

> **What you just learned:** Topics let you collect information step by step. You can use the collected data in messages, emails, or further actions.

---

## Lab 5 — Configure the Greeting Topic

**Goal:** Personalise the bot's greeting so it feels professional and helpful.

### Step 1 — Find the Greeting Topic

1. Click **Topics** in the left panel
2. Find **Greeting** in the System topics list
3. Click on it to open it

### Step 2 — Edit the Greeting Message

1. Find the **Send a message** node
2. Clear the default text
3. Replace it with:

```
Hello! I am the HR FAQ Bot.

I can help you with:
- Leave and holiday questions
- Benefits and payroll queries
- Company policy information
- Booking annual leave

Just type your question and I will do my best to help. For urgent issues, please contact hr@yourcompany.com
```

### Step 3 — Save

1. Click **Save**
2. Click **Test** and type `Hello` — confirm the new greeting appears

---

## Lab 6 — Add a Fallback Response

**Goal:** Make sure the bot handles questions it cannot answer gracefully.

### Step 1 — Find the Fallback Topic

1. In **Topics**, look for **Fallback** in the System topics list
2. Click to open it

### Step 2 — Edit the Fallback Message

Replace the default text with:

```
I'm sorry, I don't have the answer to that question yet.

Here is what you can do:
- Try rephrasing your question
- Email the HR team at hr@yourcompany.com
- Visit the HR portal at [your intranet link]

Is there anything else I can help you with?
```

### Step 3 — Save and Test

1. Click **Save**
2. Click **Test** and ask something completely unrelated — like `What is the weather today?`
3. Confirm the fallback message appears

> **What you just learned:** A good fallback message is essential. It stops users from hitting a dead end and tells them what to do next.

---

## Lab 7 — Publish to Microsoft Teams

**Goal:** Make your bot available in Microsoft Teams so colleagues can use it.

### Step 1 — Publish the Bot

1. Click **Publish** (top right button)
2. A confirmation dialog appears — click **Publish** again
3. Wait for the publishing process to complete (about 30 seconds)
4. You will see a green confirmation message

> **Important:** You must publish before the bot is available on any channel. Every time you make changes, you need to publish again for them to go live.

### Step 2 — Go to Channels

1. In the left panel, click **Channels**
2. You will see a list of available channels:
   - Microsoft Teams
   - Custom website
   - Facebook
   - Slack
   - And more

### Step 3 — Add the Teams Channel

1. Click **Microsoft Teams**
2. Click **Add to Teams**
3. Copilot Studio generates a Teams app automatically
4. You will see two options:

| Option | What it means |
|--------|--------------|
| **Open in Teams** | Opens Teams and adds the bot as a personal app right now |
| **Copy link** | Gets a link you can share with colleagues |

### Step 4 — Test in Teams

1. Click **Open in Teams**
2. Teams opens with your bot as a chat
3. Type `Hello` — confirm the greeting appears
4. Ask a question that your knowledge source covers
5. Ask `I want to book leave` — go through the topic flow

> **What you just learned:** Publishing to Teams takes one click. Your bot is now accessible to anyone in your organisation who has the link or the Teams app installed.

---

## Lab 8 — View Analytics

**Goal:** Understand how to monitor your bot after it goes live.

### Step 1 — Open Analytics

1. In the left panel, click **Analytics**
2. You will see the analytics dashboard

### Step 2 — Key Metrics to Watch

| Metric | What it tells you |
|--------|------------------|
| **Total sessions** | How many conversations the bot had |
| **Engagement rate** | How many users kept talking after the first message |
| **Resolution rate** | How many conversations ended with the user's question answered |
| **Escalation rate** | How often the bot handed off to a human |
| **Top topics** | Which topics users trigger most often |
| **Unrecognised phrases** | Things users asked that the bot did not understand |

### Step 3 — Use Unrecognised Phrases to Improve

The **Unrecognised phrases** section is the most useful for improvement:

1. Click **Unrecognised phrases** in the analytics panel
2. Review what users are asking that the bot cannot handle
3. Use these to:
   - Add new trigger phrases to existing topics
   - Create new topics for common questions
   - Add more knowledge sources

> **What you just learned:** Analytics shows you exactly where your bot is failing. Check it weekly and keep improving.

---

## Capstone — Build a Complete IT Helpdesk Bot

**Goal:** Apply everything you have learned to build a second, more complete bot from scratch.

### The Brief

Build an **IT Helpdesk Bot** with these features:

- Answers common IT questions (password reset, VPN, software access)
- Has a guided topic for raising a support ticket
- Uses a knowledge source (your IT policy page or a document you upload)
- Has a professional greeting and fallback message
- Is published to at least one channel

### Step 1 — Write Your Bot Description

Before creating anything, write a clear description of your bot:

```
Name: IT Helpdesk Bot

Purpose: Help employees with common IT issues and requests.

It should answer questions about:
- Password resets
- VPN access and setup
- Software installation and access requests
- Hardware issues and replacements
- Account lockouts

For issues it cannot solve, it should direct users to raise a ticket.
```

### Step 2 — Create the Bot

1. Go to **Create** → **New agent**
2. Use your written description above as the bot's instructions
3. Be specific — a detailed instruction gives better answers

### Step 3 — Add Knowledge

Add at least one knowledge source:

- Your company's IT support page
- An IT policy document
- A public IT help page (e.g., Microsoft support)

### Step 4 — Build a Support Ticket Topic

Create a topic called `Raise a Support Ticket` that:

1. Triggers on phrases like: `I need help`, `raise a ticket`, `log an issue`
2. Asks: `What type of issue are you experiencing?` (give options: Hardware / Software / Access / Other)
3. Asks: `Please describe the issue briefly`
4. Asks: `What is your employee email address?`
5. Sends a confirmation message with all details

### Step 5 — Edit System Topics

Update the Greeting and Fallback topics to match the IT Helpdesk context.

### Step 6 — Publish and Test

1. Publish the bot
2. Add it to Teams or copy a web chat link
3. Test all scenarios:
   - Ask a general IT question
   - Go through the support ticket flow
   - Ask something unrelated (test the fallback)

### Step 7 — Self-Evaluation Checklist

```
CAPSTONE CHECKLIST

[ ] Bot has a clear name and description
[ ] At least one knowledge source added and indexed
[ ] Bot answers knowledge questions correctly in the test panel
[ ] Raise a Support Ticket topic works end to end
[ ] Greeting message is professional and lists what the bot can help with
[ ] Fallback message tells users what to do next
[ ] Bot is published to at least one channel
[ ] I checked Analytics after testing
[ ] I identified at least one improvement from Unrecognised phrases
```

---

## Quick Reference Card

```
COPILOT STUDIO — LAB QUICK REFERENCE

GETTING STARTED
  Sign in      → copilotstudio.microsoft.com
  Create bot   → Create → New agent → Skip to configure
  Test bot     → Click Test button (top right of any screen)
  Publish      → Click Publish button → confirm

KEY SECTIONS (left panel inside a bot)
  Knowledge  → Add documents, websites, SharePoint pages
  Topics     → Build conversation flows and guided tasks
  Actions    → Connect to external apps and services
  Channels   → Publish to Teams, website, Slack, etc.
  Analytics  → See usage, top topics, unrecognised phrases

BUILDING A TOPIC
  1. Topics → Add a topic → From blank
  2. Name the topic
  3. Add trigger phrases (what users say to start it)
  4. Add nodes: Send a message / Ask a question / Condition
  5. Use variables to store user answers
  6. Save → Test

PUBLISHING
  Publish button → Publish → Channels → Pick channel → Add

KNOWLEDGE SOURCES
  Public website  → paste any public URL
  Files           → upload PDF, Word, or text file
  SharePoint      → connect to SharePoint page or library
  Dataverse       → connect to structured data

GOOD BOT INSTRUCTIONS FORMULA
  1. Who the bot is (role and name)
  2. What topics it should answer
  3. What to say when it does not know the answer
  4. Tone — formal, friendly, professional

IMPROVING YOUR BOT
  → Check Analytics → Unrecognised phrases every week
  → Add new trigger phrases to existing topics
  → Add more knowledge sources for gaps
  → Edit fallback message to guide users better
```

---

## Troubleshooting

**Bot gives a generic answer instead of using my knowledge**
→ Check the knowledge source status — it should say **Ready**, not **Indexing** or **Failed**. If failed, delete and re-add it. Wait 2 minutes after adding before testing.

**Topic does not trigger when I type the phrase**
→ Add more trigger phrase variations. Use natural language. Avoid very short phrases like just `leave` — use full sentences like `I want to book leave`.

**Bot loops or repeats the same message**
→ Check your topic flow for missing connections between nodes. Every node must have a path to the next step or an End conversation node.

**Published bot not working in Teams**
→ You must click Publish before Teams picks up changes. Publishing takes 30–60 seconds. Reload Teams after publishing.

**Variable shows as {VariableName} in the message**
→ Use the variable picker (the curly brace icon) inside the message node instead of typing the variable name manually.

**Cannot add a SharePoint knowledge source**
→ SharePoint knowledge requires a work or school Microsoft account. Personal accounts cannot access SharePoint. Use a file upload or public URL instead.

**Free trial expired**
→ The Copilot Studio trial lasts 30 days. After that, a Microsoft 365 licence with Copilot Studio included is required for production use.

---

*Microsoft Copilot Studio Workshop Lab Guide*
*Beginner Edition — For training use*
*Visit [copilotstudio.microsoft.com](https://copilotstudio.microsoft.com) to get started*
