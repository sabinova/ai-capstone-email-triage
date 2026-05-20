# Checkpoint 2 Results

**Date:** 2026-05-19  
**Team:** AI Email Triage & Auto-Responder  
**Members:** Rimsha Tahir (Component 1 — Ingestion & Classification), Sabina Ruzieva (Component 2 — Auto Response Drafting), Zainab Chaudhry (Component 3 — Integration, Testing & Dashboard)

**Test record:** 14 email records of varying types (questions, complaints, requests, informational, spam) processed through the full pipeline. Final test: Record #14 — a "Need help" request email from test@example.com — traced end-to-end from Classified status through draft generation.

## End-to-End Status: PASSED

The full pipeline runs without manual intervention. Emails enter through the webhook, get classified by the Hugging Face model, and are picked up by the auto-response drafting workflow which generates reply drafts via the Flowise LLM chain and stores them in Airtable. 13 of 14 records were processed end-to-end before the checkpoint test; the 14th was processed during testing to confirm the live pipeline.

---

## Component-by-Component Results

### Component 1: Email Ingestion, Classification & Routing (Rimsha)

- **Status:** Working
- **What happened:** Emails submitted via webhook are received by the n8n ingestion workflow, cleaned with regex-based text preprocessing, and stored in the Emails table in Airtable. The Hugging Face zero-shot classification model (`facebook/bart-large-mnli`) assigns a category (Question, Complaint, Request, Informational, Spam), an urgency level (Low, Medium, High), and a confidence score. The record status is set to `Classified` and a Route Queue value is assigned based on the classification result.
- **Evidence:** 14 records in the Emails table with Category, Urgency, Confidence, and Route Queue fields populated. All records advanced from ingestion through classification without manual intervention.
- **Screenshot:** `screenshots/emails-status-updated.png`

### Component 2: Auto Response Drafting (Sabina)

- **Status:** Working
- **What happened:** The n8n auto-response workflow polls the Emails table every 5 minutes for records with `Status = Classified`. For each classified email, it sends the email category, urgency, subject, and body to the Flowise LLM chain endpoint (`https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d`). The chain uses Groq's `llama-3.3-70b-versatile` model at temperature 0.7 to generate a context-appropriate draft reply. The draft is written to the Response Drafts table with status `NEEDS_REVIEW`, and the original email's status is updated to `Draft Generated`.
- **Evidence:** 13 draft records in the Response Drafts table, all showing the model used (`llama-3.3-70b-versatile`), linked back to the original email, and set to `NEEDS_REVIEW` status. The 14th record was processed live during the checkpoint test, confirming the pipeline is active.
- **Screenshot:** `screenshots/response-drafts-table.png`, `screenshots/flowise-chain.png`, `screenshots/n8n-workflow-success.png`

### Component 3: Integration, Testing & Dashboard (Zainab)

- **Status:** Partially Working
- **What happened:** The Airtable base structure is in place with both the Emails table and Response Drafts table correctly linked. Filtered views and basic dashboard functionality exist within Airtable's built-in interface. However, the full dashboard with dedicated views (pipeline status, error monitor, results feed) has not yet been built using Airtable Interface Designer. The data is visible and queryable, but the presentation layer for a non-technical observer is not complete.
- **Screenshot:** (Dashboard views pending — current visibility is through Airtable grid views)

---

## Gaps Found

1. **NO_DRAFT_NEEDED filter not working correctly** — The IF node in the n8n auto-response workflow uses exact-string comparison to filter out `NO_DRAFT_NEEDED` responses (for Informational and Spam categories where a reply isn't appropriate). However, 3 records (Response Drafts #5, #8, #10) have `NO_DRAFT_NEEDED` as their Draft Body, meaning they passed through the filter and were written to the table. Root cause: the Flowise response likely includes trailing whitespace or newline characters that break the exact match. **Owner: Sabina.** **Fix: Use a "contains" comparison instead of exact equals, or trim the Flowise response before comparison.**

2. **Confidence scores are low for several records** — Multiple emails received confidence scores of 0.0 to 0.4 from the Hugging Face classifier, indicating the model is uncertain about those classifications. Records with very low confidence (e.g., 0.0 for Question and Spam categories) may be misclassified, leading to inappropriate draft responses downstream. **Owner: Rimsha.** **Fix: Investigate whether the model input is being cleaned properly; consider adding a confidence threshold below which records are flagged for human review instead of auto-drafted.**

3. **Dashboard views not yet built** — The Airtable Interface Designer dashboard with pipeline status, error monitor, and results feed views has not been created. Currently, data is only visible through the default grid view. **Owner: Zainab.** **Fix: Build three Airtable Interface Designer views as specified in the Checkpoint 3 requirements.**

4. **No error-state records exist** — All 14 records progressed successfully. While this seems positive, it means the error handling path has not been tested. There are no records with `Status = Error` in either table, so we cannot confirm that error handling works. **Owner: Team.** **Fix: Intentionally submit malformed test data (empty body, missing fields) to test error paths in both Component 1 and Component 2.**

5. **Record #12 missing Route Queue** — One email (test@example.com, "Need help", row 12) shows `Status = Draft Generated` but has no Route Queue value and had a 0.0 confidence score with Question classification. This suggests the classification was unreliable but the record was still processed without flagging. **Owner: Rimsha / Sabina.** **Fix: Add routing for low-confidence edge cases; consider a confidence threshold check before draft generation.**

6. **Duplicate records exist** — Records #4 and #5 appear to be duplicates (same sender, subject, body from promo@example.com). This could indicate a webhook double-firing issue or a lack of deduplication logic. **Owner: Rimsha.** **Fix: Add deduplication check in the ingestion workflow (e.g., check for matching sender + subject + timestamp before creating a new record).**

---

## Fix Plan

| Priority | Fix | Owner | Effort |
|----------|-----|-------|--------|
| 1 | Fix NO_DRAFT_NEEDED filter — switch IF node from exact match to contains/trim | Sabina | Small (30 min) |
| 2 | Build Airtable Interface Designer dashboard (3 views) | Zainab | Medium (2 hrs) |
| 3 | Test error handling with intentionally malformed inputs | Team | Small (1 hr) |
| 4 | Investigate low-confidence classifications and add threshold routing | Rimsha | Medium (1–2 hrs) |
| 5 | Add deduplication logic to ingestion webhook | Rimsha | Small (30 min) |
| 6 | Add confidence threshold check before auto-drafting | Sabina | Small (30 min) |

---

## Pipeline Summary

| Metric | Value |
|--------|-------|
| Total records tested | 14 |
| Successfully processed end-to-end | 14 |
| Records hitting error state | 0 |
| Draft responses generated | 13 (14th confirmed during live test) |
| NO_DRAFT_NEEDED responses (correctly identified but incorrectly stored) | 3 |
| Average processing time (ingestion → draft) | < 5 minutes (limited by schedule trigger interval) |