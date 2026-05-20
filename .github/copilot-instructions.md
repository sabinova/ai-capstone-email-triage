# Copilot Instructions — AI Email Triage & Auto-Responder

## Project Overview

This is a capstone project for a university AI integration course. The system automatically classifies incoming emails by category and urgency, generates draft replies using an LLM, and presents everything through a dashboard for human review.

**Team:**
- Rimsha Tahir — Component 1: Email Ingestion, Classification & Routing
- Sabina Ruzieva — Component 2: Auto Response Drafting
- Zainab Chaudhry — Component 3: Integration, Testing & Dashboard

**Tools:** n8n (workflow automation), Flowise Cloud (LLM chain builder), Airtable (database and dashboard), Groq API (LLM backend), Hugging Face API (classification), GitHub (version control)

---

## Airtable Schema

Base name: **Email Triage System**

### Emails Table

| Field | Type | Written By | Notes |
|-------|------|------------|-------|
| Sender | Single line text | Component 1 | Email address of sender |
| Subject | Single line text | Component 1 | Original subject line |
| Body | Long text | Component 1 | Original email body |
| Timestamp | Date + time | Component 1 | When the email was received |
| Category | Single select | Component 1 | Question, Complaint, Request, Informational, Spam |
| Urgency | Single select | Component 1 | Low, Medium, High |
| Confidence | Number | Component 1 | 0.0–1.0 from HF classifier |
| Status | Single select | Components 1 & 2 | See status lifecycle below |
| Route Queue | Single select | Component 1 | General Questions, Complaints, Requests, Info, Spam Review |
| Response Drafts | Linked record | Component 2 | Links to Response Drafts table |

**Status lifecycle:** `Unprocessed` → `Classified` → `Draft Generated`

### Response Drafts Table

| Field | Type | Written By | Notes |
|-------|------|------------|-------|
| Draft ID | Autonumber | Auto | Primary key |
| Linked Email | Linked record | Component 2 | Links back to Emails table |
| Draft Body | Long text | Component 2 | AI-generated reply draft |
| Status | Single select | Component 2 | NEEDS_REVIEW (future: Approved, Rejected) |
| Created At | Date + time | Component 2 | When draft was generated |
| Model Used | Single line text | Component 2 | Always "llama-3.3-70b-versatile" currently |
| Reviewer Notes | Long text | Component 3 | For human reviewer comments |
| Email Category | Lookup | Auto | Pulled from linked Emails record |
| Email Urgency | Lookup | Auto | Pulled from linked Emails record |

---

## Component 1: Email Ingestion, Classification & Routing (Rimsha)

**Platform:** n8n  
**Workflow URL:** https://jjmopalinski.app.n8n.cloud/workflow/RL0KmfL4ZiD3Iy0B  
**Webhook endpoint:** https://jjmopalinski.app.n8n.cloud/webhook-test/email-triage

**Pipeline:** Webhook trigger → Regex text cleaning → Airtable record creation → Hugging Face API call (`facebook/bart-large-mnli` zero-shot classification) → Urgency assignment (rule-based) → Route Queue assignment → Airtable record update (Status = `Classified`)

**Webhook payload format:**
```json
{
  "sender": "user@example.com",
  "subject": "Email subject line",
  "body": "Full email body text",
  "timestamp": "2026-05-19T10:30:00"
}
```

**Classification categories:** Question, Complaint, Request, Informational, Spam  
**Urgency rules:** High → complaints or high confidence | Medium → moderate confidence | Low → spam or low confidence

**Status:** Complete and working. 14 records processed successfully.

---

## Component 2: Auto Response Drafting (Sabina)

**Platforms:** n8n + Flowise Cloud + Groq API

### Flowise Configuration

- **Chatflow name:** email-triage-draft-generator
- **Endpoint:** `https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d`
- **Chain structure:** Chat Prompt Template → LLM Chain ← Groq/ChatGroq
- **Model:** `llama-3.3-70b-versatile`
- **Temperature:** 0.7
- **Credential:** groq-email-triage
- **Override Config:** Must be enabled in Flowise security settings to allow `promptValues` in API payloads

### n8n Workflow

**Workflow name:** Component 2 - Auto Response Drafting  
**Node pipeline (6 nodes):**

1. **Schedule Trigger** — Polls every 5 minutes
2. **Search records** — Airtable search for Emails where `Status = Classified`
3. **HTTP Request** — POST to Flowise endpoint with email data as `promptValues` (category, urgency, subject, body)
4. **If** — Filters out responses containing `NO_DRAFT_NEEDED`
5. **Create a record** — Writes draft to Response Drafts table
6. **Create or update a record** — Updates original email status to `Draft Generated`

### Prompt Design

The system prompt instructs the model to act as an email reply drafting assistant for a small business customer service team. Key rules:
- 120-word maximum per draft
- Bracketed placeholders for unknown specifics (e.g., [account number], [refund amount])
- Per-category behavioral rules:
  - Question → helpful, direct answer
  - Complaint → empathetic, acknowledge issue, offer resolution
  - Request → confirm receipt, outline next steps
  - Informational → `NO_DRAFT_NEEDED`
  - Spam → `NO_DRAFT_NEEDED`

### HTTP Request Payload Format
```json
{
  "question": "Generate a reply draft for this email.",
  "overrideConfig": {
    "promptValues": {
      "category": "{{ $json.fields.Category }}",
      "urgency": "{{ $json.fields.Urgency }}",
      "subject": "{{ $json.fields.Subject }}",
      "body": "{{ $json.fields.Body }}"
    }
  }
}
```

**Status:** Complete and working. 13 draft records generated. Integrated with Component 1 via status-based handoff.

---

## Component 3: Integration, Testing & Dashboard (Zainab)

**Platform:** Airtable Interface Designer

**Planned dashboard views:**
1. **Pipeline status** — Records grouped by status (Unprocessed, Classified, Draft Generated)
2. **Error monitor** — Filtered to error states, showing what went wrong
3. **Results feed** — Completed records showing classification results and draft responses

**Status:** In progress. Dashboard views are being built. Airtable base structure and linked records are in place.

---

## Known Issues

1. **NO_DRAFT_NEEDED filter bug** — The IF node in the Component 2 n8n workflow uses exact-string comparison (`equals`) to filter `NO_DRAFT_NEEDED` responses. Flowise sometimes returns the string with trailing whitespace or newline characters, causing the filter to fail. Result: 3 records in Response Drafts have `NO_DRAFT_NEEDED` as their Draft Body. Fix: switch to `contains` comparison or add a trim step before the IF node.

2. **Low-confidence classifications** — Several records received 0.0 confidence scores from the HF classifier but were still processed through the full pipeline. There is no confidence threshold check before auto-drafting. Records with very low confidence may produce poor-quality drafts.

3. **No error handling tested** — Neither component has been tested with intentionally malformed inputs. No records exist in an error state. If the HF API or Groq API fails, records will stall mid-pipeline with no indication of failure.

4. **Duplicate records** — Records #4 and #5 in the Emails table are duplicates (same sender, subject, body). No deduplication logic exists in the ingestion workflow.

5. **Record #12 anomaly** — One email (test@example.com, "Need help") has 0.0 confidence, no Route Queue value, and was still processed through drafting. Root cause unknown.

---

## Key Technical Lessons

- **Flowise URL path:** Must use `/api/v1/prediction/` — the `/chatbot/` path does not work for API calls
- **Override Config:** Must be enabled in Flowise Cloud security settings for `promptValues` to be accepted
- **n8n node references:** Expressions must use the exact canvas node name (e.g., `$('Search records')` not `$('Airtable')`)
- **Airtable Update nodes:** Remove empty field mappings to avoid overwriting existing data with blanks
- **Status value consistency:** The exact string `Classified` (capital C) must match between Component 1's output and Component 2's search filter

---

## Repository Structure

```
ai-capstone-email-triage/
├── README.md
├── .github/
│   └── copilot-instructions.md
├── docs/
│   ├── proposal.md
│   ├── architecture.png
│   ├── architecture.drawio
│   ├── checkpoint2-results.md
│   └── checkpoint2-audit.md
├── component-1-ingestion-classification/
│   ├── README.md
│   ├── Email_Ingestion_Classification_Routing_System.md
│   └── REQBIN_GUIDE.md
├── component-2-auto-response/
│   ├── README.md
│   ├── Auto_Response_Drafting_System.md
│   ├── TESTING_GUIDE.md
│   ├── day1-prompt-testing.md
│   ├── day2-flowise-testing.md
│   ├── day3-integration-testing.md
│   └── reflection.md
├── component-3-integration-testing/
│   └── README.md
├── data/
│   └── README.md
├── prompt-log-rimsha.md
└── prompt-log-sabina.md
```

---

## Current State (Updated 2026-05-19)

**What's working:**
- Full pipeline runs end-to-end without manual intervention (Ingestion → Classification → Auto-Drafting)
- 14 emails processed, 13 drafts generated successfully
- Airtable schema is stable with consistent field names and status values across components
- Both Component 1 and Component 2 documentation is complete

**What's in progress:**
- Dashboard views in Airtable Interface Designer (Zainab)
- Scaling test data to 20+ records for Checkpoint 3

**What needs fixing:**
- NO_DRAFT_NEEDED filter bug (Sabina — switch IF node to contains)
- Error handling for malformed inputs (Team)
- Confidence threshold check before drafting (Sabina)

**Next milestones:**
- Checkpoint 3 (Week 12): 20+ records at volume, 3 dashboard views, demo video, prompt logs at 10+ entries