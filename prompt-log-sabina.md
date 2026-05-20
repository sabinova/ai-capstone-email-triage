# Prompt Log — Sabina Ruzieva

**Project:** AI Email Triage & Auto-Responder  
**Team:** Email Triage Team  
**My Component:** Component 2 — Auto Response Drafting (AI Core / Specialist)  
**AI Tools Used:** Claude, Flowise, Groq Playground

---

## Entry 1 — 2026-03-02 — Generate project proposal and component breakdown

**Context:** Early project setup. Our team had voted on the email triage topic and divided components. Rimsha created the GitHub repo and drafted the initial outline. I needed to flesh out the full proposal with detailed component descriptions, data sources, AI capabilities, success criteria, and timeline — following the lab's proposal.md template structure.

**Prompt:**
> Help me write the proposal.md for our Email Triage & Auto-Responder capstone project. We have three team members: Rimsha on ingestion and classification, I'm on auto response drafting, and Zainab on integration and dashboard. The system classifies emails and generates draft replies. Include all required sections from the lab template: team, problem statement, architecture, component breakdown with inputs/outputs/tools/standalone demos, data sources, AI capabilities, success criteria, and timeline.

**Result:** Claude generated a complete proposal covering all required sections. Each component had specific descriptions of inputs, outputs, tools, and standalone demo criteria. The AI capabilities section recommended specific models (facebook/bart-large-mnli for classification, llama-3.3-70b-versatile for draft generation). The timeline mapped correctly to the course week structure.

**Evaluation:** About 90% usable. The overall structure and content were strong and matched the lab template exactly. The team member names needed to be filled in, and the model recommendations were starting points that needed validation through actual testing. The proposal included a repository structure section that wasn't strictly part of the template but was useful reference.

**What I changed:** Added real team member names and GitHub usernames. Kept the model recommendations as-is since they looked reasonable. Moved the repository structure section to the README instead of keeping it in the proposal.

**What I learned:** Starting with a clear list of required sections (from the lab template) and providing the team structure upfront produced a much more specific output than a vague "write a proposal" request would have. The AI could map our project specifics onto the template structure because I gave it enough context about who does what.

---

## Entry 2 — 2026-03-02 — Research AI models for classification and response generation

**Context:** Needed to evaluate candidate models for both email classification (Component 1) and response generation (Component 2) to include in docs/model-research.md. I had no prior experience with Hugging Face or Groq models and needed to understand what options existed and which were appropriate for a student project.

**Prompt:**
> Research AI models for our email triage system. For classification, we need a model that can categorize emails into custom categories (question, complaint, request, informational, spam) without fine-tuning. For response generation, we need a model that can draft professional email replies through Flowise. Compare at least 3 options for each task, covering accuracy, speed, free tier availability, and fit for our project.

**Result:** Claude produced a structured comparison. For classification: facebook/bart-large-mnli (most popular zero-shot), MoritzLaurer/DeBERTa-v3-base-mnli-fever-anli (better accuracy, smaller), and typeform/distilbert-base-uncased-mnli (fastest, least accurate). For response generation: Groq llama-3.3-70b-versatile (best quality), Groq gemma2-9b-it (faster, smaller), and HF Mistral-7B-Instruct-v0.3 (avoids needing Groq account). Recommended bart-large-mnli for classification and llama-3.3-70b-versatile for generation.

**Evaluation:** Very helpful for understanding the landscape. I had no way to evaluate model quality claims myself at this stage, but the tradeoff analysis (size vs speed vs accuracy) made sense. The recommendation to use gemma2-9b-it as a fallback during development was practical. The comparison format made it easy to justify our final choices in documentation.

**What I changed:** Accepted the primary recommendations (bart-large-mnli for Rimsha's component, llama-3.3-70b-versatile for mine). Didn't end up needing the gemma2 fallback since the 70b model worked within rate limits during testing.

**What I learned:** Asking for specific comparison criteria (accuracy, speed, free tier, project fit) produced a much more useful output than just "what models should we use." The structured format also transferred directly into the model-research.md deliverable with minimal editing.

---

## Entry 3 — 2026-05-10 — Debug Flowise API endpoint URL

**Context:** Day 2 of component building. I had built the Flowise chatflow (Chat Prompt Template → Groq → LLM Chain) and was testing it via ReqBin. The request returned a 200 OK status but the response was an HTML page instead of a JSON response with a draft reply. I was using the URL from the Flowise chatbot embed page.

**Prompt:**
> On ReqBin, I tried the first test and got a 200 OK but the response is HTML, not JSON. Here's the URL I'm using: https://cloud.flowiseai.com/chatbot/5707d01b-7614-4764-a388-fa5b0fe3f61d

**Result:** Claude immediately identified the issue — the `/chatbot/` path serves the Flowise web interface HTML page, not the API endpoint. The correct path is `/api/v1/prediction/` followed by the same chatflow ID. Claude provided the corrected URL: `https://cloud.flowiseai.com/api/v1/prediction/5707d01b-7614-4764-a388-fa5b0fe3f61d`

**Evaluation:** This was a critical fix that I would have spent a long time finding on my own. The Flowise documentation doesn't clearly distinguish between the two URL paths on the main page, and getting a 200 OK response made it seem like the request was working. The distinction between the chatbot embed URL and the prediction API URL was not obvious.

**What I changed:** Replaced `/chatbot/` with `/api/v1/prediction/` in the ReqBin URL. The next request returned a proper error message asking for prompt values — which was actually progress, because it meant the API was responding correctly and just needed the right payload format.

**What I learned:** A 200 status code doesn't mean the API call worked as intended — it just means the server responded. Always check the response body format, not just the status code. Also, when a no-code platform gives you a URL, verify which URL is for the UI vs. the API — they're often different paths to the same resource.

---

## Entry 4 — 2026-05-12 — Debug n8n workflow node reference errors

**Context:** Day 3 of component building. I had built all 6 nodes in my n8n workflow (Schedule Trigger → Search records → HTTP Request → IF → Create a record → Create or update a record). The first 4 nodes had green checkmarks, but the Create a record and Create or update a record nodes showed "referenced node doesn't exist" errors. The expressions in those nodes used `$('Airtable')` to reference the search node.

**Prompt:**
> My n8n Create Record node shows "referenced node doesn't exist" error. The Linked Email field expression is `{{ [$('Airtable').item.json.id] }}`. The Create or Update node has the same error with `{{ $('Airtable').item.json.id }}`. Here's a screenshot of my workflow — the search node is called "Search records" on the canvas.

**Result:** Claude identified the root cause immediately: n8n expressions must reference the exact canvas label of the node, and my node was labeled "Search records" not "Airtable." The fix was to change `$('Airtable')` to `$('Search records')` in both expressions. Claude also caught a second issue — the Create or Update node had empty field mappings for Sender, Subject, Body, and other columns that could overwrite existing data with blanks. It recommended deleting all field mappings except the id (for matching) and Status (the only field that needs updating).

**Evaluation:** Both fixes were exactly right. The node reference error disappeared immediately after changing the name. The empty field mapping issue was a subtle bug I wouldn't have caught until data started disappearing from records. Having screenshots of the actual workflow made it possible for Claude to diagnose both issues at once.

**What I changed:** Changed `$('Airtable')` to `$('Search records')` in both node expressions. Deleted all empty field mappings from the Create or Update node, keeping only id and Status. After these two fixes, the full workflow executed successfully for the first time.

**What I learned:** In n8n, node references are literal string matches to the canvas label — not the node type. If you rename a node on the canvas, every expression that references it must be updated. Also, Airtable Update nodes should only include fields you're actively writing to — empty mappings aren't harmless, they overwrite with blanks. This was the most impactful debugging session of the project because it unblocked the entire pipeline.

---

## Entry 5 — 2026-05-19 — Analyze integration test results and write Checkpoint 2 documentation

**Context:** All components had been running. The Emails table showed 14 records processed through classification and auto-response drafting. The Response Drafts table had 13 records. I needed to analyze the actual results, identify integration gaps, and write the checkpoint2-results.md and checkpoint2-audit.md deliverables. I uploaded screenshots of both Airtable tables, the Flowise chatflow, and the n8n workflow.

**Prompt:**
> Review my Airtable screenshots. The Emails table has 14 records — 13 show "Draft Generated" status and 1 shows "Classified." The Response Drafts table has 13 records all with NEEDS_REVIEW status. Analyze the results, identify any issues, and help me write the checkpoint documentation.

**Result:** Claude identified several issues I hadn't noticed: (1) Three Response Drafts records (#5, #8, #10) had `NO_DRAFT_NEEDED` as their Draft Body — meaning the IF node filter was failing due to trailing whitespace from Flowise, a bug we had discussed as a theoretical risk that was now confirmed in real data. (2) Record #12 had 0.0 confidence and no Route Queue, suggesting a classification failure that still propagated through drafting. (3) No error-state records existed, meaning error handling had never been tested. (4) Records #4 and #5 were duplicates, revealing a deduplication gap.

**Evaluation:** This was the most valuable analysis session because it turned raw screenshots into actionable insights. I had looked at the tables and seen "everything green" but missed the NO_DRAFT_NEEDED records, the 0.0 confidence patterns, and the duplicates. Claude connected the visible data to our earlier technical discussions (the whitespace issue, the confidence threshold gap) in a way that made the gaps feel like engineering problems with solutions, not failures.

**What I changed:** Used the analysis to write a comprehensive checkpoint2-results.md with honest component-by-component assessment, a specific gap list with owners, and a prioritized fix plan. Also wrote the checkpoint2-audit.md following the full 10-question format instead of the brief summary that existed before. Updated copilot-instructions.md to cover the full project, not just Component 1.

**What I learned:** Looking at data and analyzing data are different skills. I saw 13 successful drafts and thought "it works." Claude saw 3 filter failures, a classification anomaly, missing error states, and duplicates. The lesson is to check not just "did it run" but "did every record produce the right outcome." For Checkpoint 3, I'll build specific checks into my testing: verify NO_DRAFT_NEEDED records are actually filtered, verify low-confidence records are handled appropriately, and intentionally test with bad inputs.

---

## Entry 6 — 2026-05-20 — Add error handling to the auto-response drafting workflow

**Context:** Week 10 lab, Part 1. My n8n workflow had no error handling — if the Flowise API call failed, the entire workflow stopped and the record stayed stuck at `Classified` forever with no indication of what went wrong. The Checkpoint 2 audit identified this as a critical gap.

**Prompt:**
> I need to add error handling to my n8n workflow so that when the Flowise API fails, the record gets marked as Error with a reason in Airtable instead of silently stopping. Walk me through the implementation step by step.

**Result:** Claude guided me through a four-step process: (1) add an `error_reason` long text field and an `Error` status option to the Airtable Emails table, (2) enable "Continue Using Error Output" on the HTTP Request node so the workflow doesn't crash on API failure, (3) add a new Airtable Update Record node connected to the error output that sets `Status = Error` and `error_reason` to the error message, (4) test by temporarily breaking the Flowise URL.

**Evaluation:** The implementation worked exactly as described. I broke the URL, ran the workflow, and both test records flowed down the error path — showing `Status = Error` and `error_reason = "Flowise API error: 400 - ..."` in Airtable. However, after fixing the URL back, the workflow appeared to stop after Search records. Claude explained this was actually correct — there were no `Classified` records left to process because the test had moved them all to `Error`. Setting them back to `Classified` resolved it.

During testing, I also discovered a secondary bug: one record (Informational category, row 6) had been stuck at `Classified` since before this lab. Claude identified the root cause — the IF node's false path (for NO_DRAFT_NEEDED responses) had no node to advance the status, so the record kept getting picked up and processed in an infinite loop without ever advancing. The fix was adding an Update Record node on the IF false output to set status to `Draft Generated` even when no draft is created.

**What I changed:** Added the error_reason field, enabled Continue Using Error Output, wired up the error branch Update node, and added an Update node on the IF false path for NO_DRAFT_NEEDED records. My workflow went from 6 nodes to 8 nodes with three distinct output paths: success (draft created), no-draft-needed (status advanced without draft), and error (status set to Error with reason).

**What I learned:** Two lessons. First, "Continue Using Error Output" is the key n8n setting for graceful error handling — without it, one API failure kills the entire workflow for all records, not just the failed one. Second, every branch in a workflow needs to advance the record's status, even branches where the main action is skipped. An unhandled branch creates silent infinite loops that are hard to detect because the record looks like it's just waiting to be processed.

---

## Entry 7 — 2026-05-20 — Implement confidence-based routing

**Context:** Week 10 lab, Part 2. My workflow processed all classified emails regardless of the classifier's confidence score. Records with 0.0–0.4 confidence were getting auto-drafted even though the classification was unreliable. The Checkpoint 2 audit flagged this as a quality risk.

**Prompt:**
> I need to add a confidence threshold to my n8n workflow so records above 0.5 get auto-drafted and records at or below 0.5 go to a Needs Review queue. Guide me through adding this between the Search records and HTTP Request nodes.

**Result:** Claude walked me through adding a new IF node (IF1) between Search records and HTTP Request with the condition `$json.fields.Confidence` > 0.5 (Number: greater than). True path connects to the existing HTTP Request → drafting pipeline. False path connects to a new Update Record node that sets `Status = Needs Review`.

**Evaluation:** The implementation required three debugging rounds before it worked correctly:

1. **First attempt** — both records went to the false path. Cause: Airtable sends confidence as a string, not a number. Fix: enabled "Convert types where required" toggle on the IF1 node.

2. **Second attempt** — both records still went to false path AND both got drafted. Cause: Search records was connected to both IF1 AND HTTP Request simultaneously (two output wires), so records bypassed the confidence check entirely. Fix: deleted the direct wire from Search records to HTTP Request so all records must pass through IF1 first.

3. **Third attempt** — both records still went to false path. Cause: the Airtable field name is `Confidence Score` (with a space) but the expression used `$json.fields.Confidence`. Fix: changed to `$json.fields["Confidence Score"]` using bracket notation for the space in the field name. Discovered the actual field name by examining the JSON output from the Search records node.

After all three fixes, the routing worked correctly: the 0.7 confidence record went to Draft Generated, the 0.4 confidence record went to Needs Review.

**What I changed:** Added IF1 node with confidence threshold, added Update record2 node for the Needs Review path, fixed three separate issues (type conversion, duplicate wiring, field name). The workflow now has 9 nodes with four distinct outcomes: high-confidence draft, low-confidence review, no-draft-needed skip, and API error.

**What I learned:** This entry captures the most debugging I've done in a single session. Three different root causes for the same symptom (everything going to the false path). The biggest lesson: when an n8n IF node isn't routing correctly, check three things in order: (1) Is the expression resolving to the right value? Check the JSON output. (2) Is the data type correct? Enable "Convert types where required." (3) Are the wires connected correctly? A node can have multiple output connections that bypass your logic. Also, field names with spaces require bracket notation `$json.fields["Field Name"]` — this is the same class of lesson as the node naming issue from Entry 4, but for Airtable fields instead of n8n nodes.