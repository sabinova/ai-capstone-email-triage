# Auto Response Drafting System
# Owner: Sabina Ruzieva

## Overview
This component automates the generation of draft email replies for
classified emails. It receives emails that have already been categorized
and assigned urgency levels by Component 1 (Email Ingestion &
Classification), generates context-appropriate draft responses using an
LLM, and stores them in a review queue where a human can approve, edit,
or reject each draft before sending.

The system eliminates the need to write replies from scratch for common
email types while keeping a human in the loop for quality control.

## Objectives
- Generate draft replies for classified emails automatically
- Adapt tone and content based on email category and urgency
- Never fabricate facts, prices, dates, or policy details
- Store drafts in a review queue for human approval
- Handle spam gracefully by skipping draft generation
- Prevent duplicate processing of already-handled emails

## Project Links
- Flowise Chatflow: https://cloud.flowiseai.com/chatbot/5707d01b-7614-4764-a388-fa5b0fe3f61d
- Flowise API: https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d
- Airtable Base: Email Triage System (shared with Components 1 and 3)
- n8n Workflow: Component 2 - Auto Response Drafting (jjmopalinski.app.n8n.cloud)
- Groq Console: https://console.groq.com
- Model: llama-3.3-70b-versatile

## System Workflow
```
Airtable Poll → Email Data Extraction → Flowise API Call → Draft Generation → Airtable Write → Status Update
```

## Step-by-Step

### 1. Email Polling
The n8n Schedule Trigger fires every 5 minutes and queries the Airtable
Emails table using the filter formula: {Status} = "Classified"
Only unprocessed classified emails are returned.

### 2. Data Extraction
For each classified email, the workflow reads:
- Category (question, complaint, request, informational, spam)
- Urgency (low, medium, high)
- Subject line
- Body text
- Airtable Record ID (for linking the draft back)

### 3. Flowise API Call
An HTTP Request node POSTs the email data to the Flowise LLM chain
endpoint using the overrideConfig.promptValues format:
```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "complaint",
      "urgency": "high",
      "subject": "Order arrived broken",
      "body": "Email body text..."
    }
  }
}
```

### 4. LLM Draft Generation
Inside Flowise, the chain consists of three nodes:
- Chat Prompt Template: Contains the system prompt (behavioral rules,
  category-specific instructions, output format) and the human message
  template with variable placeholders
- Groq (ChatGroq): Connects to Groq API running llama-3.3-70b-versatile
  at temperature 0.7
- LLM Chain: Orchestrates prompt construction and model invocation

The model generates a draft reply following category-specific rules:
- Questions → direct, helpful answers with [placeholders] for unknowns
- Complaints → empathetic acknowledgment, no false promises
- Requests → confirmation of receipt, team handoff
- Informational → brief acknowledgment
- Spam → returns "NO_DRAFT_NEEDED"

### 5. Spam Filtering
An IF node checks whether the Flowise response equals "NO_DRAFT_NEEDED".
- If yes (spam/no reply needed): the email is skipped
- If no (real draft): processing continues to the write step

### 6. Draft Storage
An Airtable Create Record node writes to the Response Drafts table:
- Linked Email: record ID of the source email (linked record)
- Draft Body: the generated reply text
- Status: NEEDS_REVIEW
- Created At: current timestamp
- Model Used: llama-3.3-70b-versatile
- Reviewer Notes: empty (for human reviewer)

### 7. Status Update
An Airtable Update Record node changes the source email's Status from
"Classified" to "Draft Generated". This prevents the email from being
picked up again on the next polling cycle.

## Prompt Engineering

The system uses a single universal prompt rather than separate per-category
prompts. This decision was driven by practical constraints (Flowise free
tier workflow limits) and validated by testing across all categories.

### Key Prompt Features
- Category-adaptive behavioral rules embedded in the system message
- Bracketed placeholder pattern: [insert weekend hours], [insert refund amount]
- Output format control: no preamble, no markdown, no commentary
- 120-word length ceiling for concise, reviewable drafts
- Human-in-the-loop awareness: model knows drafts will be reviewed

### Safety Mechanisms
- Model never invents specific facts, numbers, or policies
- Model never promises compensation, refunds, or timelines
- Bracketed placeholders force human reviewers to verify key details
- NO_DRAFT_NEEDED sentinel prevents drafting replies to spam

## Tools & Technologies
- LLM Chain Builder: Flowise (cloud-hosted)
- AI Model: Groq API (llama-3.3-70b-versatile, temperature 0.7)
- Workflow Automation: n8n (cloud-hosted)
- Data Storage: Airtable (shared Email Triage System base)
- Testing: Groq Playground, ReqBin, n8n manual execution

## Results
- Successfully automated draft generation for all email categories
- Processed 13 classified emails in end-to-end testing
- Generated category-appropriate drafts with correct tone and structure
- Bracketed placeholders confirmed working for unknown information
- NO_DRAFT_NEEDED correctly returned for spam emails
- Incremental processing verified (no duplicate drafts)
- Average draft generation time: ~1 second per email

## Conclusion
This component demonstrates how LLMs can be integrated into workflow
automation systems to assist with routine communication tasks while
maintaining human oversight. By combining prompt engineering with
structured data pipelines, the system produces draft replies that are
genuinely useful to human reviewers — saving time on routine emails
while ensuring accuracy and appropriate tone through the review process.
The bracketed placeholder pattern is particularly valuable as a safety
mechanism, transforming the AI from a confident content generator into
a collaborative drafting assistant that explicitly flags its own
uncertainty for human verification.