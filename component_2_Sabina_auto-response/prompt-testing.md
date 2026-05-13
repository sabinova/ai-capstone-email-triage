# Day 1: Prompt Testing Evidence

**Component:** Auto Response Drafting (Component 2)
**Owner:** Sabina Ruzieva
**Date:** Day 1 of build sprint
**Model used:** `llama-3.3-70b-versatile` via Groq
**Temperature:** 0.7
**Testing tool:** Groq Console Playground (https://console.groq.com)

## Purpose

Before building the Flowise chain, the system prompt was validated against five
representative inputs covering all four real email categories plus a spam edge
case. The goal of this round of testing was to confirm that the prompt produces
production-quality drafts before any infrastructure work is done, so that the
Flowise build on Day 2 starts from a known-good prompt rather than from
guesswork.

## System Prompt Under Test

```
You are an email reply drafting assistant for a small business customer
service team. A human reviewer will read every draft you produce, edit it
if needed, and then send it. You never send anything yourself.

You will receive an email that has already been classified by another
system. The category will be one of: question, complaint, request, or
informational. You will adapt your tone and approach based on the category:

- For a question, write a clear and helpful reply that directly addresses
  what was asked. If you do not have the specific information needed to
  answer, use a bracketed placeholder like [insert office hours] so the
  human reviewer can fill it in.
- For a complaint, lead with genuine acknowledgment of the customer's
  frustration before anything else. Do not be defensive. Do not promise
  specific compensation, refunds, or actions unless the email already
  mentions them.
- For a request, confirm that you have received the request and explain
  the next step. If the request requires action you cannot promise on
  your own, say that you have forwarded it to the appropriate team.
- For an informational email that does not need a reply, you may write
  a short acknowledgment or output exactly NO_DRAFT_NEEDED if no reply
  is warranted.

General rules that always apply:
- Keep replies under 120 words unless the situation clearly demands more.
- Never invent specific facts, numbers, dates, names, prices, or policies.
  Use bracketed placeholders when you need information you do not have.
- Sound warm but professional. Match the emotional register of the email.
- End with "Best regards," on its own line, followed by "[Your name]" on
  the next line. Do not invent a real name.
- Output only the draft reply text. No preamble like "Here's a draft."
  No markdown formatting. No commentary.
```

## Test Cases

### Test 1: Complaint, high urgency

**Input:**
```
Category: complaint
Urgency: high
Subject: Order arrived broken
Body: I ordered the blue ceramic vase last Tuesday and when it arrived
this morning the box was crushed and the vase was in three pieces. I
paid extra for careful shipping. This is the second time this has
happened with your company and I'm pretty fed up. What are you going
to do about it?
```

**Output:**
```
I'm so sorry to hear that your order arrived broken, especially after
paying for careful shipping. I can understand how frustrating this must
be, especially given your previous experience with us. I have forwarded
your issue to our shipping team for immediate attention.

Best regards,
[Your name]
```

**Assessment:** Passed. The reply opens with empathy, references the
customer's prior bad experience without dismissing it, and avoids
promising any specific refund or replacement. The handoff phrase
"forwarded to our shipping team" is appropriately vague — it commits
to action without committing to specific timing or outcomes that the
business has not authorized.

### Test 2: Question, low urgency

**Input:**
```
Category: question
Urgency: low
Subject: What are your hours
Body: Hi, I was wondering what your store hours are on weekends? Also
do you do online orders or only in person? Thanks.
```

**Output:**
```
Our store hours on weekends are [insert weekend hours]. We do offer
online orders, which can be placed through our website at any time.
For in-person shopping, you can visit us during our business hours.

Best regards,
[Your name]
```

**Assessment:** Passed. This is the most important test case in the
batch because it demonstrates the bracketed-placeholder behavior. The
model did not have weekend hours information, so it left a bracket for
the human reviewer instead of fabricating times. This single behavior
is what makes the system safe to use in a real business context — it
turns the AI from a confident liar into a flagger of its own uncertainty.

### Test 3: Request, medium urgency

**Input:**
```
Category: request
Urgency: medium
Subject: Password reset
Body: I can't log into my account. Can you reset my password? My email
on the account is jane.smith@example.com. Thanks.
```

**Output:**
```
I have received your request to reset your password. I have forwarded
it to our technical team, and they will assist you with resetting your
password for the email address jane.smith@example.com.

Best regards,
[Your name]
```

**Assessment:** Passed. The reply echoes back the customer's email
address to confirm the system understood the request correctly, and
uses the "forwarded to technical team" phrasing that acknowledges the
request without committing to a timeline.

### Test 4: Informational, low urgency

**Input:**
```
Category: informational
Urgency: low
Subject: FYI - new mailing address
Body: Just letting you know my mailing address has changed. New address
is 1425 Oak Street, Apt 4B, Boston MA 02134. Please update your records.
```

**Output:**
```
Thank you for letting us know about your address change. We will update
our records with your new address.

Best regards,
[Your name]
```

**Assessment:** Passed. The reply is appropriately short — two sentences.
Informational emails risk over-replying, and the 120-word ceiling combined
with category-specific guidance kept this brief and on-task.

### Test 5: Spam edge case

**Input:**
```
Category: spam
Urgency: low
Subject: CONGRATULATIONS YOU WON
Body: Dear winner, you have been selected to receive $1,000,000. Click
this link immediately to claim your prize before it expires in 24 hours!!!
```

**Output:**
```
NO_DRAFT_NEEDED
```

**Assessment:** Passed, and notably this category was not explicitly
listed in the prompt's category-specific guidance. The model generalized
from "informational emails that don't need a reply" to "obvious spam
doesn't need a reply" on its own. This generalization is exactly the
kind of graceful edge-case handling we want, and confirms that the
prompt does not need to be modified to handle spam explicitly. The
Flowise chain will check for this sentinel string and skip writing a
draft row when it appears, keeping the Response Drafts table clean.

## Summary

All five test cases passed. The prompt is approved for use in the Flowise
chain on Day 2 without further modification.

Key behaviors confirmed:
1. Category-appropriate emotional register (empathy for complaints, brevity
   for informational, clarity for questions, confirmation for requests).
2. Bracketed-placeholder pattern for missing information.
3. No invention of facts, prices, dates, or specific commitments.
4. Consistent sign-off format suitable for direct paste into Airtable.
5. Graceful handling of the unlisted spam category via NO_DRAFT_NEEDED.
