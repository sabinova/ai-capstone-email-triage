# Day 2: Flowise Chain API Testing Evidence

**Component:** Auto Response Drafting (Component 2)
**Owner:** Sabina Ruzieva
**Date:** Day 2 of build sprint
**Model used:** `llama-3.3-70b-versatile` via Groq, wrapped in Flowise LLM Chain
**Testing tool:** ReqBin (https://reqbin.com)
**Flowise endpoint:** `https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d`

## Purpose

After validating the system prompt in the Groq playground on Day 1, the prompt
was embedded in a Flowise chatflow consisting of three nodes: Chat Prompt
Template, Groq (ChatGroq), and LLM Chain. This round of testing confirms that
the chain produces the same quality drafts when called via its REST API
endpoint — the same way the production integration (Airtable Automation or
n8n) will call it.

## Flowise Chain Architecture

```
Chat Prompt Template ──→ LLM Chain ←── Groq (ChatGroq)
    (system prompt +         (orchestrator)    (llama-3.3-70b-versatile
     human message                              temperature: 0.7
     with variables)                            credential: groq-email-triage)
```

**Chatflow name:** email-triage-draft-generator
**Override config:** Enabled for chatPromptTemplate_0 to allow passing
promptValues (category, urgency, subject, body) via API.

## API Payload Format

```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "<category>",
      "urgency": "<urgency>",
      "subject": "<subject>",
      "body": "<body>"
    }
  }
}
```

## Test Cases

### Test 1: Complaint, high urgency

**Input:** Category: complaint | Urgency: high | Subject: Order arrived broken
**Response time:** 1105 ms | Status: 200 OK

**Output:**
```
I'm so sorry to hear that your order arrived broken, especially after paying
for careful shipping. It's unacceptable that this has happened to you twice.
I have forwarded your issue to our shipping team for immediate attention.
They will be in touch with you shortly to discuss a resolution.

Best regards,
[Your name]
```

**Assessment:** Passed. Empathetic opening, acknowledges repeat issue, no
specific compensation promised, appropriate handoff to shipping team.

### Test 2: Question, low urgency

**Input:** Category: question | Urgency: low | Subject: What are your hours
**Response time:** 633 ms | Status: 200 OK

**Output:**
```
Our store hours on weekends are [insert weekend hours]. We offer both
in-person shopping and online ordering for your convenience. You can
place online orders through our website at any time.

Best regards,
[Your name]
```

**Assessment:** Passed. Bracketed placeholder for unknown weekend hours
confirms the model does not fabricate information. Both parts of the
customer's question (hours and online ordering) are addressed.

### Test 3: Request, medium urgency

**Input:** Category: request | Urgency: medium | Subject: Password reset
**Response time:** 949 ms | Status: 200 OK

**Output:**
```
We have received your request to reset your password. We will forward it
to our technical team for assistance. You will receive an email at
jane.smith@example.com with further instructions.

Best regards,
[Your name]
```

**Assessment:** Passed. Confirms receipt of request, forwards to
appropriate team, echoes back the customer's email address for clarity.

### Test 4: Informational, low urgency

**Input:** Category: informational | Urgency: low | Subject: FYI - new mailing address
**Response time:** 551 ms | Status: 200 OK

**Output:**
```
Thank you for letting us know about your new mailing address. We will
make sure to update our records with the new information.

Best regards,
[Your name]
```

**Assessment:** Passed. Brief two-sentence acknowledgment. Does not
over-reply to an email that is purely informational.

### Test 5: Spam edge case

**Input:** Category: spam | Urgency: low | Subject: CONGRATULATIONS YOU WON
**Response time:** 394 ms | Status: 200 OK

**Output:**
```
NO_DRAFT_NEEDED
```

**Assessment:** Passed. Sentinel string returned correctly. The production
integration will check for this value and skip writing a draft row to
the Response Drafts table, keeping the review queue clean.

## Summary

All five test cases passed via the Flowise REST API. The chain produces
drafts of equivalent quality to the Day 1 Groq playground tests,
confirming that the Flowise wrapper does not degrade prompt performance.

Key metrics:
- 5/5 tests returned 200 OK
- Average response time: 726 ms
- All category-specific behaviors preserved (empathy, placeholders,
  brevity, forwarding, spam detection)
- Override config with promptValues works correctly for structured input
