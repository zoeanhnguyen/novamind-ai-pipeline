# NovaMind AI Content Pipeline

An automated marketing pipeline that generates, distributes, and analyzes blog and newsletter content using AI — built with n8n, Mistral AI, and Google Sheets.

---

## Overview

NovaMind is a fictional early-stage AI startup helping creative agencies automate their workflows. This pipeline takes a blog topic as input and fully automates content creation, audience segmentation, campaign logging, performance analysis, and next-topic recommendations — with zero manual steps after triggering.

---

## Architecture

```
[Webhook Trigger]
       ↓
[Edit Fields]          — normalize input, generate campaign_id + timestamp
       ↓
[Basic LLM Chain]      — Mistral AI generates blog + 3 persona newsletters (JSON)
       ↓
[Code: Parse JSON]     — extract and structure AI output
       ↓
[Google Sheets #1]     — log content to "Content" sheet
       ↓
[Code: Simulate Metrics] — generate mock open/click/unsubscribe rates per persona
       ↓
[Google Sheets #2]     — log analytics to "Analytics" sheet
       ↓
[Wait]                 — rate limit buffer
       ↓
[Performance Summary]  — Mistral AI writes plain-text campaign insight
       ↓
[Aggregate]            — combine 3 persona metrics into single item
       ↓
[Wait]                 — rate limit buffer
       ↓
[Prepare Summary Input] — Mistral AI suggests 3 next blog topics based on metrics
       ↓
[Wait]                 — rate limit buffer
       ↓
[Suggestions: Parse]   — extract topic suggestions from AI output
       ↓
[Respond to Webhook]   — return HTML dashboard with all results
```

### Flow Diagram

```
Webhook → Edit Fields → Basic LLM Chain → Parse JSON → Sheets (Content)
                                                              ↓
                                               Simulate Metrics → Sheets (Analytics)
                                                              ↓
                                                         Wait → Performance Summary
                                                              ↓
                                                    Aggregate → Wait → Topic Suggestions
                                                              ↓
                                                    Parse → Respond to Webhook (HTML)
```

---

## Tools & Technologies

| Component | Tool |
|---|---|
| Workflow automation | [n8n](https://n8n.io) (self-hosted, local) |
| AI model | Mistral AI — `mistral-small-latest` via LangChain Basic LLM Chain |
| Content storage | Google Sheets (2 tabs: Content, Analytics) |
| Trigger interface | HTTP Webhook (GET/POST) |
| Dashboard | HTML response from Respond to Webhook node |
| Runtime | Node.js 20.x |

---

## Features

### Core Requirements
- **AI Content Generation** — single LLM call produces a full blog draft (800–1000 words with SEO metadata) + 3 persona-targeted newsletter versions in one structured JSON response
- **Persona Segmentation** — 3 audience types:
  - *Creative Director* — focuses on team efficiency, creative vision, ROI
  - *Solo Freelancer* — focuses on saving time, earning more, freelance tools
  - *Agency Manager* — focuses on scaling operations, client delivery, margins
- **CRM Simulation** — Google Sheets acts as mock CRM; each campaign is logged with title, newsletter content, campaign ID, and send timestamp
- **Performance Logging** — simulated open rate (20–50%), click rate (2–12%), unsubscribe rate (0–1%), and emails sent per persona, stored historically in Analytics sheet
- **AI Performance Summary** — Mistral analyzes the metrics and writes a 3–4 sentence plain-text insight identifying top-performing persona and recommending improvements

### Bonus Features
- **AI Topic Suggestions** — after each campaign, Mistral suggests 3 next blog topics based on which persona performed best and why
- **Live Dashboard UI** — Respond to Webhook returns a styled HTML page showing blog title, distribution status, AI insight, and topic suggestions

---

## Assumptions & Design Decisions

- **Google Sheets as CRM**: HubSpot free tier requires verified domain for email sending. Google Sheets was used as a lightweight, zero-cost CRM substitute that demonstrates the same data structure (contact logging, campaign tracking, segmentation by persona). The payload structure mirrors what would be sent to HubSpot's Contacts and Timeline APIs.
- **Simulated metrics**: Newsletter performance data (open rate, click rate, etc.) is randomly generated within realistic ranges. In production, these would be fetched from HubSpot, Mailchimp, or similar via API.
- **Mistral AI**: Used instead of OpenAI due to free tier availability. The model (`mistral-small-latest`) is capable of producing structured JSON output reliably when prompted correctly.
- **Wait nodes**: Added between LLM calls to prevent 503 rate limit errors from Mistral's free tier. Each wait is 3–5 seconds.
- **Blog length**: Prompt targets 800–1000 words to stay within JSON token limits while still producing substantive content.
- **"Preview Only" button**: The generated blog URL (`www.novamind.com/blog/...`) is a mock slug. No live site exists — the button is intentionally disabled to reflect this.

---

## Project Structure

```
novamind-ai-pipeline/
├── workflow/
│   └── novamind-workflow.json     ← import this into n8n
├── ui/
│   └── trigger.html               ← optional HTML trigger form
└── README.md
```

---

## How to Run Locally

### Prerequisites

- Node.js v20+ ([nodejs.org](https://nodejs.org))
- n8n installed globally
- Mistral AI API key ([console.mistral.ai](https://console.mistral.ai) — free tier)
- Google account with Google Sheets access

### Step 1 — Install and start n8n

```bash
npm install -g n8n
n8n start
```

Open [http://localhost:5678](http://localhost:5678) in your browser.

### Step 2 — Set up credentials

In n8n → **Settings → Credentials**:

1. Add **Mistral Cloud** credential with your API key
2. Add **Google Sheets OAuth2** credential (connect your Google account)

### Step 3 — Set up Google Sheets

Create a new Google Spreadsheet with two tabs:

**Tab 1 — "sheet" (Content)**
```
Title | Meta Description | URL | Draft | Newsletter Creative Director | Newsletter Solo Freelancer | Newsletter Agency Manager | Timestamp
```

**Tab 2 — "Analytics"**
```
Campaign_ID | Persona | Open_Rate | Click_Rate | Unsubscribe_Rate | Emails_Sent | Timestamp
```

### Step 4 — Import workflow

In n8n → top-right menu (⋮) → **Import from file** → select `workflow/novamind-workflow.json`

Update the Google Sheets nodes to point to your spreadsheet.

### Step 5 — Trigger the pipeline

Option A — Direct URL (test mode):
```
http://localhost:5678/webhook-test/novamind-pipeline?topic=AI+in+creative+automation&author=NovaMind+Team
```

Option B — curl:
```bash
curl "http://localhost:5678/webhook-test/novamind-pipeline?topic=AI+in+creative+automation&author=NovaMind+Team"
```

Option C — Production (after activating workflow):
```
http://localhost:5678/webhook/novamind-pipeline?topic=YOUR+TOPIC&author=YOUR+NAME
```

The browser will display the live HTML dashboard with results.

### Step 6 — View results

- **Dashboard**: returned directly in browser after ~30–60 seconds
- **Content sheet**: blog title, meta, draft, 3 newsletters
- **Analytics sheet**: open/click/unsubscribe metrics per persona

---

## Google Sheets Output Example

**Content sheet:**

| Title | Meta Description | Draft | Newsletter Creative Director | ... | Timestamp |
|---|---|---|---|---|---|
| AI in Agency Workflows: 2026 Trends | How AI is reshaping... | ## Introduction\n\n... | Subject: ... Body: ... | ... | 2026-04-19T19:12:40Z |

**Analytics sheet:**

| Campaign_ID | Persona | Open_Rate | Click_Rate | Unsubscribe_Rate | Emails_Sent | Timestamp |
|---|---|---|---|---|---|---|
| camp_1776640360042 | creative_director | 38.4 | 7.2 | 0.3 | 42 | 2026-04-19T... |
| camp_1776640360042 | solo_freelancer | 28.7 | 4.1 | 0.8 | 31 | 2026-04-19T... |
| camp_1776640360042 | agency_manager | 33.1 | 5.9 | 0.2 | 55 | 2026-04-19T... |

---

## Known Limitations

- Mistral free tier has rate limits — Wait nodes (3–5s) are used to mitigate 503 errors
- All email sending and CRM contact creation is simulated via structured data logged to Google Sheets
- Pipeline runs sequentially (~30–60s total execution time) due to multiple LLM calls
- n8n must be running locally for the webhook to be accessible
