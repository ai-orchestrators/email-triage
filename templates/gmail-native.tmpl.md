---
name: {{SKILL_SLUG}}
description: Triage and classify emails for the {{BRAND_NAME}} mailbox ({{PRIMARY_EMAIL}} — Gmail via the native Cowork Gmail MCP). READ-ONLY triage with drafts. Cannot file or move emails. Use when the user says "triage {{BRAND_NAME}}", "what's in {{BRAND_NAME}}", "check {{BRAND_NAME}} email". Surfaces drafts for approval. Never auto-sends.
---

# {{BRAND_NAME}} Email Triage (Read-Only)

Inbox-specific triage for **{{PRIMARY_EMAIL}}** using the native Cowork Gmail MCP.

## Provider & MCP — read this first

| Field | Value |
|---|---|
| Provider | Gmail (Google Workspace) |
| MCP | Native Cowork Gmail MCP |
| Filing capability | ✗ — this MCP cannot apply labels or archive |
| Draft capability | ✓ |

**LIMITATION:** the native Gmail MCP cannot label, move, or archive emails. This skill is therefore **read + classify + draft** only. The user reviews the brief and applies labels manually in Gmail, or upgrades to Unipile / Google Cloud APIs for automated filing.

If the user says "I want this to file the emails for me" — pause and tell them they need to switch to Unipile or set up Google Cloud APIs. Re-run the email-triage setup skill when they have one of those connected.

## Inbox config

| Field | Value |
|---|---|
| Primary email | {{PRIMARY_EMAIL}} |
<!-- BEGIN: aliases_block -->
| Aliases routing into this mailbox | {{ALIASES_CSV}} |
<!-- END: aliases_block -->
| Default fetch window | last 24h (param-tunable) |
| Notification target | {{NOTIFY_CHANNEL}} |
<!-- BEGIN: signature_block -->
| Signature file | `{{SIGNATURE_FILENAME}}` (in this skill folder) |
<!-- END: signature_block -->

## Brand context

**One-liner:** {{BRAND_ONELINER}}

<!-- BEGIN: clients_block -->
**Active clients** (auto-tag as `project_update` or `client_request`):
{{ACTIVE_CLIENTS_CSV}}
<!-- END: clients_block -->

<!-- BEGIN: prospects_block -->
**Active prospects** (auto-tag as `lead_nurture`):
{{PROSPECTS_CSV}}
<!-- END: prospects_block -->

**Internal team domains**: {{INTERNAL_DOMAINS_CSV}}

<!-- BEGIN: voice_cues_block -->
**Voice cues** — phrases that boost confidence in `client_request` / `lead_nurture`:
{{VOICE_CUES_CSV}}
<!-- END: voice_cues_block -->

## Triage flow

### Step 1 — Fetch (excludes starred — see flag semantics below)

Native Gmail MCP search for unread mail in last 24h, addressed to {{PRIMARY_EMAIL}}{{ALIASES_TO_FILTER_SUFFIX}}, **excluding starred emails**. Gmail query syntax: `is:unread -is:starred newer_than:1d`.

**Why:** starred = "the user has opted this email out of triage". See "Flag semantics" below.

### Step 2 — Pre-classify (no LLM, first match wins)

Same rules as the other Gmail/Outlook variants:
- **Calendar subjects** → `events_calendar`
- **Calendar senders** → `events_calendar`
- **Internal domains** → `internal`: {{INTERNAL_DOMAINS_CSV}}
- **No-reply prefixes** → `platform_notifications`
- **Common platform domains** → `platform_notifications`

### Step 3 — Local sender match

<!-- BEGIN: clients_block -->
Match sender against active clients → classify as `project_update` or `client_request`.
<!-- END: clients_block -->
<!-- BEGIN: prospects_block -->
Match against prospects → `lead_nurture`.
<!-- END: prospects_block -->

### Step 4 — LLM classify the rest

Use the Email Classifier prompt below.

### Step 5 — Per-email summary

Same structure as full-MCP variants.

### Step 6 — Draft replies (where the table says draft = ✓)

<!-- BEGIN: signature_block -->
Drafts include the signature from `{{SIGNATURE_FILENAME}}`. Save via the native Gmail MCP draft call.
<!-- END: signature_block -->
<!-- IF NOT signature_block -->
Save plain reply text via the native Gmail MCP draft call. No signature configured.
<!-- END IF -->

Never auto-send.

### Step 6.5 — Apply filing rules (LOCKED)

Even though this MCP can't file, the rules still drive what gets SUGGESTED in the brief and what gets surfaced under 🔴 URGENT vs LABELS-TO-APPLY.

1. **Urgent → suggest "star, don't file".** If `urgency_level >= 4`: the brief tells the user to star the email and leave in INBOX. Do NOT include this email in LABELS-TO-APPLY.
2. **Pending reply → suggest "star, don't file".** If a draft was created this run for this thread, OR an existing draft for the same `thread_id` already lives in Drafts: same as Rule 1.
3. **SPAM with safe unsubscribe → suggest unsubscribe + SPAM label.** Note: this MCP cannot fire an unsubscribe URL itself; surface the URL in the brief so the user can click once.
4. **Otherwise** → list the email in LABELS-TO-APPLY with the suggested label.

### Step 7 — Filing (manual)

This MCP cannot file emails. The brief lists the suggested label per email so the user can apply it manually in Gmail. The brief format makes this easy to action in batch.

### Step 8 — Notify

{{NOTIFY_CHANNEL}}

## Categories

{{CATEGORIES_KEPT_LIST}}

(Full definitions identical to the other variants — see Client Request, Project Update, Internal, New Lead, Lead Nurture, Industry News, Events Calendar, Support, Platform Notifications, Personal, Accounting, Spam, Partners, Other.)

## Urgency scale (1-5)

| Level | Meaning |
|---|---|
| 1 | FYI / no action |
| 2 | Respond within a week |
| 3 | Respond within 2-3 days |
| 4 | Action within 24 hours |
| 5 | Urgent — immediate attention |

## Per-category routing table (manual filing)

Since this MCP can't file, the "Suggested label" column tells the user what to apply by hand.

| Category | Suggested label | Draft? |
|---|---|:-:|
{{ROUTING_TABLE_ROWS_NO_FILING}}

## Email Classifier prompt

```
You are an email categorisation assistant for {{BRAND_NAME}}.
{{BRAND_ONELINER}}

Categorise the email into ONE of these categories:
{{CATEGORIES_KEPT_LIST_INLINE}}

Return ONLY valid JSON:
{
  "category": "[Category Name]",
  "confidence": 0.95,
  "reasoning": "Brief explanation",
  "key_points": ["Main point 1", "Main point 2"],
  "urgency_level": 3,
  "requires_action": true
}

Guidelines:
1. Analyse the entire email.
2. Sender domain matters for Internal classification.
3. Cold outreach selling TO us → spam, NEVER new_lead.
4. Calendar acceptances → ALWAYS events_calendar.
```

## Output brief

```
{{BRAND_NAME}} INBOX ({{PRIMARY_EMAIL}}) — last 24h — [N emails]

🔴 URGENT (5) — N
  - [per-email summary] → suggested label: <label>

🟡 NEEDS ATTENTION (4) — N
  - [per-email summary] → suggested label: <label>

🟢 ROUTINE (1-3) — collapsed by category

📝 DRAFTS — M (in Gmail Drafts, threaded)
  - [draft body inline + original]

LABELS TO APPLY MANUALLY:
  - <label>: <count> emails (<subject lines>)
  - <label>: <count> emails
```

## Flag semantics (the star is the opt-out signal)

A starred email = "this email is OUT of triage". For the native Gmail MCP, ONLY the user can star (this MCP can't programmatically star). The brief should tell them: "Star anything you want triage to leave alone next run."

**On every subsequent run, Step 1 excludes starred emails from the fetch** via `-is:starred` in the search query. Triage will not reclassify, suggest moves, or draft against starred emails. They sit in INBOX until the user un-stars (after handling), then the next run treats them as a normal email again.

Tell the user once during onboarding that starring is THEIR opt-out lever.

## Hard rules

- NEVER send email. Drafts only.
- This MCP can't archive — never claim it has, just suggest.
- NEVER classify an active-client email as `spam` or `newsletter`.
- NEVER process a starred email — it's the user's opt-out signal.
