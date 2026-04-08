# n8n-gmail-ai-agent

# AI Email Summarizer + Auto Reply Agent

> An intelligent email automation agent built with **n8n** and **Google Gemini AI** that automatically reads, classifies, and replies to emails — with zero human involvement.

---

## What It Does

- Monitors your Gmail inbox automatically every few minutes
- Classifies every incoming email as `urgent`, `complaint`, `request`, `inquiry`, or `normal` using Gemini AI
- Filters out no-reply and automated emails to prevent reply loops
- Generates personalized, context-aware replies using the full email content
- Sends different reply tones — empathetic for urgent, professional for normal
- Logs every email, its classification, and the exact reply sent to Google Sheets

---

## Workflow Architecture

<img width="1307" height="449" alt="Image" src="https://github.com/user-attachments/assets/adcdb075-bf45-4e08-9dda-192a7c264d8d" />
---

## Full Workflow Logic

```
Gmail Trigger
      ↓
AI Agent (classify intent using Gemini)
      ↓
Set Node (extract and store email data cleanly)
      ↓
Check Sender — IF noreply/donotreply → STOP
      ↓
Check Urgent — IF urgent or complaint
      ↓ true                    ↓ false
AI Agent1                   AI Agent2
(Urgent prompt)             (Normal prompt)
      ↓                         ↓
Gmail Reply (urgent)      Gmail Reply (normal)
      ↓                         ↓
      └──────── merge ──────────┘
                    ↓
          Google Sheets (log everything)
```

---

## Tech Stack

| Tool | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow automation engine |
| [Google Gemini 2.5 Flash](https://aistudio.google.com) | AI classification + reply generation |
| Gmail API | Trigger on new emails + send replies |
| Google Sheets | Log all emails, intents and replies |

---

## How Each Node Works

### 1. Gmail Trigger
Watches your inbox and fires the workflow the moment a new email arrives. Polls every 1–5 minutes automatically.

### 2. AI Agent — Classify Intent
Reads the email subject and body and outputs exactly one word:
`urgent` / `complaint` / `request` / `inquiry` / `normal`

**Prompt used:**
```
You are an email classifier.
Read this email and reply with ONLY one word from this list:
urgent, complaint, request, inquiry, normal
No explanation. No punctuation. Just the single word.

Email Subject: {{ $json.subject }}
Email Body: {{ $json.snippet }}
```

### 3. Set Node — Extract Intent
Saves all email data into clean variables for downstream nodes:
- `intent` — the classified category
- `sender` — who sent the email
- `subject` — email subject line
- `body` — email body content
- `emailId` — used to reply in the same thread

### 4. Check Sender — IF Node (No-reply Filter)
Checks if the sender address contains `noreply`, `no-reply`, `donotreply`, or `notifications`.
- If yes → workflow stops. No reply is sent.
- If no → continues to urgency check.

This prevents the agent from getting into automated reply loops.

### 5. Check Urgent — IF Node
Checks if the intent is `urgent` or `complaint`.
- True branch → Urgent AI Agent
- False branch → Normal AI Agent

### 6a. AI Agent1 — Urgent Reply
Generates an empathetic, high-priority reply that directly addresses the person's specific problem.

**Prompt used:**
```
You are a professional email assistant handling an urgent or complaint email.
Write a specific, empathetic reply that directly addresses this person's exact problem.
Acknowledge the issue. Promise quick follow-up. Keep it under 80 words.

Classification: {{ $json.Intent }}
From: {{ $json.Sender }}
Subject: {{ $json.Subject }}
Original email: {{ $json.Body }}
```

### 6b. AI Agent2 — Normal Reply
Generates a friendly, professional reply tailored to the specific inquiry or request.

**Prompt used:**
```
You are a professional email assistant.
Write a friendly, formal reply that directly addresses what this person is asking about.
Keep it under 60 words. Be warm but concise.

Classification: {{ $json.Intent }}
From: {{ $json.Sender }}
Subject: {{ $json.Subject }}
Original email: {{ $json.Body }}
```

### 7. Gmail Reply
Sends the AI-generated reply back to the sender in the same email thread — not as a new email.

### 8. Google Sheets — Log Everything
Appends one row per email processed with these columns:

| Column | What is stored |
|---|---|
| Timestamp | When the email was received |
| Sender | Who sent the email |
| Subject | Email subject line |
| Body | Original email content |
| Intent | AI classification result |
| Reply Sent | The exact reply the AI generated and sent |

---

## Setup Instructions

### Prerequisites
- n8n account (free 14-day cloud trial at [n8n.io](https://n8n.io) or self-hosted free)
- Google account (Gmail + Google Sheets)
- Gemini API key (free at [aistudio.google.com](https://aistudio.google.com))

### Step 1 — Get your Gemini API key
1. Go to [aistudio.google.com](https://aistudio.google.com)
2. Sign in with your Google account
3. Click **Get API key** → **Create API key**
4. Copy and save the key

### Step 2 — Use N8N
1. Open your n8n dashboard
2. Click **New Workflow**
3. Start Connecting Nodes

### Step 3 — Connect your accounts
- **Gmail** — connect via OAuth when prompted inside the Gmail nodes
- **Google Sheets** — connect via OAuth when prompted inside the Sheets node
- **Gemini** — paste your API key into the Google Gemini Chat Model node credentials

### Step 4 — Set up your Google Sheet
Create a new Google Sheet with these exact headers in Row 1:
```
Timestamp | Sender | Subject | Body | Intent | Reply Sent
```

### Step 5 — Activate the workflow
1. Click **Save**
2. Toggle the workflow to **Active** (top right switch turns green)
3. Done — the agent now runs automatically 24/7

---

## Key Design Decisions

**Why classify before filtering?**
Every email including no-reply ones gets classified and would be logged — giving a complete audit trail of all incoming mail before any filtering happens.

**Why filter noreply emails?**
Without this filter, the agent would reply to automated emails from Amazon, GitHub, notifications etc — causing infinite reply loops.

**Why two separate AI Agents for urgent and normal?**
Each agent has a completely different prompt and tone. Urgent emails get an empathetic, high-priority response. Normal emails get a calm, professional one. Using one agent for both would produce generic replies.

**Why pass the full email body to the reply agent?**
If you only pass the intent word (`urgent`) to the AI, it generates the same generic reply for every urgent email. Passing the full body ensures every reply is specific to that person's exact problem.

**Why is Google Sheets at the very end?**
Only at the end does the workflow have all the data — original email, classification, AND the actual reply sent. Placing Sheets earlier would mean the reply column is always empty.

---

## Example Google Sheet Output

| Timestamp | Sender | Subject | Body | Intent | Reply Sent |
|---|---|---|---|---|---|
| 2026-04-09 09:14 | ravi@gmail.com | Account hacked! | My account has been compromised... | urgent | Dear Ravi, we understand this is urgent. Our team has been notified... |
| 2026-04-09 09:31 | priya@work.com | Meeting tomorrow? | Can we schedule a meeting at 3pm... | request | Hi Priya, tomorrow at 3pm works great. I'll send a calendar invite... |
| 2026-04-09 10:02 | sam@client.com | Worst service ever | I have been waiting 2 weeks for a refund... | complaint | Dear Sam, we sincerely apologise. Your refund has been escalated... |
| 2026-04-09 10:45 | john@company.com | Pricing info | Could you share your latest pricing plans... | inquiry | Hi John, our Pro plan starts at $29/month. Happy to walk you through... |

---

## Handling Rate Limits

The free Gemini API allows 15 requests per minute. Since this workflow makes multiple Gemini calls per email, add a **Wait node** (5 seconds) between each AI Agent step to avoid hitting rate limits.

Alternatively switch to `gemini-2.0-flash-lite` in the Gemini Chat Model node for higher free tier limits.

---

## Future Improvements

- [ ] Add confidence threshold — if AI is unsure, flag for human review instead of auto-replying
- [ ] Add WhatsApp notification for urgent emails
- [ ] Support multiple language replies
- [ ] Add sentiment score column to Google Sheets
- [ ] Switch to webhook trigger for real-time response instead of polling

---

## Author

- Built by **Junaid Shariff**
- Connect with me on [LinkedIn](https://www.linkedin.com/in/junaid-shariff10/)

---

