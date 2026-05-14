# Component 2 – Auto Response Drafting

**Owner:** Sabina Ruzieva

## Overview

This component generates draft email replies for classified emails using a
Flowise LLM chain powered by Groq's Llama 3.3 70B model. Drafts are
automatically written to the Response Drafts table in Airtable with status
`NEEDS_REVIEW`, allowing a human reviewer to approve, edit, or reject each
draft before it is sent. The system handles all four email categories
(question, complaint, request, informational) with category-appropriate
tone and uses a `NO_DRAFT_NEEDED` sentinel for spam.

## Architecture

```
Airtable (Emails table, Status = "Classified")
    ↓  n8n polls every 5 minutes
n8n Workflow: Component 2 - Auto Response Drafting
    ↓  reads Category, Urgency, Subject, Body
Flowise LLM Chain (email-triage-draft-generator)
    ↓  sends prompt to Groq API
Groq API (llama-3.3-70b-versatile, temperature 0.7)
    ↓  returns draft reply text
n8n IF Node (filters out NO_DRAFT_NEEDED)
    ↓  writes draft to Airtable
Airtable (Response Drafts table, Status = "NEEDS_REVIEW")
    ↓  updates source email status
Airtable (Emails table, Status = "Draft Generated")
```

## How It Works

1. A Schedule Trigger in n8n fires every 5 minutes.
2. An Airtable Search node queries the Emails table for rows where
   `{Status} = "Classified"`.
3. For each classified email, an HTTP Request node POSTs the email's
   Category, Urgency, Subject, and Body to the Flowise chain's API
   endpoint using the `overrideConfig.promptValues` format.
4. The Flowise chain combines a system prompt (defining the assistant's
   role and per-category behavior rules) with the email data, sends
   the combined prompt to Groq's Llama 3.3 70B model, and returns the
   generated draft.
5. An IF node checks whether the response is `NO_DRAFT_NEEDED` (spam
   or emails that don't warrant a reply). If it is, the email is
   skipped.
6. For valid drafts, an Airtable Create Record node writes the draft
   to the Response Drafts table with a link to the source email,
   status `NEEDS_REVIEW`, timestamp, and model identifier.
7. An Airtable Update Record node changes the source email's status
   from `Classified` to `Draft Generated`, preventing duplicate
   processing on the next trigger cycle.

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| Flowise | LLM chain builder — hosts the prompt template and Groq connection |
| Groq API | Runs llama-3.3-70b-versatile for fast text generation |
| n8n | Workflow automation — polls Airtable, calls Flowise, writes results |
| Airtable | Shared database — reads from Emails table, writes to Response Drafts |

## Prompt Design

The system uses a single universal prompt that handles all four real email
categories through category-specific behavioral instructions embedded in
the system message. Key design decisions:

- **Bracketed placeholders**: When the model lacks specific information
  (e.g., store hours, refund policies), it inserts `[insert ...]`
  placeholders rather than fabricating facts. This is the primary safety
  mechanism ensuring AI drafts don't misinform customers.
- **No invented commitments**: The model never promises specific refunds,
  timelines, or actions the business hasn't authorized.
- **Category-adaptive tone**: Complaints get empathetic openings, questions
  get direct answers, requests get confirmation and handoff language,
  informational emails get brief acknowledgments.
- **Spam handling**: The `NO_DRAFT_NEEDED` sentinel string allows the
  pipeline to skip draft generation for spam without needing a separate
  workflow branch.
- **Output format control**: The prompt specifies "no preamble, no
  markdown, no commentary" to ensure raw draft text suitable for direct
  paste into a customer email.

The full system prompt and all test results are documented in:
- `day1-prompt-testing.md` — Groq playground validation (5 test cases)
- `day2-flowise-testing.md` — Flowise API validation via ReqBin (5 test cases)

## Airtable Schema: Response Drafts Table

| Field | Type | Description |
|-------|------|-------------|
| Draft ID | Autonumber | Auto-incrementing unique identifier |
| Linked Email | Link to Emails | Foreign key linking draft to source email |
| Draft Body | Long text | Generated draft reply text |
| Status | Single select | NEEDS_REVIEW, APPROVED, or REJECTED |
| Created At | Date with time | Timestamp of draft generation |
| Model Used | Single line text | LLM model identifier (llama-3.3-70b-versatile) |
| Reviewer Notes | Long text | Space for human reviewer feedback |

## Flowise Chain Configuration

- **Chatflow name:** email-triage-draft-generator
- **Endpoint:** `https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d`
- **Nodes:** Chat Prompt Template → LLM Chain ← Groq (ChatGroq)
- **Model:** llama-3.3-70b-versatile
- **Temperature:** 0.7
- **Override config:** Enabled for promptValues (category, urgency, subject, body)

## n8n Workflow Configuration

- **Workflow name:** Component 2 - Auto Response Drafting
- **Trigger:** Schedule Trigger, every 5 minutes
- **Nodes:** Schedule Trigger → Airtable Search → HTTP Request → IF → Airtable Create → Airtable Update
- **Filter formula:** `{Status} = "Classified"`
- **Spam gate:** IF node checks `text ≠ NO_DRAFT_NEEDED`

## API Payload Format

```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "complaint",
      "urgency": "high",
      "subject": "Order arrived broken",
      "body": "Email body text here..."
    }
  }
}
```

## Test Results Summary

| Test Phase | Tool | Tests | Passed | Details |
|------------|------|-------|--------|---------|
| Day 1: Prompt validation | Groq Playground | 5 | 5 | All categories + spam edge case |
| Day 2: API validation | ReqBin → Flowise | 5 | 5 | Same cases via REST API |
| Day 3: End-to-end integration | n8n → Airtable | 13 | 13 | All classified emails processed |

## Standalone Demo

The system successfully generated draft replies for 13 classified emails
across all categories:

- **Questions** → Clear, helpful replies with bracketed placeholders for
  unknown information
- **Complaints** → Empathetic acknowledgments without false promises
- **Requests** → Confirmation of receipt with appropriate team handoff
- **Informational** → Brief acknowledgments
- **Spam** → `NO_DRAFT_NEEDED` (no draft row created)

All 13 drafts appeared in the Response Drafts table with status
`NEEDS_REVIEW`, linked to their source emails, and all source emails
were updated to status `Draft Generated`.

## Setup Instructions

To reproduce this component:

1. **Groq account**: Sign up at console.groq.com and create an API key.
2. **Flowise**: Create a chatflow with Chat Prompt Template + Groq +
   LLM Chain nodes. Paste the system prompt from `day1-prompt-testing.md`.
   Save the Groq API key as a credential. Enable override config for
   the prompt template node.
3. **Airtable**: Create the Response Drafts table in the shared base
   with the schema documented above. Add "Draft Generated" as a Status
   option in the Emails table.
4. **n8n**: Build the 6-node workflow described above. Set the Airtable
   credential, Flowise endpoint URL, and field mappings. Test with
   "Execute workflow" before activating the schedule.

## Known Limitations

- The IF node's exact-string comparison for `NO_DRAFT_NEEDED` may miss
  cases where Flowise returns the sentinel with trailing whitespace.
  A production system should use a "contains" check instead.
- The 5-minute polling interval means drafts appear with up to 5 minutes
  of latency after classification. A webhook-based trigger would reduce
  this to near-real-time.
- The model occasionally generates slightly different drafts for the same
  input due to the 0.7 temperature setting. This is intentional for
  natural-sounding language but means drafts are not deterministic.

## Status

- [x] Design complete
- [x] Flowise chain configured
- [x] System prompt validated (5 test cases, Groq playground)
- [x] Flowise API validated (5 test cases, ReqBin)
- [x] n8n workflow built
- [x] End-to-end integration tested (13 emails processed)
- [x] Airtable Response Drafts table created and populated
- [x] Documentation complete