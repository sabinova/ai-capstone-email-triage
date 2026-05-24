# Prompt Log — Zainab Chaudhry

**Project:** AI Email Triage & Auto-Responder  
**Team:** Email Triage Team  
**My Component:** Component 3 — Integration, Testing & Dashboard  
**AI Tools Used:** Claude, Airtable Interface Designer, draw.io

---

## Entry 1 — 2026-03-02 — Design the shared Airtable base schema

**Context:** Early project setup. Rimsha and Sabina had finalized what their components would produce and consume, but no one had actually created the Airtable base yet. My role required me to design the shared schema before either of them could start writing records. I needed to think through every field, data type, and linked-record relationship before touching Airtable.

**Prompt:**
> I'm designing an Airtable base for an email triage system. Two tables are needed: an Emails table (written by the ingestion workflow) and a Response Drafts table (written by the auto-response workflow). The two tables need to be linked. Here are the fields each component needs: Emails — Sender, Subject, Body, Timestamp, Category, Urgency, Confidence, Status, Route Queue. Drafts — Draft Body, Status, Created At, Model Used, Reviewer Notes. What field types should each field use, and how should the linked record relationship be set up?

**Result:** Claude produced a complete field-type mapping for both tables. For the Emails table: Sender and Subject as Single Line Text, Body as Long Text, Timestamp as Date (with time), Category and Urgency and Route Queue as Single Select with predefined option lists, Confidence as Number (formatted to 2 decimal places), Status as Single Select with the `New → Classified → Draft Generated` lifecycle. For the Response Drafts table: Draft ID as Autonumber, Linked Email as Link to Another Record pointing at the Emails table, Draft Body and Reviewer Notes as Long Text, Status as Single Select with `NEEDS_REVIEW / APPROVED / REJECTED`, Created At as Date, Model Used as Single Line Text. For the linked record relationship: set it up from the Response Drafts side (link to Emails), which automatically creates the inverse link column on the Emails table. Claude also recommended adding lookup fields on the Response Drafts side to pull Category and Urgency from the linked email record so reviewers can see the classification context without navigating away from the draft.

**Evaluation:** About 95% usable out of the box. The field types and status lifecycle were correct. The recommendation to add lookup fields for Category and Urgency was something I hadn't thought of, and it turned out to be one of the most useful parts of the dashboard later. The only thing I adjusted was the Confidence field — Claude recommended 2 decimal places but the Hugging Face API returns values like 0.6, 0.5, which display fine at the default setting, so I left it at the default.

**What I changed:** Created the Airtable base exactly as specified. Added the Category and Urgency lookup fields to the Response Drafts table from the start, based on the recommendation. Left the Confidence number format at the default instead of explicitly setting decimal places.

**What I learned:** Designing a database schema before any data flows through it is much easier when you think through each field from the perspective of who writes it vs. who reads it. Framing the prompt as "Component 1 writes these fields, Component 2 writes these fields, Component 3 reads all of them" gave Claude enough context to recommend the lookup fields — which are read-only derived fields that serve the reviewer, not the automation. That distinction doesn't come out of a generic "design my schema" prompt.

---

## Entry 2 — 2026-03-10 — Generate 30 test emails covering all categories and edge cases

**Context:** The proposal required at least 30 test emails spanning all five classification categories and all three urgency levels. I needed to create realistic workplace emails that would give the classifier a genuine challenge — not just obvious examples, but also ambiguous cases, tricky urgency signals, and at least a few edge cases that might expose weaknesses.

**Prompt:**
> Generate 30 realistic workplace emails for testing an email classification system. Categories are: Question, Complaint, Request, Informational, Spam. Urgency levels are: Low, Medium, High. Requirements: at least 5 emails per category, at least 8 per urgency level, realistic sender names and domains, varied subjects and lengths. Also include at least 3 edge cases: one email with ambiguous category, one with explicit urgency keywords in the subject, and one that looks like spam but could be a legitimate request. Format each email as a JSON object with fields: sender, subject, body, expected_category, expected_urgency.

**Result:** Claude produced 30 email JSON objects covering all required combinations. The dataset included HR benefits inquiries (open enrollment deadlines, PTO balance questions), IT support requests (VPN connectivity, account lockouts), payroll issues (missing direct deposit, recurring underpayment), office facilities complaints, all-hands announcements, newsletter blasts, and three types of spam (a phishing credential request, an advance-fee scam, and a promotional marketing email). Edge cases included: a subject line with "URGENT:" followed by a routine question (designed to test whether urgency keywords override content), a benefits-related email from an unfamiliar external domain (designed to blur the spam/legitimate-request boundary), and an IT request written in vague, incomplete language with no clear action item (designed to stress-test the classifier's confidence threshold).

**Evaluation:** High quality starting point. The emails were realistic and varied — I wouldn't have been able to write 30 from scratch without them feeling repetitive. The edge cases were well-targeted and ended up being genuinely useful: the "URGENT:" keyword email was one of the ones that later confirmed the urgency under-assignment bug (it got classified as Low urgency despite the explicit subject-line signal). About 5 of the 30 emails needed minor edits for realism (one sender domain was fictional in an obvious way, and two body lengths were unrealistically short for the scenario).

**What I changed:** Edited 5 emails for realism. Added 4 more emails during Checkpoint 2 integration testing to bring the total to 34 and ensure all gap findings from the test results were represented. Kept the JSON format for documentation but converted them to webhook payloads when actually sending them through Rimsha's ingestion workflow.

**What I learned:** Specifying distribution requirements upfront (at least 5 per category, at least 8 per urgency level) prevents the AI from clustering around the easiest categories. Without that constraint, most generated test sets skew toward Questions and Requests because those are the most common workplace email types. Also, asking explicitly for edge cases produces much more useful stress-test data than asking for general variety.

---

## Entry 3 — 2026-04-15 — Design the architecture diagram

**Context:** The project required a visual architecture diagram showing how all three components connect. I had a rough sketch on paper but needed to translate it into a proper diagram for `docs/capstone-architecture.png`. I was using draw.io and needed help structuring the layout clearly before starting to place nodes.

**Prompt:**
> I need to draw an architecture diagram for our email triage system. Three main pipeline stages: (1) Email Ingestion — webhook receives email, n8n parses and stores in Airtable. (2) Classification & Routing — n8n sends to Hugging Face API, classification result written back to Airtable. (3) Auto Response Drafting — n8n polls Airtable, sends to Flowise LLM chain (Groq backend), draft stored in Response Drafts table. A fourth stage is the Dashboard — Airtable Interface Designer showing results to a human reviewer. How should I structure this as a diagram? What grouping and layout approach makes the data flow clearest?

**Result:** Claude recommended a left-to-right swimlane layout with four labeled subsections (Ingestion, Classification, Storage, Auto-Response, Dashboard), using Airtable as the central hub that both Component 1 writes to and Component 2 reads from. It suggested using two distinct node shapes — rounded rectangles for services/APIs and cylinders for databases — to visually distinguish actions from storage. It also recommended showing the bidirectional relationship between Airtable and the Dashboard with a dashed line (read access) rather than a solid arrow (write), to clarify that the dashboard consumes data rather than producing it.

**Evaluation:** The swimlane and shape-distinction suggestions were genuinely useful and carried directly into the final diagram. The dashed-vs-solid line convention for read vs. write helped make the data flow much clearer than my original sketch. I didn't use swimlanes exactly (used shaded background boxes instead, which draw.io handles more easily), but the grouping logic was the same.

**What I changed:** Used shaded background boxes instead of swimlane containers in draw.io for ease of construction. Kept the rounded-rectangle vs. cylinder shape distinction exactly as recommended. Added color-coding per component zone (blue for ingestion, yellow for classification, purple for storage, green for dashboard) which wasn't in Claude's suggestion but improved readability.

**What I learned:** Before opening a diagramming tool, getting a clear recommendation on layout logic and visual conventions saves a lot of drag-and-rearrange time. The key question isn't "how do I draw this" but "how do I make the data flow obvious to someone seeing it for the first time." Asking that question first produced a more specific and actionable answer than asking for draw.io instructions would have.

---

## Entry 4 — 2026-05-08 — Build the Airtable Interface Designer dashboard

**Context:** Component 3 required building Airtable Interface Designer views for the human review workflow. I had never used Interface Designer before and needed to understand what was possible and how to structure two pages: a summary dashboard and a draft review queue.

**Prompt:**
> I need to build an Airtable Interface Designer dashboard for our email triage project. I want two pages. Page 1 (Dashboard): show total email count, a bar chart of urgency distribution, a bar chart of category distribution, and a filtered list of high-urgency emails. Page 2 (Draft Review): show the Response Drafts table as a list grouped by review status (NEEDS_REVIEW / APPROVED / REJECTED), with a detail panel showing the linked email's subject, body, category, urgency, and the draft body. Walk me through how to set up each element in Interface Designer.

**Result:** Claude walked me through the Interface Designer setup step by step. For the Dashboard page: add a Number element pointing to the Emails table with no filter for the total count; add a Chart element, select Bar type, set X-axis to the Urgency field and Y-axis to Count; duplicate for Category. For the high-urgency list: add a List element, filter by Urgency = High, set sort to Timestamp descending. For the Draft Review page: add a List element pointing to the Response Drafts table, enable the "Group by" option and select Status; add a Record Detail element linked to the List element's selected record, then add fields to the detail panel including Linked Email, Email Category (lookup), Email Urgency (lookup), Draft Body, Status, and Reviewer Notes.

**Evaluation:** The step-by-step instructions were accurate for Interface Designer's current UI. One step required a small adjustment — the "Group by" option is under the List element's settings panel, not immediately visible; Claude described it correctly but I needed to expand the settings section to find it. The Record Detail element linking was the most important clarification — without knowing that the detail panel is a separate element that needs to be explicitly linked to the list's selection, I would have tried to embed the detail inside the list configuration and gotten confused.

**What I changed:** Added a fourth element to the Dashboard page — a "High Urgency Queue" list — which Claude had included in the instructions but I nearly skipped assuming the charts would be sufficient. After seeing it built, the focused queue list is actually more useful to a reviewer than the charts because it surfaces the records that need immediate attention without any clicks.

**What I learned:** Interface Designer is significantly more capable than Airtable's built-in filtered views, but it requires understanding that each element (list, chart, record detail) is an independent block that needs to be explicitly connected to other blocks via the "linked to" setting. Getting that mental model right before clicking made the build much smoother than if I had just started experimenting.

---

## Entry 5 — 2026-05-16 — Create the integration test plan

**Context:** We needed a plan for running the 17-email integration batch for Checkpoint 2 — the batch that would bring the dataset from 14 to 32 records and verify all three components handled a non-trivial load correctly. I wanted a structured test plan that covered what to verify at each stage, not just "send emails and see what happens."

**Prompt:**
> Help me write an integration test plan for our email triage pipeline. The pipeline stages are: (1) webhook ingestion → Airtable, (2) Hugging Face classification → Airtable update, (3) n8n auto-draft workflow polling every 5 minutes → Flowise → Airtable Response Drafts. I'm running a batch of 17 new emails. What should I verify at each stage, what evidence should I capture, and what failure modes should I watch for?

**Result:** Claude produced a three-stage test plan with specific checkpoints for each. Stage 1 checks: all 17 records appear in Airtable within 30 seconds of sending, each has a non-empty Sender/Subject/Body, Status = Classified (not New or blank), and no duplicate records appear. Evidence to capture: screenshot of the Emails table filtered to the 17 new records. Stage 2 checks: Category and Urgency are non-blank, Confidence is between 0.0 and 1.0, Route Queue is assigned. Spot-check 5 records manually against the expected classification from the test dataset to estimate accuracy. Evidence: screenshot showing classified records with all fields populated. Stage 3 checks: within 10 minutes of Stage 2, new records appear in Response Drafts table linked to the source emails, all show NEEDS_REVIEW status, and source emails show Draft Generated. Failure modes to watch for: records stuck at Classified after 10+ minutes (drafting workflow not triggering), NO_DRAFT_NEEDED appearing as literal draft body (IF node filter bug), duplicate drafts for the same email (race condition), records with blank Route Queue or 0.0 confidence getting drafted without review.

**Evaluation:** The failure mode list was the most valuable part because it went beyond "did it work" to "how might it fail in specific ways." All four failure modes Claude listed were actually observed during the test — the NO_DRAFT_NEEDED literal body bug, the 0.0 confidence routing failure, the duplicate race condition, and one record stuck at Classified. Having named failure modes before running the test made it much easier to identify what was happening when things went wrong, rather than just seeing a wrong state and not knowing why.

**What I changed:** Added a 4th stage to the plan covering dashboard verification — confirming that charts and the draft review list updated correctly after the batch. Ran the test plan exactly as written for Stages 1–3.

**What I learned:** A test plan that names specific failure modes is far more useful than one that only describes success criteria. Knowing what wrong looks like in advance meant I caught the NO_DRAFT_NEEDED records immediately rather than scrolling past them, and I knew to check for duplicate drafts rather than just counting total draft records.

---

## Entry 6 — 2026-05-19 — Analyze the integration test results

**Context:** The 17-email integration batch had completed. The Emails table now had 32 records; the Response Drafts table had 31. I had screenshots of both tables and needed to analyze the results systematically before writing the checkpoint documentation — not just report the headline numbers, but identify gaps and issues in the actual data.

**Prompt:**
> I ran the integration test — 32 records in Emails, 31 in Response Drafts. Here's what I see: 3 draft records have "NO_DRAFT_NEEDED" as their literal Draft Body. One source email (communications@company.com, "May office bulletin") shows Status = Draft Generated but has no linked draft in the Response Drafts table. Two records (promo@example.com, identical subject and body) appear as separate rows. One record (test@example.com, "Need help") shows Confidence = 0.0 and a blank Route Queue. Two emails that I designed as obvious spam (a phishing message and an advance-fee scam) were classified as Request and Question respectively. Help me turn these observations into a structured gap analysis.

**Result:** Claude organized the observations into 7 named gaps with root cause analysis and proposed fixes for each. Gap 1 (NO_DRAFT_NEEDED filter failure): the IF node comparison is failing due to trailing whitespace in Flowise responses; fix is switching to a `contains` check or adding a trim step. Gap 2 (Orphaned parent status on Row 6): the n8n workflow updated the source email status before confirming the draft write succeeded; fix is adding a validation check requiring a Draft ID before updating parent status. Gap 3 (Duplicate ingestion on promo@example.com): no deduplication logic at the webhook entry point; fix is a composite-key hash on Sender + Subject + Timestamp. Gap 4 (Blank Route Queue on 0.0 confidence): the routing module fails silently on edge-case confidence scores; fix is a fallback "Manual Triage" queue for sub-threshold records. Gap 5 (Spam misclassification): the zero-shot classifier has a known weakness on promotional and scam email patterns; fix is a rule-based pre-filter on sender domain and known spam phrase patterns. Gap 6 (Urgency under-assignment): time-sensitive language in the body isn't being elevated properly; fix is a keyword override rule for "urgent," "ASAP," "locked out," etc. Gap 7 (Race condition duplicates): the drafting workflow has no record locking, so parallel polling cycles can pick up the same record; fix is a "Processing" status lock before the Flowise call.

**Evaluation:** The structured gap analysis was directly usable as the checkpoint2-results.md deliverable. Without this session, I would have written a summary that mentioned the 3 filter failures and the classification issues, but probably missed the race condition (since no duplicates appeared in the draft table during my specific test window) and the orphaned status problem. Claude's systematic approach of asking "what could cause this specific observation?" for each data point surfaced issues I'd seen but hadn't connected to their root causes.

**What I changed:** Used the 7-gap structure as the backbone of checkpoint2-results.md, adding test screenshots and assigning owners and effort estimates to each fix. Added a priority ranking to the fix plan based on what would be most visible in the demo.

**What I learned:** Presenting observations as a list of specific data anomalies (rather than a general "here's what happened") gives Claude enough to work with to produce actual root cause analysis rather than generic suggestions. The gap between "the filter is failing" and "the filter is failing because of trailing whitespace from Flowise" is a root cause — and I got that root cause because I described exactly what I saw in the data.

---

## Entry 7 — 2026-05-19 — Write the Checkpoint 2 audit report

**Context:** The checkpoint required a 10-question audit report covering current component status, integration gaps, schema details, error handling, test coverage, dashboard status, and priorities. I had the analysis from Entry 6 and the test results, but the audit format required pulling information from across all three components into one coherent document.

**Prompt:**
> Help me write the checkpoint2-audit.md for our email triage project. The format requires 10 questions answered in interview style. I'll give you the content for each question and you help me write clear, specific answers that are honest about gaps. Component status: all three working and integrated, 32 records processed. End-to-end trace: laura.kim@email.com flowed from webhook → classified → draft generated → visible in dashboard. Schema: [pasted the full field table from the README]. AI output structure: Flowise produces plain text under 120 words, no JSON parsing needed. Gaps: the 7 from the analysis above. Test data: 32 records, 5 categories, 3 urgency levels, specific edge cases. Dashboard: 2 pages built and published. Integration risk: NO_DRAFT_NEEDED bug, missing error handling, classifier quality on spam and urgency. Contributions since Week 8: [described each person's work]. Priorities before Checkpoint 3: [listed the fix plan].

**Result:** Claude wrote all 10 audit responses in a consistent, direct style. Each answer was factually specific — citing record counts, field names, status values, and actual error messages rather than vague descriptions. The Component 3 contribution section accurately captured the schema work, lookup field additions, the 17-email batch generation, and the dashboard build. The risk section ranked gaps by severity and correctly identified the NO_DRAFT_NEEDED bug as the most demo-visible issue. The priorities section listed 9 items in the right order with realistic time estimates.

**Evaluation:** The audit document came out strong — specific, honest, and well-organized. The key reason it worked was that I provided structured content for each question rather than asking Claude to infer from general context. Every answer in the final document was grounded in actual data from the project. The only substantial edit I made was expanding the Q9 (contributions) section to include more specific implementation details for Rimsha and Sabina's work.

**What I changed:** Expanded Q9 for all three team members to include more specific implementation details. Added a final "Structured Assessment" section (Strengths / Gaps / Overall Status) that wasn't in the original prompt but that Claude suggested as a useful summary frame.

**What I learned:** Audit documents work best when you front-load all the specific facts and let the writing layer organize and phrase them clearly. The temptation is to ask for the document first and fill in facts later — but that produces vague, generic answers that require heavy rewriting. Providing the content for each question in advance reduced editing time dramatically.

---

## Entry 8 — 2026-05-20 — Debug Airtable Interface Designer chart not updating

**Context:** After running the Checkpoint 2 integration batch, the category distribution bar chart on the Dashboard page still showed only 5 categories from the original 14-record dataset — it hadn't updated to reflect the new 32-record distribution. The chart was connected to the Emails table and all records were visible in the base, but the chart hadn't refreshed.

**Prompt:**
> My Airtable Interface Designer bar chart isn't updating after I added 17 new records. The Emails table has 32 records now but the chart still shows the old distribution. The chart is set to X-axis = Category, Y-axis = Count, no filters. What could cause this and how do I fix it?

**Result:** Claude offered three possible causes in order of likelihood: (1) the Interface Designer page needs to be refreshed — Interface Designer elements don't always auto-update when the underlying table changes, especially after a bulk insert; try closing and reopening the interface or pressing the browser refresh button. (2) The chart has a hidden filter that limits it to a specific view or date range — check the chart's filter settings for any conditions that might be excluding the new records. (3) The Emails table has a cached view that the chart is reading from rather than live data — check whether the chart is connected to "All Records" or a named view, and switch to "All Records" if it's pointing to a view.

**Evaluation:** Cause 1 was the answer. A hard browser refresh immediately updated the chart to the correct 32-record distribution. No filter issues or view-caching problems were involved. The answer took about 30 seconds to try and confirmed the issue.

**What I changed:** Refreshed the browser. Chart updated correctly. Added a note to the dashboard testing guide that Interface Designer charts require a page refresh after bulk data imports.

**What I learned:** This was a simple fix that I could have tried myself before asking — "have you tried refreshing?" is basic debugging. The lesson is that when a UI element isn't showing expected data, the first three things to try are: refresh the page, check for filters, and verify the data source is correct. These are quick checks that take under a minute each and should come before more complex debugging.

---

## Entry 9 — 2026-05-21 — Write the README and final documentation

**Context:** Final documentation pass before Checkpoint 3 submission. The repository had component-level READMEs from Rimsha and Sabina but the root README was incomplete — it had the architecture mermaid diagram and tech stack but was missing a clear How It Works section, full Airtable schema tables, component details, and the project status table. I needed to complete it in a style consistent with what existed.

**Prompt:**
> Help me complete the root README for our email triage project. The existing README has the mermaid architecture diagram, tech stack table, and Airtable schema tables. I need to add: (1) a How It Works section with numbered steps for the full pipeline, (2) a Component Details section describing each component with key implementation details, (3) a Pipeline Flow section as a code block showing the full sequence, (4) an AI Capabilities table listing each model and its purpose, and (5) a project Status table listing all milestones. Content is: [pasted all project details].

**Result:** Claude generated all five sections in clean Markdown with consistent formatting to match the existing README style. The How It Works section broke the pipeline into four numbered steps (ingestion, classification, drafting, dashboard). The Component Details section captured each component's implementation specifics accurately, including the Flowise chatflow architecture, n8n workflow node count, and Airtable schema ownership. The AI Capabilities table correctly attributed each model to its task. The Status table listed 12 milestones with checkmarks.

**Evaluation:** Very close to final quality — about 10 minutes of edits needed. The Component Details section had a few technical details that needed updating to match the final implementation (the n8n workflow had grown from 6 nodes to 9 by the time of final submission, and the polling interval was 5 minutes, not the 3 the draft mentioned). The Status table needed the correct milestone names from the course spec. Everything else was accurate.

**What I changed:** Updated node count and polling interval in Component 2 details. Fixed milestone names in the Status table. Added the demo video link after it was recorded. Minor rephrasing in the pipeline flow section.

**What I learned:** Providing all the content in a structured way (explicitly listing what goes in each section) produces a much more accurate draft than saying "write the README." The remaining edits were minor factual corrections — numbers that had changed since an earlier document was written — not structural rewrites. Having a clear "here's what each section should contain" instruction is the difference between a usable draft and a document that needs to be rebuilt from scratch.

---

## Entry 10 — 2026-05-21 — Prepare the end-to-end demo trace for presentation

**Context:** Final demo preparation. The showcase required walking through a complete end-to-end trace — one specific email from webhook submission through classification, drafting, and dashboard surfacing. I needed to script this trace clearly enough that someone unfamiliar with the system could follow it, and identify which screenshots to use as evidence at each step.

**Prompt:**
> Help me script a 5-minute end-to-end demo for our email triage project. The trace is a real record from our test dataset: laura.kim@email.com, subject "Question about open enrollment dates", body [pasted body text], classified as Question / Medium urgency / Confidence 0.5 / Route Queue = General Questions, Draft ID #29 generated by llama-3.3-70b-versatile, status NEEDS_REVIEW. The demo should walk through each pipeline stage, reference what's visible on screen at each step, and end with the human review moment in the dashboard. Keep it to 5 minutes and highlight the AI components specifically.

**Result:** Claude scripted a 5-step demo walkthrough: (1) show the raw webhook payload in ReqBin and submit — narrate that this simulates an email arriving; (2) switch to the Airtable Emails table and show the new record appearing with Status = Classified, pointing out the Category, Urgency, Confidence, and Route Queue fields — narrate the Hugging Face zero-shot classification; (3) wait for the 5-minute polling interval (or show a record already drafted) — switch to the Response Drafts table and show Draft ID #29 linked to laura.kim's email with Status = NEEDS_REVIEW — narrate the Flowise/Groq LLM chain; (4) open the Airtable Interface Designer dashboard and show the category and urgency charts updating — narrate the dashboard's role in giving reviewers the overview; (5) navigate to the Draft Review page, click laura.kim's draft, show the detail panel with the email body and the AI-generated reply side-by-side — narrate the human-in-the-loop review moment and approve the draft live.

**Evaluation:** The script was exactly right in structure and the 5-step breakdown matched the architecture cleanly. The most useful addition was the suggested narration for Step 2 — explicitly calling out that the classification happened via a zero-shot model (no training required, just label names) is the kind of AI-specific detail that distinguishes a technical demo from a general product walkthrough.

**What I changed:** Added a brief opening context statement (30 seconds) before Step 1 covering what the system does and why. Adjusted Step 3 to use a pre-classified record rather than waiting for the live polling interval during the demo, since a 5-minute live wait is impractical in a timed presentation. Added a closing line after Step 5 noting the two known gaps (urgency under-assignment, spam pre-filter) to show honest assessment.

**What I learned:** A good demo script isn't just a sequence of actions — it's a sequence of narration choices. The question for each step is "what's the single most important thing to say here?" rather than "what am I doing here?" Getting that distinction in the script made the actual demo much smoother because each step had a clear talking point, not just a screen transition.