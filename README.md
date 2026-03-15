# Voice AI Lead Qualification & Appointment Booking System

An n8n automation that handles inbound leads end-to-end — from the moment a lead hits your CRM to a booked appointment on your calendar — using AI voice calls powered by Retell AI and Twilio SIP.

![System Overview](assets/system-overview.png)

---

## What It Does

When a new lead comes in (from ads, website forms, referrals), the workflow automatically:

1. Picks up the lead from Frappe CRM via webhook
2. Validates the phone number and checks for duplicates
3. Places an outbound AI voice call through Retell AI + Twilio SIP
4. Lets the AI qualify the lead by asking screening questions
5. Books an appointment on Google Calendar if the lead qualifies
6. Syncs everything back to Frappe CRM — call status, notes, and recording

The sales team only talks to pre-qualified leads who've already confirmed they're interested and picked a time slot.

---

## The Problem It Solves

Before this, the team was spending 6–8 hours a day manually calling 50–100 new leads. Most of those calls went to voicemail or were with people who weren't a fit. Follow-ups fell through the cracks, and good leads got cold.

This workflow cut manual calling work by roughly 80%. The team now focuses on closing, not dialing.

---

## Tech Stack

| Tool | Role |
|------|------|
| **n8n** | Workflow orchestration |
| **Frappe CRM** | Lead source and CRM sync |
| **Retell AI** | AI voice agent |
| **Twilio SIP** | Outbound call routing |
| **Google Calendar** | Appointment booking |
| **Google Sheets** | Error logging |

---

## Workflow Architecture

![n8n Workflow](assets/n8n-workflow.png)

The system is split into six sub-workflows, each triggered by its own webhook:

### 1. New Lead Flow
**Trigger:** `POST /webhook/new-lead`

Receives the lead from Frappe CRM, validates the phone number, checks if the lead already exists in the system, and kicks off the outbound call after a short delay.

### 2. Outbound AI Call (Retell + Twilio SIP)
**Trigger:** `POST /webhook/start-outbound-call`

Registers the call with Retell AI, builds the SIP URI, and places the call through Twilio. The AI agent takes it from there.

### 3. AI Lead Qualification (LLM)
**Trigger:** `POST /webhook/retell-check-qualification`

During the call, Retell hits this webhook to score the lead in real time. The LLM evaluates the conversation and returns a qualification decision with a recommended next action.

### 4. Available Slots
**Trigger:** `POST /webhook/retell-get-slots`

The AI agent calls this mid-conversation to check real availability on Google Calendar. Returns open slots in natural language so the AI can offer them to the lead directly on the call.

### 5. Appointment Booking & Confirmation
**Trigger:** `POST /webhook/book-appointment`

Once the lead picks a slot, this workflow creates the calendar event, updates Frappe CRM, sends a confirmation email, fires an SMS confirmation to the lead, and pings the sales team on WhatsApp.

### 6. CRM Sync & Error Handling
**Trigger:** `POST /webhook/update-crm`

Keeps Frappe CRM updated throughout the process — called after every AI call to sync status, notes, and recording URL. A global error handler catches failures across all flows and logs them to Google Sheets with an SMS alert to the admin.

---

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Frappe CRM instance with API access
- Retell AI account with a configured voice agent
- Twilio account with a SIP-enabled phone number
- Google Calendar with a service account or OAuth credentials
- Google Sheets for error logging (optional but recommended)

### Step 1 — Import the Workflow

1. Download `workflow.json`
2. In n8n, go to **Workflows → Import from File**
3. Select the JSON file

### Step 2 — Configure Credentials

You'll need to set up the following credentials in n8n before activating:

- **Frappe CRM** — API token (`token YOUR_KEY:YOUR_SECRET`)
- **Twilio** — Account SID and Auth Token
- **Retell AI** — API Key
- **Google Calendar** — OAuth2 or Service Account
- **Gmail / Email** — For confirmation emails

### Step 3 — Set Configuration Variables

Each sub-workflow has a **Workflow Configuration** node. Update the following values:

```
FRAPPE_URL              → https://YOUR-INSTANCE.frappe.cloud
FRAPPE_TOKEN            → token YOUR_API_KEY:YOUR_API_SECRET
TWILIO_OUTBOUND_WEBHOOK → https://YOUR-N8N-INSTANCE/webhook/start-outbound-call
TWILIO_ACCOUNT_SID      → YOUR_TWILIO_ACCOUNT_SID
TWILIO_FROM_NUMBER      → +1XXXXXXXXXX
RETELL_API_KEY          → key_YOUR_RETELL_API_KEY
RETELL_AGENT_ID         → YOUR_RETELL_AGENT_ID
```

### Step 4 — Configure the Frappe Webhook

In your Frappe CRM instance, set up a webhook that fires on lead creation:

- **URL:** `https://YOUR-N8N-INSTANCE/webhook/new-lead`
- **Method:** POST
- **Events:** `CRM Lead - after_insert`

### Step 5 — Set Up Your Retell AI Agent

Your Retell agent needs to be configured to call these webhook URLs during conversations:

| Action | Webhook |
|--------|---------|
| Check qualification | `/webhook/retell-check-qualification` |
| Get available slots | `/webhook/retell-get-slots` |
| Book appointment | `/webhook/book-appointment` |
| Update CRM | `/webhook/update-crm` |

Refer to the [Retell AI docs](https://docs.retellai.com) for how to configure custom functions in your agent.

### Step 6 — Activate & Test

1. Activate all workflows in n8n
2. Create a test lead in Frappe CRM with a valid phone number
3. Watch the execution log to confirm the call gets triggered
4. Check your Google Calendar and Frappe CRM after the test call

---

## Frappe CRM Custom Fields

The workflow uses a few custom fields on the `CRM Lead` doctype. You'll need to create these:

| Field Name | Type | Description |
|------------|------|-------------|
| `custom_ai_call_status` | Select | Current status of the AI call |
| `custom_twilio_call_sid` | Data | Twilio call SID for reference |
| `custom_qualification_score` | Int | Lead score returned by the LLM |
| `custom_call_notes` | Text | AI-generated call summary |
| `custom_call_recording` | Data | URL to call recording |

---

## Environment Variables Reference

All sensitive config lives in the **Workflow Configuration** nodes. No credentials are hardcoded. Here's a full reference:

```env
# Frappe CRM
FRAPPE_URL=https://YOUR-INSTANCE.frappe.cloud
FRAPPE_TOKEN=token YOUR_API_KEY:YOUR_API_SECRET

# n8n Webhook Base
N8N_BASE_URL=https://YOUR-N8N-INSTANCE.app.n8n.cloud

# Twilio
TWILIO_ACCOUNT_SID=ACXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
TWILIO_FROM_NUMBER=+1XXXXXXXXXX
TWILIO_OUTBOUND_WEBHOOK=https://YOUR-N8N-INSTANCE/webhook/start-outbound-call

# Retell AI
RETELL_API_KEY=key_XXXXXXXXXXXXXXXXXXXXXXXX
RETELL_AGENT_ID=YOUR_AGENT_ID

# Google
GOOGLE_CALENDAR_ID=your-calendar@group.calendar.google.com
ERROR_LOG_SHEET_ID=YOUR_GOOGLE_SHEET_ID
```

---

## Error Handling

Every sub-workflow is connected to a global error handler. When something fails:

1. The error is caught and formatted with the node name, message, and timestamp
2. Details are appended to a Google Sheet for review
3. An SMS alert fires to the admin number via Twilio

This means if the AI call fails, or Frappe is unreachable, or the calendar booking errors out — you'll know immediately.

---

## Folder Structure

```
.
├── README.md
├── workflow.json              # n8n workflow export (sanitized)
├── assets/
│   ├── system-overview.png    # Before/after diagram
│   └── n8n-workflow.png       # Full n8n canvas screenshot
└── docs/
    └── SETUP.md               # Extended setup notes
```

---

## Changelog

### v1.1 — Production hardening
- **Security:** Added `Auth Check` Code nodes after every webhook. Each one validates an `X-Webhook-Secret` header against `WEBHOOK_SECRET` in the config. Set the env variable to activate; leave it unset during local dev and it passes through with a warning.
- **Security:** Moved `RETELL_SIP_DOMAIN` out of hardcoded code into the `Set Configuration` node.
- **Security:** Added `WEBHOOK_SECRET` variable to all config nodes.
- **Reliability:** Added duplicate call guard — `Guard — Not Already Called` IF node checks `custom_ai_call_status` before placing the call. Leads already in `Calling` or `Completed` state are silently skipped.
- **Reliability:** Added 3-retry with 2-second backoff to all 11 HTTP/external nodes (Frappe, Retell, Twilio, OpenAI, Google Calendar).
- **Reliability:** `Wait Before Call` now has an explicit 5-second delay (was unconfigured).
- **Reliability:** Added `Validate LLM Response` Code node after OpenAI — parses JSON, clamps score to 0–100, and returns a safe `not_qualified` fallback if the model returns malformed output.
- **Config:** Fixed OpenAI model ID from non-existent `gpt-5-mini` to `gpt-4o-mini`.
- **Config:** Added `TIMEZONE` variable (default: `Africa/Cairo`) to `Workflow Configuration 3`. `Find Free Slots` now uses this for correct local time slot generation.
