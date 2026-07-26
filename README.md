# AI Customer Support Ticket Categorizer

An n8n-based triage system that takes an incoming support ticket (via webhook form submission), classifies it with Gemini across category, priority, sentiment, and assigned team, routes a notification to the right team, and logs a single consistent record in Airtable.

## Why this exists

Ticket triage is a deceptively simple-looking problem: the hard part isn't classifying a ticket, it's making sure the classification consistently maps to an actual team and an actual downstream action, and that the categories you show a human match the enum values in your actual database. This project is built around getting that consistency right, not just producing a plausible-looking label.

## Architecture

```
Webhook (ticket submitted)
   │
   ▼
Extract Form Values → Formating Data (name/email normalization)
   │
   ▼
Basic LLM Chain (Gemini, primary + fallback model)
   → category, priority, sentiment, assigned_team, summary
   → Error in Basic LLM Chain (stop) on failure
   │
   ▼
Switch (routes on assigned_team — 5 explicit branches, explicit fallback)
   │
   ├─ Technical Support → Send a message  ─┐
   ├─ Billing Team      → Send a message1 ─┤
   ├─ Sales Team        → Send a message2 ─┼─→ Merge → Create a record on airtable
   ├─ Customer Success  → Send a message3 ─┤        (single shared write, all 5 branches)
   └─ General Support   → Send a message4 ─┘
   └─ (no match)         → Nothing Matched (stop — should not occur; explicit rather than silent)
```

## Key engineering decisions

**The LLM is told explicitly how category maps to team, not left to infer it.** Early versions of this prompt asked the model to choose both a category (7 options) and a team (5 options) independently, with no stated relationship between them — that's a recipe for the same category landing on different teams across runs. The current prompt includes an explicit mapping table (e.g., "Technical Support if category is Technical Issue OR Bug Report") so the routing logic is deterministic-by-instruction rather than left to the model's judgment each time.

**A single shared Airtable write, not five duplicated ones.** All five team branches converge through a Merge node into one `Create a record` call, rather than five nodes with identical field mappings. This means a future field-mapping change only needs to happen in one place.

**Fallback model for reliability.** The classification chain uses two Gemini models wired to the same input — a preview model as primary, a stable release as fallback — consistent with the reliability pattern used across this whole portfolio of agents.

**Sentiment and category values are exact matches to the Airtable schema, not approximations.** The prompt's allowed values (Positive/Neutral/Negative, and the 7 category options) are kept in sync with the Single Select fields they get written to, so classification output never creates stray enum values in Airtable.

## Tech stack
- **Orchestration:** n8n (Code nodes limited to form-field extraction and text normalization; everything else native)
- **Classification:** Google Gemini (primary + fallback model)
- **Storage:** Airtable (single Ticket table)
- **Notifications:** Gmail

## Known limitations (honest, as of this version)
- **Category→team mapping is enforced by prompt instruction, not by code.** This is a deliberate design choice, but it means the mapping's correctness depends on the model reliably following the stated rule every time, rather than being logically guaranteed. If a ticket ever lands on an unexpected team, this prompt-following reliability — not a logic bug — is the first place to check.
- **Subject line is title-cased**, which can visibly alter acronyms and product names (e.g., "API down" → "Api Down"). Kept intentionally for casing consistency across stored tickets.
- **Notification recipient (`example@your-domain.com`) is a placeholder** — intentionally left as dummy data since this project is published publicly; replace with a real team address before deploying.

## Setup requirements
- n8n instance with `@n8n/n8n-nodes-langchain` package installed
- Airtable base with the Ticket table per the schema used throughout this build
- Gmail OAuth credential with send scope
- Google Gemini API credential (stored in n8n credential manager)
## Screenshot
<img width="1573" height="679" alt="AI Customer Support Ticket Categorizer Screenshot" src="https://github.com/user-attachments/assets/e371ec15-97ad-4c15-b5c8-efaccb75517e" />
