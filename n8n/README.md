# NDIS AI Receptionist — n8n Workflow

Import `ndis-ai-receptionist-workflow.json` via n8n's **Import from File** (or
paste into **Import from Clipboard**).

## Flow

1. **Webhook** (`POST /ndis-receptionist`) — accepts `session_id`, `message`,
   `name` (optional), `contact` (optional).
2. **AI Agent** (Anthropic Chat Model + Session Memory keyed on `session_id`)
   — answers using only the knowledge base embedded in its system prompt,
   escalating anything outside its scope. On failure (API error, timeout)
   the error output routes to **Fallback Reply**, which returns the static
   "please call us" message instead of failing the webhook.
3. **Parse AI Response** — extracts `reply` / `escalate` / `escalation_reason`
   from the model's JSON output (with a parse-failure fallback).
4. **Check Escalation** (IF `escalate === true`):
   - **true** → **Log Escalation** (Google Sheets) → **Notify Staff**
     (Twilio SMS) → **Log Conversation**
   - **false** → **Log Conversation** directly
   (Both branches feed the same "Log Conversation" node and then the same
   "Respond to Client" node — a single Merge node isn't needed here since
   only one branch ever fires per execution.)
5. **Respond to Client** — returns `{ session_id, reply }` to the chat widget.

## Before you go live, fill in / configure

- **System prompt knowledge base** (inside the AI Agent node): services
  offered, coverage area, intake process, operating hours — replace every
  `[FILL IN BY PROVIDER]` placeholder. Also replace `[Provider Name]`,
  `[EMERGENCY CONTACT PLACEHOLDER]`, and `[PHONE PLACEHOLDER]`.
- **Credentials** (create these in n8n's Credentials manager, then select
  them on the relevant node — none are hardcoded in the JSON):
  - Anthropic API credential → **Anthropic Chat Model** node
  - Google Sheets OAuth2 credential → **Log Escalation** and
    **Log Conversation** nodes
  - Twilio API credential → **Notify Staff** node
- **Google Sheets**: set `documentId` (the sheet's ID/URL) on both Sheets
  nodes, and confirm the tab names (`Escalations`, `Conversation Log`) match
  your spreadsheet, or create tabs with those names and columns:
  - `Escalations`: timestamp, session_id, name, contact, message, escalation_reason
  - `Conversation Log`: timestamp, session_id, message, reply, escalate
- **Twilio**: set the `from` (your Twilio number) and `to` (staff phone
  number) values on the **Notify Staff** node.
- **Model**: the Anthropic Chat Model node defaults to
  `claude-sonnet-4-5-20250929` — change it if you prefer a different Claude
  model.

## Notes

- The AI Agent's `onError` is set to `continueErrorOutput`, so a Claude API
  failure never breaks the webhook response — it always falls through to
  the static fallback reply.
- Session memory is keyed by the caller-supplied `session_id`, so the same
  conversation continues across multiple webhook calls from the chat widget.
