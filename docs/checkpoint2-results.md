# Checkpoint 2 Results

**Date:** 2026-05-20  
**Team:** AI Email Triage & Auto-Responder  
**Members:** Rimsha Tahir (Component 1 — Ingestion & Classification), Sabina Ruzieva (Component 2 — Auto Response Drafting), Zainab Chaudhry (Component 3 — Integration, Testing & Dashboard)

**Test record:** 32 email records combining 15 pre-existing records and 17 newly generated realistic workplace emails (covering HR benefits, IT support, payroll, complaints, facilities, and spam) processed through the full pipeline. Final trace example: Draft ID #1 — a marketing spam email from promo@example.com — traced end-to-end from webhook entry to the dashboard review queue.

## End-to-End Status: PASSED

The full pipeline runs without manual intervention across all three modules. Emails enter via the production webhook, get cleaned using regex text preprocessing, and are classified by the Hugging Face zero-shot model (facebook/bart-large-mnli). These classified records are automatically picked up by the auto-response workflow, which utilizes a Flowise LLM chain backed by Groq's llama-3.3-70b-versatile model to write draft replies directly into Airtable. The entire pipeline is exposed dynamically via a published Airtable Interface Designer dashboard for human review. Out of 32 test records, 31 were processed successfully end-to-end through the automated drafting layer.
---

## Component-by-Component Results

### Component 1: Email Ingestion, Classification & Routing (Rimsha)

- **Status:** Working
- **What happened:** Emails submitted via webhook are received by the n8n ingestion workflow, stripped of URLs and special characters using regex text preprocessing, and saved into the Emails table in Airtable. The Hugging Face zero-shot classification model evaluates the text to assign an automated category (Request, Question, Complaint, Informational, Spam), an urgency level (Low, Medium, High), and a confidence score. The record status updates to Classified and a deterministic Route Queue value is designated.`Classified` and a Route Queue value is assigned based on the classification result.

- **Evidence:** 34 records successfully ingested, classified, and routed inside the Emails table. The verified category distribution resulted in 13 Requests, 7 Questions, 5 Complaints, 5 Informational, and 2 Spam records, spanning 10 Low, 15 Medium, and 7 High urgency assignments.

- **Screenshot:** `screenshots/emails-status-updated.png`

### Component 2: Auto Response Drafting (Sabina)

- **Status:** Working
- **What happened:** The n8n auto-response drafting workflow polls the Emails table for records marked Status = Classified. It securely forwards the email context (category, urgency, subject, body) to a Flowise LLM chain endpoint running Groq's llama-3.3-70b-versatile model at a temperature of 0.7. The workflow applies category-aware logic to generate tailored, polite responses. Drafts are written directly into the Response Drafts table with an initial status of NEEDS_REVIEW and are linked bidirectionally back to their source email. Non-actionable categories natively trigger a NO_DRAFT_NEEDED value instead of a reply text.

- **Evidence:**  draft records in the Response Drafts table, all showing the model used (`llama-3.3-70b-versatile`), linked back to the original email, and set to `NEEDS_REVIEW` status. The 14th record was processed live during the checkpoint test, confirming the pipeline is active.
- **Screenshot:** `screenshots/response-drafts-updated.png`, `screenshots/flowise-chain.png`, `screenshots/n8n-workflow-success.png`, `screenshots/email-status-updted.png`

### Component 3: Integration, Testing & Dashboard (Zainab)

- **Status:** Working
- **What happened:** The shared relational base structural connections work seamlessly. Bidirectional linked-record vectors connect drafts to their source rows, pulling down live metadata (such as Category and Urgency lookup fields) with 100% data integrity.
- **Screenshot:** `screenshots/dashboard-interface.png`, `screenshots/draft-review-interface.png`

---

## Gaps Found

1. **Multi-Draft Polling Loop (Race Condition)** — The n8n auto-response workflow fails to lock a record while processing it. Because the parent email status is not updated immediately before firing the Flowise chain, a single email is pulled into multiple parallel polling tracks. This resulted in 4 duplicate drafts for Row 4 and Row 14, and 2 duplicate drafts for Rows 15, 17, 19, and 22, causing immense token waste and manual review clutter. Owner: Sabina. Fix: Add an immediate status-locking update node (e.g., Status = Processing) at the start of the drafting loop.

2. **Route Queue drop on absolute zero confidence — Row 25** (test@example.com, "Need help") returned a confidence score of 0.0 from the Hugging Face model under the Question category. The routing module failed silently on this value, leaving the Route Queue field completely blank while the record advanced to drafting anyway. Owner: Rimsha. Fix: Add a fallback routing switch that forces any record with a confidence score under a specific threshold into a "Manual Triage" queue.

3. **Orphaned Parent Status (Missing Draft Record)** — Row 6 (communications@company.com, "May office bulletin") has its status flag marked as Draft Generated, yet its Response Drafts linked cell is completely empty. The system executed the parent status change without successfully writing or mapping the corresponding draft entry to the database. Owner: Sabina / Zainab. Fix: Implement a validation check ensuring a valid Draft ID is returned from the table insert before updating the parent email row status.

4. **Critical Urgency Under-Assignment** — The keyword parser fails to accurately elevate time-sensitive emergencies. Row 7 (roy.tanaka@email.com — "URGENT: Account locked...") and Row 18 (johndoe@email.com — "Urgent Payroll Issue") were both cataloged as Low urgency. Highly disruptive employee access and financial blockers are sitting at the bottom of the triage queue. Owner: Rimsha. Fix: Implement a hard regex keyword matching override that force-escalates any subject containing words like "URGENT", "ASAP", or "LOCKED" to High urgency.

5. **Weak Zero-Shot Spam Identification** — Malicious or promotional external mail is consistently penetrating the core filter layers. Row 1 (dr.adekunle@private-mail.co — classic advance-fee scam) and Row 3 (secur1ty-alert@bank-update.net — obvious credential phishing) were both misclassified as Requests with moderate confidence (0.6 and 0.5), letting phishing risks bleed right into standard operational queues. Owner: Rimsha. Fix: Introduce a lightweight pre-filtering script checking sender domain records and high-risk URL strings prior to running the AI classifier.

6. **Filter leak writing NO_DRAFT_NEEDED records to table** — The text-comparison filter successfully flags non-actionable categories (Informational/Spam), but instead of halting processing, it writes a NO_DRAFT_NEEDED string into the table as an active row (Draft IDs #5, #8, #10). Owner: Sabina. Fix: Change the IF node path to completely terminate the loop for non-actionable emails rather than creating an empty draft row.

7. **Webhook Data Duplication** — Multiple identical testing messages are breaking ingestion constraints. Rows 19/20, 27/28, and 29/30 are identical matching sets indicating lack of deduplication logic at the ingress point. Owner: Rimsha. Fix: Configure a deduplication step at the webhook node using a composite key hash (Sender + Subject + Timestamp).

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
| 7 | Data Integrity Check: Add a validation node checking for Draft ID existence before changing parent status | Zainba | Small (30 min) |
| 8 | Spam Security Pre-filter: Add a programmatic blacklist/domain reputation validation layerTeamMedium (2 hrs) | Team | Medium (2hrs)

---

## Pipeline Summary

| Metric | Value |
|--------|-------|
| Total records tested | 34 |
| Successfully processed end-to-end | 33 |
| Records with active response draft links | 33 |
| Records hitting error state | 0 |
| Draft responses generated | 13 (14th confirmed during live test) |
| Crtical routing failures (Blank queue fields)
| NO_DRAFT_NEEDED responses (correctly identified but incorrectly stored) | 3 |
| Average processing time (ingestion → draft) | < 5 minutes (limited by schedule trigger interval) |