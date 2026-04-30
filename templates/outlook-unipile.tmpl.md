---
name: {{SKILL_SLUG}}
description: Triage and classify emails for the {{BRAND_NAME}} mailbox ({{PRIMARY_EMAIL}} — Outlook via Unipile). Use when the user says "triage {{BRAND_NAME}}", "what's in {{BRAND_NAME}}", "check {{BRAND_NAME}} email". Surfaces drafts for approval. Never auto-sends.
---

# {{BRAND_NAME}} Email Triage

Inbox-specific triage for **{{PRIMARY_EMAIL}}** (Outlook via Unipile).

## Provider & MCP

| Field | Value |
|---|---|
| Provider | Microsoft 365 (Outlook) |
| MCP | Unipile |
| Filing capability | ✓ via direct REST API (the MCP `update_email` tool is broken — see below) |
| Draft capability | ✓ via the user's native Outlook MCP or ms365 MCP (the Unipile MCP `create_email_draft` returns 404) |
| Reading capability | ✓ via Unipile MCP `list_emails` + `get_email` |
| Account ID | `{{UNIPILE_ACCOUNT_ID}}` |

**Note:** Unipile works for Outlook but tends to dump full bodies into list responses and the folder filter is less reliable than the native ms365 MCP. If the user wants better Outlook ergonomics, recommend the `outlook-ms365` template instead. Unipile is the right choice when the user wants ONE MCP across multiple providers.

## Known Unipile quirks (CRITICAL — read before triaging)

Verified 2026-04-30. Same gotchas as the Gmail variant — Unipile's broken endpoints affect both providers.

| Quirk | Impact | Workaround |
|---|---|---|
| `mcp__unipile__update_email` returns success but does NOT actually move emails | Filing silently fails | Use the direct REST API PUT pattern in Step 7 |
| `mcp__unipile__create_email_draft` returns 404 | Cannot save drafts via Unipile MCP | Use the user's native Outlook MCP or ms365 MCP `create-draft-email` |
| Email IDs rotate (~24h) | A list_emails ID today may 404 tomorrow | Always list fresh before any batch; never reuse IDs older than ~24h |
| 504 timeouts under volume | Bulk filing fails | 0.5s delay between calls + exponential backoff (2^attempt s) |
| Folder names case-sensitive on M365 | "INBOX" fails on Outlook (Gmail-only); use "Inbox" | Pass `folder: "Inbox"` (sentence case) for the M365 mailbox |

## Inbox config

| Field | Value |
|---|---|
| Primary email | {{PRIMARY_EMAIL}} |
<!-- BEGIN: aliases_block -->
| Aliases routing into this mailbox | {{ALIASES_CSV}} |
<!-- END: aliases_block -->
| Default fetch window | last 24h (param-tunable) |
| Notification target | {{NOTIFY_CHANNEL}} |
<!-- BEGIN: crm_block -->
| CRM | {{CRM_LABEL}} |
| CRM details | {{CRM_DETAILS}} |
<!-- END: crm_block -->
<!-- BEGIN: signature_block -->
| Signature file | `{{SIGNATURE_FILENAME}}` (in this skill folder) |
<!-- END: signature_block -->

## Brand context

**One-liner:** {{BRAND_ONELINER}}

<!-- BEGIN: clients_block -->
**Active clients**: {{ACTIVE_CLIENTS_CSV}}
<!-- END: clients_block -->

<!-- BEGIN: prospects_block -->
**Active prospects**: {{PROSPECTS_CSV}}
<!-- END: prospects_block -->

**Internal team domains**: {{INTERNAL_DOMAINS_CSV}}

<!-- BEGIN: voice_cues_block -->
**Voice cues**: {{VOICE_CUES_CSV}}
<!-- END: voice_cues_block -->

## Unipile call pattern (read this first)

### Listing emails — `mcp__unipile__list_emails`

```
mcp__unipile__list_emails
  account_id: {{UNIPILE_ACCOUNT_ID}}
  meta_only: true       # MANDATORY — without this, large mailboxes blow context
  limit: 10             # MANDATORY — keep small. 10 is enough for daily triage.
  after: "<ISO 8601 with .000Z>"   # MUST include milliseconds. Bare `Z` will fail with "Expected union value".
  folder: "Inbox"       # M365 uses sentence case. NOT "INBOX" (that's Gmail-only).
```

Generate the timestamp in shell so the format is right:
```bash
TZ='UTC' date -u -v-24H +'%Y-%m-%dT%H:%M:%S.000Z'
```

### Reading a full body — `mcp__unipile__get_email`

`meta_only: true` returns a snippet, not the full body. When drafting a reply or doing deeper classification, call `get_email` for the specific message ID.

## Triage flow

### Step 1 — Fetch (excludes flagged — see flag semantics below)

Unipile `list_emails` with the account ID, last 24h, limit 10. Then **filter out any result with `is_flagged: true`** — Unipile doesn't expose a flag-status query param, so do it post-fetch on the metadata.

**Why:** a flagged email = "the user (or a previous run) has opted this email out of triage". See "Flag semantics" below.

### Step 2 — Pre-classify (no LLM)
Same rules as other variants:
- Calendar subjects / senders → `events_calendar`
- Internal domains → `internal`: {{INTERNAL_DOMAINS_CSV}}
- No-reply prefixes → `platform_notifications`
- Common platform domains → `platform_notifications`

### Step 3 — Local sender match
<!-- BEGIN: clients_block -->
Match against active clients → `project_update` / `client_request`.
<!-- END: clients_block -->
<!-- BEGIN: prospects_block -->
Match against prospects → `lead_nurture`.
<!-- END: prospects_block -->

### Step 4 — LLM classify
Use the Email Classifier prompt below.

### Step 5 — Per-email summary
Standard structure (From, Urgency, Summary, Required Action, Thread).

### Step 6 — Draft replies

**`mcp__unipile__create_email_draft` is broken (returns 404).** Use the user's native Outlook MCP or the ms365 MCP `create-draft-email` instead. If they don't have either connected, fall back to the direct Microsoft Graph API.

<!-- BEGIN: signature_block -->
3-block HTML structure: reply text + signature from `{{SIGNATURE_FILENAME}}` + quoted thread.

Never auto-send.
<!-- END: signature_block -->
<!-- IF NOT signature_block -->
Plain reply text. No signature configured.
<!-- END IF -->

### Step 6.5 — Apply filing rules (LOCKED — read before Step 7)

Apply in order BEFORE deciding to move. Rules 1-3 cause the email to stay in INBOX with a flag; only Rule 5 actually files.

1. **Urgent → flag, don't file.** If `urgency_level >= 4`: flag the email, leave in INBOX, surface in the brief under 🔴 URGENT. Skip Step 7 for this email.
2. **Pending reply → flag, don't file.** If a draft was created this run for this thread, OR an existing draft for the same `conversationId` already lives in Drafts: flag and leave in INBOX, surface. Skip Step 7.
3. **SPAM with safe unsubscribe → trigger then file.** If category is `spam` AND the body has an `RFC 2369 List-Unsubscribe` header or visible unsubscribe link AND the sender is NOT phishing: fire HTTP GET / POST on the unsubscribe URL first, then proceed to Step 7. NEVER click unsubscribe from suspected phishing.
4. **Mark-as-read on file.** Every move in Step 7 also sets `is_read: true`.
5. **Otherwise** → proceed to Step 7 (move to folder).

### Step 7 — Move to folder (CRITICAL: MCP is broken — use direct REST API)

`mcp__unipile__update_email` returns `{"object":"EmailUpdated"}` but the move silently fails. Verified 2026-04-30. **Use the direct Unipile REST API.**

```
PUT https://{UNIPILE_DSN}/api/v1/emails/{email_id}?account_id={UNIPILE_ACCOUNT_ID}
Headers:
  X-API-KEY: {UNIPILE_API_KEY}
  Content-Type: application/json
Body:
  {"folders": ["<destination_folder_name>"]}    # plural array, NOT singular
```

Outlook folder names match what's shown in the Outlook UI (e.g. "Project Updates", "Junk Email"). If a folder doesn't exist yet, surface in the brief — the user must create it in Outlook first.

**Batch rules:** 0.5s delay between calls, exponential backoff on 504s, list fresh before any batch (IDs rotate every ~24h).

### Step 8 — Notify
{{NOTIFY_CHANNEL}}

<!-- BEGIN: slack_block -->
Slack DM the brief to the channel ID configured above.
<!-- END: slack_block -->

### Step 9 — CRM action
<!-- BEGIN: crm_block -->
For `new_lead` (and any category flagged `crm = ✓`): propose adding to {{CRM_LABEL}} ({{CRM_DETAILS}}). Always require approval.
<!-- END: crm_block -->
<!-- IF NOT crm_block -->
No CRM integration configured.
<!-- END IF -->

## Categories

{{CATEGORIES_KEPT_LIST}}

Same definitions as other variants.

## Urgency scale

| Level | Meaning |
|---|---|
| 1 | FYI / no action |
| 2 | Respond within a week |
| 3 | Respond within 2-3 days |
| 4 | Action within 24 hours |
| 5 | Urgent |

## Per-category routing table

| Category | Outlook folder | Draft? | Notify? | CRM write? |
|---|---|:-:|:-:|:-:|
{{ROUTING_TABLE_ROWS}}

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
```

User content per call:
```
Email From: <sender>
Subject: <subject>
Message: <body or first 2KB>
```

## Output brief

Standard format (see other variants).

## Flag semantics (the flag is the opt-out signal)

A flagged email = "this email is OUT of triage". Two ways an email becomes flagged:

1. **The user flags it manually** in Outlook. They want to handle it themselves and don't want triage suggesting moves on it again.
2. **This skill flags it** when filing rules 1-2 in Step 6.5 fire (urgent, or pending reply). The flag marks "left in INBOX intentionally — handle when ready".

**On every subsequent run, Step 1 excludes flagged emails from the fetch.** Triage will not reclassify them, will not propose moves, will not draft against them. They sit in INBOX until the user un-flags (after handling) — at which point the next run treats them as a normal email again.

**To flag via Unipile:** the MCP `update_email` is broken (per known quirks). Use the direct REST API:

```
PUT https://{UNIPILE_DSN}/api/v1/emails/{email_id}?account_id={UNIPILE_ACCOUNT_ID}
Body: {"flagged": true}
```

(If `flagged` doesn't work — Unipile's Outlook flag schema may differ — fall back to the ms365 MCP `update-mail-message` with `{"flag": {"flagStatus": "flagged"}}` for the flagging step only.)

Tell the user once during onboarding that flagging is THEIR opt-out lever.

## Hard rules

- NEVER send email. Drafts only.
- NEVER auto-archive without approval.
- NEVER classify an active-client email as `spam` or `newsletter`.
- NEVER process a flagged email — it's the user's opt-out signal.
