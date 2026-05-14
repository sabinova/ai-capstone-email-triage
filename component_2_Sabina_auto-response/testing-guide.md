# Testing Guide – Auto Response Drafting (Component 2)

## Purpose
This guide explains how to test the Auto Response Drafting system
using ReqBin to call the Flowise LLM chain directly.

## Project Links
- Flowise Chain: https://cloud.flowiseai.com/chatbot/5707d01b-7614-4764-a388-fa5b0fe3f61d
- Flowise API Endpoint: https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d
- Airtable Base: (same shared base as Component 1 — Email Triage System)
- n8n Workflow: Component 2 - Auto Response Drafting (on jjmopalinski.app.n8n.cloud)
- ReqBin Testing: https://reqbin.com

## Method 1: Test via ReqBin (Flowise API directly)

### Step 1: Open ReqBin
Go to: https://reqbin.com

### Step 2: Setup Request
- Method: POST
- URL: https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d

### Step 3: Select Body
- Click Body
- Select JSON

### Step 4: Paste Test Data

Complaint example:
```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "complaint",
      "urgency": "high",
      "subject": "Order arrived broken",
      "body": "I ordered the blue ceramic vase last Tuesday and when it arrived this morning the box was crushed and the vase was in three pieces. I paid extra for careful shipping. This is the second time this has happened and I'm pretty fed up. What are you going to do about it?"
    }
  }
}
```

### Step 5: Send Request
- Click Send
- This calls the Flowise LLM chain which sends the prompt to Groq API

### Step 6: Check Results
The response JSON will contain a "text" field with the generated draft:
```json
{
  "text": "I'm so sorry to hear that your order arrived broken...",
  "question": "Draft a reply.",
  "chatId": "...",
  "chatMessageId": "..."
}
```

## Test Different Categories

### Question
```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "question",
      "urgency": "low",
      "subject": "What are your hours",
      "body": "Hi, I was wondering what your store hours are on weekends? Also do you do online orders or only in person? Thanks."
    }
  }
}
```
Expected: Reply with [insert weekend hours] placeholder, answers both questions.

### Request
```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "request",
      "urgency": "medium",
      "subject": "Password reset",
      "body": "I can't log into my account. Can you reset my password? My email on the account is jane.smith@example.com. Thanks."
    }
  }
}
```
Expected: Confirms receipt, forwards to technical team, echoes email address.

### Informational
```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "informational",
      "urgency": "low",
      "subject": "FYI - new mailing address",
      "body": "Just letting you know my mailing address has changed. New address is 1425 Oak Street, Apt 4B, Boston MA 02134. Please update your records."
    }
  }
}
```
Expected: Brief 2-sentence acknowledgment.

### Spam
```json
{
  "question": "Draft a reply.",
  "overrideConfig": {
    "promptValues": {
      "category": "spam",
      "urgency": "low",
      "subject": "CONGRATULATIONS YOU WON",
      "body": "Dear winner, you have been selected to receive $1,000,000. Click this link immediately to claim your prize before it expires in 24 hours!!!"
    }
  }
}
```
Expected: Returns "NO_DRAFT_NEEDED" — no draft is generated for spam.

## Method 2: Test via n8n Workflow (Full Pipeline)

### Step 1: Ensure Classified Emails Exist
Check the Emails table in Airtable for rows with Status = "Classified".
If none exist, either run Component 1's classification workflow first
or manually set an email's Status to "Classified" and fill in the
Category and Urgency fields.

### Step 2: Run the Workflow
Open n8n → Component 2 - Auto Response Drafting → Click "Execute workflow"

### Step 3: Verify Results
Check two things in Airtable:
1. Response Drafts table: New rows should appear with draft text,
   status NEEDS_REVIEW, and linked email references
2. Emails table: Processed emails should now show Status = "Draft Generated"

## What to Look For in Draft Quality

Good drafts will:
- Open with empathy for complaints
- Use [bracketed placeholders] for unknown facts instead of making them up
- Confirm receipt and mention team handoff for requests
- Be brief (under 120 words) for informational emails
- Return NO_DRAFT_NEEDED for spam
- End with "Best regards, [Your name]"

Bad drafts (indicating a prompt issue) would:
- Invent specific prices, dates, or policies
- Promise refunds or timelines the business hasn't authorized
- Sound generic across all categories
- Include preamble like "Here's a draft for you:"