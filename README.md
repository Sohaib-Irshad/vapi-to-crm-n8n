# 📞 VAPI Call → AI-Powered CRM Automation (n8n Workflow)

An n8n workflow that turns every VAPI voice-AI phone call into a fully processed CRM record — no manual data entry required. The workflow listens for VAPI's end-of-call webhook, runs the transcript through a three-stage Gemini AI pipeline, and automatically syncs the result to Google Sheets, books a Google Calendar appointment when needed, opens a ClickUp task, and escalates high-priority leads to a manager.

## Overview

When a call ends, VAPI sends a webhook with the transcript, call metadata, and recording link. This workflow validates that payload, then hands it through three chained LLM agents that progressively extract, classify, and standardize the data into a clean CRM record. From there it looks up whether the caller already exists, books a calendar slot if an appointment was requested, creates a ClickUp task for follow-up, writes everything back to a Google Sheet, and fires off an AI-generated manager alert for any lead flagged as high priority, VIP, or needing review.

## How It Works

1. **VAPI Webhook** receives the end-of-call payload from VAPI.
2. **Simple Validation** checks the payload shape and extracts the transcript, duration, phone number, cost, and recording URL.
3. **Is Valid?** routes malformed payloads straight to an error response.
4. **AI Data Extractor** (Gemini) pulls the customer's name, contact details, service interest, urgency, sentiment, and intent out of the raw transcript.
5. **AI Customer Classifier** (Gemini) scores the extracted lead — status, value, priority, lead quality, and who it should be assigned to.
6. **AI Field Standardizer** (Gemini) merges the extraction and classification into one clean, CRM-ready record with consistent field names and formats.
7. **Parse Standardized Data** safely parses the AI's JSON output, falling back to a "needs manual review" record if parsing fails.
8. **Lookup Existing Customer** checks the Google Sheet for a prior record on this caller.
9. **Appointment Needed?** branches the flow depending on whether the caller asked to schedule something.
10. **Create Calendar Event** / **Create ClickUp Task** book the appointment and open a follow-up task in parallel.
11. **Update Google Sheet** appends or updates the customer's row with the full standardized record.
12. **Needs Manager Review?** checks the priority/risk flags set by the classifier.
13. **Generate Manager Alert** (Gemini) drafts a concise escalation message for high-priority or VIP leads.
14. **Success Response** / **Error Response** reply to the original VAPI webhook so the call platform knows the result.

## Features

- Webhook payload validation with a dedicated error path
- Three-stage chained AI pipeline (extract → classify → standardize) instead of one overloaded prompt
- Deduplication against existing CRM records before writing
- Conditional, parallel branching for appointment booking and task creation
- Automatic manager escalation for VIP, urgent, or risk-flagged calls
- Structured JSON responses back to VAPI for both success and failure cases
- Graceful fallback when the AI returns malformed JSON (flagged for manual review rather than silently dropped)

## Tech Stack

| Component | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow orchestration |
| [VAPI](https://vapi.ai) | Voice AI phone calls / webhook source |
| Google Gemini (via n8n LangChain nodes) | Data extraction, classification, standardization, alerting |
| Google Sheets | Lightweight CRM record store |
| Google Calendar | Appointment booking |
| ClickUp | Follow-up task management |

## Repository Structure

```
vapi-crm-automation/
├── README.md                          # This file
├── LICENSE                            # MIT license
├── .gitignore                         # Keeps local n8n/env files out of git
└── workflows/
    └── vapi-crm-automation.json       # Importable n8n workflow
```

## Prerequisites

- A running n8n instance (self-hosted or n8n Cloud) with the LangChain nodes available
- A VAPI account with an assistant configured to call this workflow's webhook on `end-of-call-report`
- A Google Cloud project with OAuth credentials for Sheets and Calendar
- A ClickUp OAuth app (or personal API token) with access to your target list
- A Google AI Studio / Gemini API key

## Quick Start

1. Clone this repository.
2. In n8n, go to **Workflows → Import from File** and select `workflows/vapi-crm-automation.json`.
3. Open each node that has a credential placeholder and connect your own n8n credentials (see `docs/SETUP.md` for the full list).
4. Replace the placeholder IDs in the **Lookup Existing Customer**, **Update Google Sheet**, **Create Calendar Event**, and **Create ClickUp Task** nodes with your real spreadsheet ID, calendar ID, and ClickUp team/space/folder/list/user IDs.
5. Activate the workflow and copy its production webhook URL into your VAPI assistant's server URL settings.
6. Place a test call and confirm a row appears in your Google Sheet.

## Security Notes

- This workflow JSON does **not** contain real API keys or OAuth secrets — n8n stores those separately in its own credential vault and the file only references a credential ID.
- The original spreadsheet ID has been replaced with a placeholder in this repository. Before publishing your own copy, double-check every node for IDs (sheet, calendar, ClickUp team/space/list, assignee) that you don't want public, since spreadsheet/calendar IDs can make a private resource easier to guess if combined with a sharing-link misconfiguration.
- Never commit a `.env` file or an exported n8n credentials file. The included `.gitignore` already excludes the common patterns for this.

## Customization Ideas

- Swap the Gemini model for OpenAI or Anthropic by changing the LangChain chat-model node.
- Add a Slack or WhatsApp notification node alongside the manager alert.
- Extend the classifier prompt with your own lead-scoring criteria.
- Replace Google Sheets with Airtable, HubSpot, or a proper database once volume grows.

## License

Released under the [MIT License](LICENSE) — use, modify, and adapt freely.

## Contributing

Issues and pull requests are welcome. If you adapt this for a different voice-AI platform or CRM, consider opening a PR so others can benefit.
