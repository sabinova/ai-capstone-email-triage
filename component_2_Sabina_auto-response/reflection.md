# Individual Reflection – Sabina Ruzieva

## Component 2: Auto Response Drafting

My contribution to the Email Triage & Auto-Responder project was designing
and building the auto response drafting pipeline (Component 2). This
component sits between Rimsha's email classification system (Component 1)
and Zainab's dashboard and review interface (Component 3), making its
output — the draft replies stored in Airtable — a critical handoff point
in the full system architecture.

The most important lesson I learned during this project was the value of
testing in layers before integrating. On Day 1, I validated my prompt
design by running five representative emails through the Groq playground
and critically evaluating each output. On Day 2, I wrapped the same prompt
in a Flowise LLM chain and verified via ReqBin that the API produced
equivalent-quality drafts. On Day 3, I built the n8n workflow to connect
Flowise to Airtable and tested the full pipeline end-to-end. Because each
layer was proven before adding the next, when the final integration test
ran successfully on the first attempt, I had confidence that every piece
was working correctly. This layer-by-layer testing approach is something
I plan to carry into future projects.

The hardest design decision was choosing a single universal prompt over
separate category-specific prompts. A universal prompt is faster to build
and maintain, but risks producing generic-sounding drafts. I addressed
this by embedding explicit per-category behavioral rules in the system
message (e.g., "for a complaint, lead with genuine acknowledgment of the
customer's frustration") and validating that the model followed them
across all categories. The bracketed-placeholder pattern — where the
model writes `[insert weekend hours]` instead of inventing information —
was the most critical safety feature, and confirming it worked reliably
gave me confidence that the system is safe for human reviewers to use.

One thing I would do differently with more time is implement a more robust
spam-filtering gate. The current IF node uses exact string matching for
`NO_DRAFT_NEEDED`, which occasionally fails when the Flowise response
includes trailing whitespace. A "contains" check or a dedicated
preprocessing step would make this more reliable. I would also explore
using category-specific prompt templates for higher-quality drafts in
each category, rather than relying on a single universal prompt.

Working on this project deepened my understanding of how AI components
fit into larger automation systems. Component 2 does not exist in
isolation — it depends on Component 1's classification output and feeds
into Component 3's review dashboard. Designing for this interdependence
meant thinking carefully about data contracts (what fields I read from
and write to Airtable), status management (the Classified → Draft
Generated transition), and failure handling (what happens when the AI
returns an unexpected response). These are the same concerns that
engineers at large companies deal with every day, and having hands-on
experience with them is something I value greatly from this project.