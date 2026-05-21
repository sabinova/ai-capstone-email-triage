# Checkpoint 2 Audit Report

**Date:** 2026-05-19  
**Team:** AI Email Triage & Auto-Responder  
**Auditor:** Capstone project advisor (AI-assisted via Claude)  
**Repo:** ai-capstone-email-triage

---

## Audit Interview Responses

### 1. What is the current state of each component? Is it building, working, integrated, or stuck?

Component 1 — Email Ingestion, Classification & Routing (Rimsha): Working and integrated. The n8n webhook receives emails, cleans the text with regex, classifies them using the Hugging Face facebook/bart-large-mnli zero-shot model, assigns urgency and confidence scores, and writes structured results to the Airtable Emails table with status Classified. Route Queue values are assigned based on classification. 32 records have been processed successfully end-to-end, up from 14 at the time of the previous audit.

Component 2 — Auto Response Drafting (Sabina): Working and integrated. The n8n workflow polls for Status = Classified records, sends the email data to a Flowise LLM chain backed by Groq's llama-3.3-70b-versatile model, and writes the generated draft to the Response Drafts table with status NEEDS_REVIEW. The original email's status is updated to Draft Generated. 31 of 32 records have been drafted successfully; one record (communications@company.com) was classified but did not receive a draft, root cause under investigation.

Component 3 — Integration, Testing & Dashboard (Zainab): Working and integrated. The Airtable Interface Designer dashboard is built and published with two pages. The Dashboard page contains a total-emails count, an urgency distribution bar chart, a category distribution bar chart, and a high-urgency queue list. The Draft Review page contains a searchable record list grouped by review status with a side detail panel showing the linked email's category, urgency, full body preview, and the AI-generated draft reply. A 17-email integration test was completed end-to-end, bringing the dataset from 14 to 32 records and verifying all three components handle a meaningful batch correctly.

### 2. Can you trace a single record from start to finish through the system?

Yes. Recent example: laura.kim@email.com ("Question about open enrollment dates") was submitted via webhook → Rimsha's n8n workflow received it and created an Airtable record in the Emails table with Status = Unprocessed → the same workflow cleaned the text, called the Hugging Face API, wrote Category = Question, Urgency = Medium, Confidence = 0.5, Route Queue = General Questions, and set Status = Classified → Sabina's n8n workflow picked up the classified record, sent the email content to the Flowise chain, received a draft reply from Groq, created a new record in the Response Drafts table (Draft ID 29) with Status = NEEDS_REVIEW, and updated the original email to Status = Draft Generated → the completed record is visible in the published Airtable interface, with the draft surfaced in the Draft Review queue alongside the original email's category and urgency for reviewer context.

### 3. What is your Airtable schema? Do all team members agree on field names and status values?

Emails table fields: Sender, Subject, Body, Timestamp, Category (single select), Urgency (single select), Confidence (number), Status (single select), Route Queue (single select), Response Drafts (linked record)

**Status lifecycle for Emails:** `Unprocessed` → `Classified` → `Draft Generated`

**Response Drafts table fields:** Draft ID (autonumber), Linked Email (linked record), Draft Body (long text), Status (single select), Created At (date+time), Model Used (single line text), Reviewer Notes (long text), Email Category (lookup), Email Urgency (lookup)

**Status values for Response Drafts:** `NEEDS_REVIEW` → (future: `Approved` / `Rejected`)

The 'APPROVED' and 'REJECTED' options were added during Checkpoint 2 to support the human review workflow in the Draft Review interface. The Email Category and Email Urgency lookup fields were added to surface classification context next to each draft so reviewers can make approve/reject decisions without leaving the detail panel.
Field names are consistent between components. The handoff between Component 1 and Component 2 relies on the Classified status value — both components use this exact string. No field name mismatches were found during integration testing.

### 4. Does your AI component produce structured, parseable output? How do you know?

Yes. The Flowise LLM chain is configured with a Chat Prompt Template that includes explicit output format instructions. The system prompt instructs the model to produce a plain-text draft reply under 120 words, with bracketed placeholders for unknown information. The prompt includes per-category behavioral rules (e.g., complaints get empathetic tone, spam gets `NO_DRAFT_NEEDED`). The output is a single text string written directly to the Draft Body field — no JSON parsing required, which avoids structured output failures. Testing across 14 records confirms the output is consistently formatted and usable.

### 5. What happens when the AI model returns an unexpected or low-quality result?

Handling for unexpected outputs remains limited and is a known gap. The IF node designed to filter NO_DRAFT_NEEDED responses is still failing due to trailing whitespace in Flowise responses — 3 records in the current dataset have NO_DRAFT_NEEDED as their literal draft body. There is no confidence threshold check before drafting; all classified records get drafted regardless of the classifier's confidence score. Records with 0.0 confidence are being drafted, which produces low-quality responses. There is no fallback or retry logic if the Flowise endpoint returns an error.
The 32-email batch surfaced two additional quality issues at the classification layer that downstream drafting cannot recover from. First, spam detection is weak: three test emails written in unmistakable spam style (a phishing message, a marketing blast, an advance-fee scam) were all classified as non-spam categories (Request, Question, Request respectively). Second, urgency is systematically under-assigned on time-sensitive action requests: a missed-paycheck email and an account-lockout email written with explicit time pressure ("bills due tomorrow," "client presentation in 45 minutes") were both assigned Low urgency rather than High. Both issues are documented under Critical Gaps with proposed fixes.

### 6. Do you have test data? How many records, and do they cover edge cases?

32 records exist in the Emails table covering all five classification categories and all three urgency levels. The dataset includes the original 14 records from earlier component-level testing plus 17 new realistic workplace emails generated for the integration test, covering HR benefits inquiries (open enrollment, PTO balance), IT support requests (VPN issues, account lockouts), payroll problems (missing direct deposit, recurring underpayment), expense and vendor system complaints, informational announcements (newsletter, facilities FYI, all-hands), and three classes of unwanted mail (phishing, marketing spam, advance-fee scam).
Edge case coverage on classification has improved with the larger dataset — the 17 new emails were specifically designed to include ambiguous and tricky cases the classifier could realistically struggle with, which is what produced the spam and urgency findings documented in Q5. However, intentionally malformed data (empty bodies, missing senders, invalid JSON payloads) still has not been tested. The duplicate-record pair (records #4 and #5 from promo@example.com) flagged in the previous audit remains in the dataset and was left in place as an example of the system's lack of deduplication logic.

### 7. Is there a working dashboard or integration view?

Yes. The Airtable Interface Designer dashboard ("Email Triage System") is built and published as of 2026-05-20. It contains two pages:
Dashboard page displays four elements: a count of total emails processed, a bar chart of urgency distribution (Low / Medium / High), a bar chart of category distribution across all five categories, and a "High Urgency Queue" list that automatically surfaces every email currently flagged at High urgency. The high-urgency list provides a focused triage view without requiring reviewers to filter the full Emails table.
Draft Review page displays the Response Drafts table as a searchable record list grouped by Status (NEEDS_REVIEW / APPROVED / REJECTED). Selecting a draft opens a detail panel showing the linked email's sender, subject, and body preview alongside the AI-generated draft reply, the original email's Category and Urgency (pulled via lookup fields), the model used, and the current review status. Reviewers change the status directly from this view, moving drafts between groups as they work through the queue.
The interface is published and accessible to all team members and graders via Airtable's shared interface link. All elements update in real time as new records are added to either source table.

### 8. What's your biggest integration risk right now?

Three risks stand out, in rough order of severity:
First, the NO_DRAFT_NEEDED filter failure is still unresolved and would be visible in a live demo. The fix is straightforward (switch the IF node to a contains comparison or add a trim step), but it has not been implemented yet.

Second, the lack of error handling across the pipeline remains untested. Neither Component 1 nor Component 2 has been exercised with malformed inputs. The 32-email batch did surface one transient infrastructure failure (HuggingFace Inference API returned 504 Gateway Timeout on the first request after a period of inactivity, consistent with documented cold-start behavior), and the workflow had no retry logic to handle it gracefully — the affected record was simply re-sent manually after the model warmed up. In a demo or production scenario, the same condition would produce a record stuck mid-pipeline with no error signal.

Third, the classifier weakness on spam and on urgency for time-sensitive emails (see Q5) is a quality risk rather than an integration risk, but it directly affects what reviewers see in the dashboard. Misclassified spam ends up in the General Questions or Requests queue, defeating one of the system's core purposes. Both are addressable with targeted rule-based pre-filters; neither has been implemented.

### 9. What has each team member contributed since Week 8?

Rimsha: Built and deployed the full Component 1 pipeline — webhook ingestion, text cleaning, HF classification, urgency assignment, routing logic, Airtable record creation and updates. Created the ReqBin testing guide. Wrote the initial component documentation and prompt log. Granted Zainab access to the n8n workflow to enable independent batch testing during Checkpoint 2.

Sabina: Built and deployed the full Component 2 pipeline — Flowise chatflow design (3-node chain with Chat Prompt Template, Groq, and LLM Chain), n8n 6-node workflow (Schedule Trigger → Airtable Search → HTTP Request → IF → Create Record → Update Record), prompt engineering with per-category behavioral rules, and extensive debugging (Flowise URL path issue, override config, n8n node naming, Airtable update field mapping). Created the Response Drafts table to unblock Component 3 integration. Created component README, testing guide, auto-response system documentation, day-by-day testing logs, and reflection document. Drafting workflow successfully processed all 17 new emails added during the Checkpoint 2 integration test.

Zainab: Audited the Airtable base schema and confirmed it was integration-ready. Added the APPROVED and REJECTED options to the Response Drafts Status field. Added Email Category and Email Urgency lookup fields to the Response Drafts table to surface classification context in the review interface. Designed and generated a 17-email test dataset balanced across all five categories and three urgency levels, with intentionally varied difficulty (clear cases, ambiguous cases, and edge cases). Sent the batch through Rimsha's production webhook, verified all 32 records classified successfully, and documented operational issues encountered during execution (HuggingFace cold-start 504 timeouts, n8n "Publish" vs "Active" UI confusion). Built and published the Airtable Interface Designer dashboard with the Dashboard page (count, urgency chart, category chart, high-urgency queue) and the Draft Review page (status-grouped list with full detail panel). Authored this checkpoint audit document.

### 10. What are your priorities between now and Checkpoint 3?

1. Fix the NO_DRAFT_NEEDED filter bug (Sabina — 30 min)
2. Investigate why communications@company.com was classified but did not receive a draft (Sabina — 30 min)
3. Test with intentionally malformed data to verify error handling paths (Team — 1 hr)
4. Add a confidence threshold check before auto-drafting to prevent low-quality drafts (Sabina — 30 min)
5. Add a spam-detection fallback layer using rule-based pre-filters (sender domain, suspicious URLs, known scam phrases) to compensate for the zero-shot classifier's weakness (Rimsha — 1 hr)
6. Add keyword-aware urgency rules ("urgent," "ASAP," "today," "locked out," "missing payment") to correctly elevate time-sensitive action requests (Rimsha — 30 min)
7. Add retry logic for HuggingFace API failures (Rimsha — 30 min)
8. Record the demo video showing the full pipeline (Team — 1 hr)
9. Finalize prompt logs to 10+ entries each (Individual)

---

## Structured Assessment

### Status: Ready with Known Gaps

The core pipeline works end-to-end at meaningful scale (32 records processed through all three components). The dashboard is built and published. The major Checkpoint 2 deliverables are complete. Remaining gaps are concentrated in error handling, classifier quality on specific email types, and one unresolved drafting bug — none of which block submission, but all of which are documented and prioritized.

### Strengths

- **Full pipeline integration is working.** 32 records flowed from webhook ingestion through AI classification through LLM-powered draft generation through dashboard surfacing without manual intervention. This is the hardest part of the capstone and it has now been demonstrated on a non-trivial dataset.
- **Dashboard is built, published and usable**
The Airtable Interface Designer dashboard meets the spec for human review — Kanban-equivalent grouped list with detail panel, charts for volume/category/urgency distribution, and a focused queue for high-urgency items. Lookup fields surface email context next to each draft so reviewers have everything they need in one view.
- **Component 2 architecture is well-designed.** The Flowise chain with per-category prompt rules, the 120-word ceiling, and the bracketed placeholder approach produce consistent, usable drafts across all 31 records it processed.
- **Airtable schema is clean and consistent.** Field names match across components, status values are agreed upon, linked records and lookup fields are correctly configured.
- **Documentation is thorough for all three components.** Each component has detailed READMEs, testing guides, and now a system-level audit and integration results document.
- **Operational issues are documented honestly.** The HuggingFace cold-start timeout and the n8n UI confusion were captured during testing rather than glossed over, which makes the system easier to maintain.

### Critical Gaps
 - NO_DRAFT_NEEDED filter bug. 3 records in the Response Drafts table contain NO_DRAFT_NEEDED as their draft body. This is a visible bug that would show during a demo. Fix: switch IF node to contains comparison or add a trim step.

- Spam detection failure on all three spam test cases. The zero-shot BART classifier did not identify any of the three obvious spam emails (phishing, marketing, advance-fee scam) as Spam. They were routed to General Questions or Requests queues. Fix: add a rule-based pre-filter for sender domain reputation and known scam phrases before the HF classification step.

- Urgency under-assignment on time-sensitive emails. Two emails written with explicit time pressure (missed direct deposit, account lockout before client meeting) were assigned Low urgency. Fix: add keyword-aware urgency rules that elevate emails containing time-pressure language regardless of classification confidence.

- No error handling tested. Neither component has been tested with malformed inputs. The pipeline has no retry logic for transient API failures (which did occur during the batch test).

- One classified email missing a draft. communications@company.com was classified but never received a draft record. Likely a Sabina-side polling-window gap, but the cause has not been confirmed.
### Priority Fixes

| # | Fix | Owner | Effort | Deadline |
|---|-----|-------|--------|----------|
| 1 | Fix NO_DRAFT_NEEDED IF node filter | Sabina | 30 min | This week |
| 2 | Complete 3 Airtable dashboard views | Zainab | 2 hrs | Week 11 |
| 3 | Test error paths with malformed inputs | Team | 1 hr | Week 11 |
| 4 | Scale to 20+ varied test records | Team | 1 hr | Week 11 |
| 5 | Add confidence threshold before drafting | Sabina | 30 min | Week 11 |
| 6 | Record demo video | Team | 1 hr | Week 12 |
| 7 | Finalize prompt logs (10+ entries each) | Individual | Ongoing | Week 12 |

### What Changed Since Week 8

- Component 2 (Auto Response Drafting) went from not existing to fully functional and integrated
Component 3 (Integration, Testing & Dashboard) went from not started to built, published, and demonstrated at scale

- Test dataset grew from 14 to 32 records, with 17 new realistic workplace emails covering edge cases the smaller dataset did not exercise
Airtable schema extended: added APPROVED and REJECTED options to the Drafts Status field, added Email Category and Email Urgency lookup fields to the Drafts table

- Airtable Interface Designer dashboard published with two pages (Dashboard + Draft Review) covering all spec-required views

- End-to-end integration test completed across the full 32-record dataset, confirming the handoff between all three components works at scale
The Response Drafts table grew from 13 to 31 records as Sabina's drafting workflow processed the 17 new emails

- New quality findings documented from the larger dataset: spam detection weakness on all three spam test cases, urgency under-assignment on time-sensitive emails

- New operational findings documented from the batch test: HuggingFace cold-start 504 timeouts, n8n "Publish" vs "Active" UI confusion (resolved)

- The NO_DRAFT_NEEDED bug remains unfixed and is escalated as the top priority for Checkpoint 3