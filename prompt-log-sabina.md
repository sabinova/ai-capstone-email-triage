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