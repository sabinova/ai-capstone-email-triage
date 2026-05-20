# Checkpoint 2 Audit Report

**Date:** 2026-05-19  
**Team:** AI Email Triage & Auto-Responder  
**Auditor:** Capstone project advisor (AI-assisted via Claude)  
**Repo:** ai-capstone-email-triage

---

## Audit Interview Responses

### 1. What is the current state of each component? Is it building, working, integrated, or stuck?

**Component 1 — Email Ingestion, Classification & Routing (Rimsha):** Working and integrated. The n8n webhook receives emails, cleans the text with regex, classifies them using the Hugging Face `facebook/bart-large-mnli` zero-shot model, assigns urgency and confidence scores, and writes structured results to the Airtable Emails table with status `Classified`. Route Queue values are assigned based on classification. 14 records have been processed successfully.

**Component 2 — Auto Response Drafting (Sabina):** Working and integrated. The n8n workflow polls for `Status = Classified` records every 5 minutes, sends the email data to a Flowise LLM chain backed by Groq's `llama-3.3-70b-versatile` model, and writes the generated draft to the Response Drafts table with status `NEEDS_REVIEW`. The original email's status is updated to `Draft Generated`. 13 of 14 records processed successfully in the initial run; the 14th was confirmed during live testing.

**Component 3 — Integration, Testing & Dashboard (Zainab):** In progress. The Airtable base structure with both tables and linked records is in place. Dashboard views using Airtable Interface Designer have been started but are not yet complete. The three required views (pipeline status, error monitor, results feed) are still being built.

### 2. Can you trace a single record from start to finish through the system?

Yes. A test email is submitted via webhook → Rimsha's n8n workflow receives it and creates an Airtable record in the Emails table with `Status = Unprocessed` → the same workflow cleans the text, calls the Hugging Face API, writes the category, urgency, confidence, and route queue, and sets `Status = Classified` → Sabina's n8n workflow picks up the classified record, sends the email content to the Flowise chain, receives a draft reply from Groq, creates a new record in the Response Drafts table with `Status = NEEDS_REVIEW`, and updates the original email to `Status = Draft Generated` → the completed record is visible in Airtable with all fields populated across both tables.

### 3. What is your Airtable schema? Do all team members agree on field names and status values?

**Emails table fields:** Sender, Subject, Body, Timestamp, Category (single select), Urgency (single select), Confidence (number), Status (single select), Route Queue (single select), Response Drafts (linked record)

**Status lifecycle for Emails:** `Unprocessed` → `Classified` → `Draft Generated`

**Response Drafts table fields:** Draft ID (autonumber), Linked Email (linked record), Draft Body (long text), Status (single select), Created At (date+time), Model Used (single line text), Reviewer Notes (long text), Email Category (lookup), Email Urgency (lookup)

**Status values for Response Drafts:** `NEEDS_REVIEW` → (future: `Approved` / `Rejected`)

Field names are consistent between components. The handoff between Component 1 and Component 2 relies on the `Classified` status value — both components use this exact string. No field name mismatches have been found during integration testing.

### 4. Does your AI component produce structured, parseable output? How do you know?

Yes. The Flowise LLM chain is configured with a Chat Prompt Template that includes explicit output format instructions. The system prompt instructs the model to produce a plain-text draft reply under 120 words, with bracketed placeholders for unknown information. The prompt includes per-category behavioral rules (e.g., complaints get empathetic tone, spam gets `NO_DRAFT_NEEDED`). The output is a single text string written directly to the Draft Body field — no JSON parsing required, which avoids structured output failures. Testing across 14 records confirms the output is consistently formatted and usable.

### 5. What happens when the AI model returns an unexpected or low-quality result?

Currently, there is limited handling for unexpected outputs. The IF node is designed to filter `NO_DRAFT_NEEDED` responses so they don't create unnecessary drafts, but this filter is failing due to trailing whitespace in Flowise responses (3 records made it through with `NO_DRAFT_NEEDED` as their draft body). There is no confidence threshold check before drafting — all classified records get drafted regardless of the classifier's confidence score. Records with 0.0 confidence are being drafted, which likely produces low-quality responses. There is no fallback or retry logic if the Flowise endpoint returns an error.

### 6. Do you have test data? How many records, and do they cover edge cases?

14 records exist in the Emails table covering all five classification categories (Question, Complaint, Request, Informational, Spam) with varying urgency levels and confidence scores. This covers the basic functional range. However, no intentionally malformed data has been tested — no empty bodies, missing senders, or invalid JSON payloads. The test data also includes at least one duplicate record pair (records #4 and #5 from promo@example.com with identical content), which was unintentional and reveals a lack of deduplication logic. Edge case coverage is weak.

### 7. Is there a working dashboard or integration view?

The Airtable base has grid views for both the Emails and Response Drafts tables, and linked records connect them. Zainab has started building dashboard views using Airtable Interface Designer but they are not yet complete. The three views required for Checkpoint 3 (pipeline status, error monitor, results feed) are in progress. Currently, data is visible through default grid views only.

### 8. What's your biggest integration risk right now?

Two risks stand out:

First, the **NO_DRAFT_NEEDED filter failure** — this is a known bug where the IF node's exact-string comparison doesn't match the Flowise response due to trailing whitespace. It causes unnecessary draft records to be created for email categories that shouldn't receive replies (Informational, Spam). The fix is straightforward (switch to contains comparison or trim the response), but it hasn't been implemented yet.

Second, the **lack of error handling across the pipeline**. Neither Component 1 nor Component 2 has been tested with malformed inputs. If the Hugging Face API or the Groq API returns an error, there is no error status, no retry logic, and no logging. A failure during a live demo would result in a record simply stopping mid-pipeline with no indication of what went wrong.

### 9. What has each team member contributed since Week 8?

**Rimsha:** Built and deployed the full Component 1 pipeline — webhook ingestion, text cleaning, HF classification, urgency assignment, routing logic, Airtable record creation and updates. Created the ReqBin testing guide. Wrote the initial checkpoint2-results.md and prompt log.

**Sabina:** Built and deployed the full Component 2 pipeline — Flowise chatflow design (3-node chain with Chat Prompt Template, Groq, and LLM Chain), n8n 6-node workflow (Schedule Trigger → Airtable Search → HTTP Request → IF → Create Record → Update Record), prompt engineering with per-category behavioral rules, and extensive debugging (Flowise URL path issue, override config, n8n node naming, Airtable update field mapping). Created component README, testing guide, auto-response system documentation, day-by-day testing logs, and reflection document.

**Zainab:** Began work on dashboard views using Airtable Interface Designer. Full scope of contribution to be confirmed.

### 10. What are your priorities between now and Checkpoint 3?

1. Fix the NO_DRAFT_NEEDED filter bug (Sabina — 30 min)
2. Complete the three dashboard views in Airtable Interface Designer (Zainab — 2 hrs)
3. Test with intentionally malformed data to verify error handling paths (Team — 1 hr)
4. Scale test data to 20+ records with varied, realistic inputs for Checkpoint 3 (Team — 1 hr)
5. Add a confidence threshold check before auto-drafting to prevent low-quality drafts (Sabina — 30 min)
6. Record the demo video showing the full pipeline (Team — 1 hr)
7. Finalize prompt logs to 10+ entries each (Individual)

---

## Structured Assessment

### Status: NEEDS POLISH

The core pipeline works end-to-end. Both Component 1 (Ingestion & Classification) and Component 2 (Auto Response Drafting) are functional and integrated. The primary gaps are in error handling, edge case coverage, and the dashboard presentation layer.

### Strengths

- **Full pipeline integration is working.** Records flow from webhook ingestion through AI classification through LLM-powered draft generation without manual intervention. This is the hardest part of the capstone and it's done.
- **Component 2 architecture is well-designed.** The Flowise chain with per-category prompt rules, the 120-word ceiling, and the bracketed placeholder approach produce consistent, usable drafts.
- **Airtable schema is clean and consistent.** Field names match across components, status values are agreed upon, and linked records connect the two tables correctly.
- **Documentation is thorough for Components 1 and 2.** Both have detailed READMEs, testing guides, and system documentation.

### Critical Gaps

- **NO_DRAFT_NEEDED filter bug** — 3 records in the Response Drafts table contain `NO_DRAFT_NEEDED` as their draft body. This is a visible bug that would show during a demo. Fix: switch IF node to contains comparison or add a trim step.
- **No error handling tested** — Neither component has been tested with bad inputs. No records exist in an error state. If something fails during a live demo, there's no graceful fallback.
- **Dashboard views incomplete** — The Checkpoint 3 requirement is three distinct views (pipeline status, error monitor, results feed). These are in progress but not done.
- **Test data volume insufficient for Checkpoint 3** — 14 records is enough for Checkpoint 2, but Checkpoint 3 requires 20+ records with evidence of actual pipeline execution at scale.

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
- The Response Drafts table was created and populated with 13 AI-generated drafts
- The pipeline was tested end-to-end with 14 records across all classification categories
- Integration between Component 1 and Component 2 was confirmed working via status-based handoff
- The NO_DRAFT_NEEDED bug was discovered and documented
- Low-confidence classification issues were identified as a downstream quality risk