---
name: {{SKILL_SLUG}}
description: Triage and classify emails for the {{BRAND_NAME}} mailbox ({{PRIMARY_EMAIL}} — Gmail via Unipile). Use when the user says "triage {{BRAND_NAME}}", "what's in {{BRAND_NAME}}", "check {{BRAND_NAME}} email", "any new leads", or any prompt referencing the {{BRAND_NAME}} inbox. Filters to mail addressed to {{PRIMARY_EMAIL}}{{ALIASES_TO_FILTER_SUFFIX}}. Surfaces drafts for approval. Never auto-sends.
---

# {{BRAND_NAME}} Email Triage

Inbox-specific triage for **{{PRIMARY_EMAIL}}** (Gmail via Unipile).

## Provider & MCP

| Field | Value |
|---|---|
| Provider | Gmail (Google Workspace) |
| MCP | Unipile |
| Filing capability | ✓ via direct REST API (the MCP `update_email` tool is broken — see below) |
| Draft capability | ✓ via the user's native Gmail MCP (the MCP `create_email_draft` tool returns 404 — see below) |
| Reading capability | ✓ via Unipile MCP `list_emails` + `get_email` |
| Account ID | `{{UNIPILE_ACCOUNT_ID}}` |

## Known Unipile quirks (CRITICAL — read before triaging)

Verified 2026-04-30. Every triage run MUST account for these:

| Quirk | Impact | Workaround |
|---|---|---|
| `mcp__unipile__update_email` returns success but does NOT actually move emails | Filing silently fails | Use the direct REST API PUT pattern in Step 7 |
| `mcp__unipile__create_email_draft` returns 404 | Cannot save drafts via Unipile MCP | Use the user's native Gmail MCP `create_draft` tool instead |
| Email IDs rotate (~24h) | A list_emails ID today may 404 tomorrow | Always list fresh before any batch of moves; never reuse IDs older than ~24h |
| 504 timeouts under volume (>20 PUTs in <30s) | Bulk filing fails | Add 0.5s delay between calls + exponential backoff (2^attempt seconds) |
| Drafts can't be threaded (no `thread_id` param) | Drafts appear standalone in Drafts folder | Tell the user to send from Drafts; Gmail auto-threads by subject + recipient |
| No `delete_draft` via MCP | Stale/wrong drafts can't be removed via tools | User deletes manually in Gmail |
| `starred` flag may also no-op via MCP | Can't programmatically star | Use direct API PUT with `{"starred": true}` body |

If the user hits a NEW failure mode, append it here so the next run doesn't repeat it.

## Inbox config

| Field | Value |
|---|---|
| Primary email | {{PRIMARY_EMAIL}} |
<!-- BEGIN: aliases_block -->
| Aliases routing into this mailbox | {{ALIASES_CSV}} |
<!-- END: aliases_block -->
| To-filter | `{{TO_FILTER}}` |
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
**Active clients** (auto-tag as `project_update` or `client_request`):
{{ACTIVE_CLIENTS_CSV}}
<!-- END: clients_block -->

<!-- BEGIN: prospects_block -->
**Active prospects** (auto-tag as `lead_nurture`):
{{PROSPECTS_CSV}}
<!-- END: prospects_block -->

**Internal team domains** (auto-tag as `internal`): {{INTERNAL_DOMAINS_CSV}}

<!-- BEGIN: internal_named_senders_block -->
**Internal named senders** (also auto-tag as `internal`): {{INTERNAL_NAMED_SENDERS_CSV}}
<!-- END: internal_named_senders_block -->

<!-- BEGIN: voice_cues_block -->
**Voice cues** — phrases that boost confidence in `client_request` / `lead_nurture` for this brand:
{{VOICE_CUES_CSV}}
<!-- END: voice_cues_block -->

## Unipile call pattern (read this first)

### Listing emails — `mcp__unipile__list_emails`

```
mcp__unipile__list_emails
  account_id: {{UNIPILE_ACCOUNT_ID}}
  meta_only: true       # MANDATORY — without this, large mailboxes return 100K+ chars
  limit: 10             # MANDATORY — keep small. 10 is enough for daily triage. Bump to 20 only if justified.
  after: "<ISO 8601 with .000Z>"   # MUST include milliseconds. Bare `Z` will fail with "Expected union value".
  folder: "INBOX"       # Gmail uses uppercase. (M365 uses sentence-case "Inbox".)
```

Generate the timestamp in shell so the format is right:
```bash
TZ='UTC' date -u -v-24H +'%Y-%m-%dT%H:%M:%S.000Z'
# 2026-04-29T18:14:00.000Z
```

### Reading a full body — `mcp__unipile__get_email`

`meta_only: true` returns a snippet, not the full body. When drafting a reply or doing deeper classification, call `get_email` for the specific message ID.

### Why these constraints

`meta_only: true` + `limit: 10` keep responses under ~10K chars. Drop either and a busy mailbox blows past 100K chars and breaks the response.

## Triage flow

### Step 1 — Fetch (excludes starred — see flag semantics below)

Unipile `list_emails` with the account ID and To-filter above, last 24h, limit 10. Then **filter out any result with `is_starred: true`** — Unipile doesn't expose a `-is:starred` query param, so do it post-fetch on the metadata.

If after filtering you have nothing left, that's fine — surface "0 unflagged emails in window" and stop.

**Why:** starred = "the user (or a previous run) has opted this email out of triage". See "Flag semantics" below.

### Step 2 — Pre-classify (no LLM, first match wins)

Apply these in order, fast path before any LLM call:

- **Calendar subject patterns** → `events_calendar`
  - `^Accepted:`, `^Declined:`, `^Tentative:`, `^Proposed new time:`
  - `^Invitation:`, `^Updated invitation:`, `^Canceled:` / `^Cancelled:`
  - `\bRSVP\b`, `\bcalendar invite\b`
- **Calendar sender addresses** → `events_calendar`
  - `calendar-notification@google.com`, `noreply@google.com`, `outlook.live.com`, `calendar.google.com`
- **Internal domains** → `internal`: {{INTERNAL_DOMAINS_CSV}}
- **No-reply prefixes** → `platform_notifications`: `noreply@`, `no-reply@`, `notifications@`
- **Common platform domains** → `platform_notifications`: `stripe.com`, `github.com`, `slack.com`, `notion.so`, `vercel.com`, `zoom.us`, `calendly.com`, `apple.com`, `figma.com`, `loom.com`, `hubspot.com`, `mailchimp.com`, `sendgrid.net`, `amazonaws.com`, `microsoft.com`, `office365.com`

If the sender is on the user's `important-senders` list, bump urgency one level (never downgrade).

### Step 3 — Local sender match

<!-- BEGIN: clients_block -->
Match sender domain or address against the active clients list. Match → classify as `project_update` or `client_request` based on subject cues.
<!-- END: clients_block -->
<!-- BEGIN: prospects_block -->
Match against active prospects → classify as `lead_nurture`.
<!-- END: prospects_block -->

This step skips the LLM for known senders.

### Step 4 — LLM classify the rest

Use the Email Classifier prompt (see "Email Classifier prompt" below). Returns:

```json
{
  "category": "[Category Name]",
  "confidence": 0.95,
  "reasoning": "Brief explanation",
  "key_points": ["Main point 1", "Main point 2"],
  "urgency_level": 3,
  "requires_action": true
}
```

### Step 5 — Per-email summary

For each email, produce:

```
From — [name]
Urgency — [1-5]
Summary — [1-2 sentence summary of latest email]
Required Action — [1 sentence on what's required]
Part of a Thread — [Yes/No]
Thread Context — [if Yes, summary of full conversation]
```

### Step 6 — Draft replies (where the routing table says draft = ✓)

**`mcp__unipile__create_email_draft` is broken (returns 404).** Use the user's native Gmail MCP `create_draft` tool instead. If they don't have one connected, fall back to the direct Gmail API via a wrapper script.

<!-- BEGIN: signature_block -->
Drafts MUST follow this 3-block structure in HTML:

1. Reply text (the new content)
2. Signature (read full HTML from `{{SIGNATURE_FILENAME}}` in this skill folder)
3. Quoted thread underneath (with From/Sent/To/Subject header)

The reply text does NOT include sign-off — the signature block handles that.

**Threading caveat:** Unipile drafts can't carry a `thread_id`, so they appear standalone in the Drafts folder. The user sends from Drafts and Gmail auto-threads by subject + recipient. Tell the user this once during onboarding so they don't think threading is broken.

Never auto-send.
<!-- END: signature_block -->
<!-- IF NOT signature_block -->

Plain reply text via the user's native Gmail MCP `create_draft`. Never auto-send. (No signature configured.)
<!-- END IF -->

### Step 6.5 — Apply filing rules (LOCKED — read before Step 7)

Apply these in order BEFORE deciding to move an email. Rules 1-3 cause the email to stay in INBOX with a star; only Rule 5 actually files.

1. **Urgent → star, don't file.** If `urgency_level >= 4` (24h or sooner): star the email, leave in INBOX, surface in the brief under 🔴 URGENT. Skip Step 7 for this email.
2. **Pending reply → star, don't file.** If a draft was created this run for this thread, OR an existing draft for the same `thread_id` already lives in Drafts: star the email, leave in INBOX, surface in the brief. Skip Step 7. (Once the user sends the draft, the next run will file it.)
3. **SPAM with safe unsubscribe → trigger then file.** If category is `spam` AND the body has an `RFC 2369 List-Unsubscribe` header or visible unsubscribe link AND the sender is NOT phishing: fire HTTP GET / POST on the unsubscribe URL first, then proceed to Step 7 (move to SPAM). NEVER click unsubscribe from suspected phishing — it confirms address validity OR steals creds.
4. **Mark-as-read on file.** Every PUT in Step 7 also sets `read: true` in the same body. Filed = handled.
5. **Otherwise** → proceed to Step 7 (file).

### Step 7 — File / archive (CRITICAL: MCP is broken — use direct REST API)

`mcp__unipile__update_email` returns `{"object":"EmailUpdated"}` for every call but the move silently fails. Verified 2026-04-30. **Use the direct Unipile REST API** instead.

```
PUT https://{UNIPILE_DSN}/api/v1/emails/{email_id}?account_id={UNIPILE_ACCOUNT_ID}
Headers:
  X-API-KEY: {UNIPILE_API_KEY}
  Content-Type: application/json
Body:
  {"folders": ["<destination_label>"]}    # plural array, NOT singular `folder`
```

Gmail "archive" = the moment a non-INBOX label is added, the email is filed. The PUT above strips INBOX and adds the named label in one operation.

**Batch rules** to avoid 504s:
- Add a 0.5s delay between calls
- Retry on 504 with exponential backoff (2^attempt seconds, max 3 attempts)
- List the inbox FRESH right before any batch — IDs older than ~24h may 404

Suggested helper script (Python): expose `move_email(dsn, key, account_id, email_id, folder)` and `apply_moves(account_id, [(eid, folder), ...])`. Read creds from a secrets file (e.g. `~/.ai-secrets/.env`). Never hardcode the API key in this SKILL.md.

Use the routing table below to pick the destination label per category.

### Step 8 — Notify the user

{{NOTIFY_CHANNEL}}

<!-- BEGIN: slack_block -->
Slack DM the brief to the channel ID configured in the inbox config above.
<!-- END: slack_block -->

### Step 9 — CRM action

<!-- BEGIN: crm_block -->
For `new_lead` (and any other category flagged `crm = ✓` in the routing table): propose adding the contact to {{CRM_LABEL}} ({{CRM_DETAILS}}). ALWAYS require approval before write.
<!-- END: crm_block -->
<!-- IF NOT crm_block -->
No CRM integration configured. Skip this step.
<!-- END IF -->

## Categories

{{CATEGORIES_KEPT_LIST}}

Full definitions:

### Client Request
Existing client requesting a NEW service / automation / product. Always for something that doesn't currently exist.

### Project Update
Conversations about ongoing projects in active development. Information sharing, progress, feedback.

### Internal
From team / contractors. Domains: {{INTERNAL_DOMAINS_CSV}}.

### New Lead
First-time inquiry from a prospect who has never engaged before.

### Lead Nurture
Ongoing communications with existing prospects (engaged but not yet client). Follow-ups, re-engagement.

### Industry News
Industry / domain newsletters and digests the user has actively subscribed to.

### Events Calendar
Event invitations, webinars, meeting confirmations, RSVPs, calendar notifications.

### Support
Technical support / maintenance for previously deployed work. Includes complaints, escalations, legal/compliance.

### Platform Notifications
Automated SaaS notifications: Stripe, hosting, GitHub, Slack, Notion, Vercel, vendor T&Cs.

### Personal
Family, personal banking, mortgages, super, non-business.

### Accounting
Invoices, receipts, payment reminders, financial statements, tax documents.

### Spam
Unsolicited commercial / cold outreach selling THEIR services TO us.

### Partners
Business partners, referral partners, service providers, suppliers, recruitment agencies.

### Other
Doesn't fit any above category.

## Urgency scale (1-5)

| Level | Meaning |
|---|---|
| 1 | FYI / no action |
| 2 | Respond within a week |
| 3 | Respond within 2-3 days |
| 4 | Action within 24 hours |
| 5 | Urgent — immediate attention |

## Per-category routing table

| Category | Gmail label | Draft? | Notify? | CRM write? |
|---|---|:-:|:-:|:-:|
{{ROUTING_TABLE_ROWS}}

**Sub-label resolution:** if a `<Label>/<Sender>` sub-label exists in the user's Gmail (e.g. `Projects/Acme Corp` for an active client), prefer it. Never create new labels — file under the parent and surface a one-line note in the brief.

## Email Classifier prompt

System prompt for the LLM classifier (use when pre-classification misses):

```
You are an email categorisation assistant for {{BRAND_NAME}}.
{{BRAND_ONELINER}}

Categorise the email into ONE of these categories:
{{CATEGORIES_KEPT_LIST_INLINE}}

Return ONLY valid JSON in this format:
{
  "category": "[Category Name]",
  "confidence": 0.95,
  "reasoning": "Brief explanation",
  "key_points": ["Main point 1", "Main point 2"],
  "urgency_level": 3,
  "requires_action": true
}

Guidelines:
1. Analyse the entire email — subject and body.
2. Sender domain matters for Internal classification.
3. <0.7 confidence = uncertain; 0.7-0.9 = probable; >0.9 = clear.
4. Cold outreach selling TO us → spam, NEVER new_lead.
5. Calendar acceptances/declines → ALWAYS events_calendar.
```

User content sent per call:

```
Email From: <sender>
Subject: <subject>
Message: <body or first 2KB if very long>
```

## Output brief

```
{{BRAND_NAME}} INBOX ({{PRIMARY_EMAIL}}) — last 24h — [N emails]

🔴 URGENT (5) — N
  - [per-email summary]

🟡 NEEDS ATTENTION (4) — N
  - [per-email summary]

🟢 ROUTINE (1-3) — collapsed by category
  Client Requests: N · Project Updates: N · ...

📝 DRAFTS — M (in Gmail Drafts, threaded)
  - [draft body inline + the original message it's replying to]

📁 FOLDER MOVES PROPOSED — K
  - X to Projects · Y to Internal · Z to Notifications · ...
  Approve all / Approve [list] / Skip [list]

🆕 CRM PROPOSALS — L (New Leads only)
  - [contact details + suggested pipeline stage]
  Approve / Skip
```

## Approvals

- **Drafts saved silently** — they live in Gmail Drafts, the user reviews before sending.
- **Folder moves require approval** — propose in batch.
- **CRM writes always require approval**.

## Flag semantics (the star is the opt-out signal)

A starred email = "this email is OUT of triage". Two ways an email becomes starred:

1. **The user stars it manually** in Gmail. They've decided they want to handle it themselves and don't want triage suggesting moves on it again.
2. **This skill stars it** when filing rules 1-2 in Step 6.5 fire (urgent, or pending reply). The star marks "left in INBOX intentionally — handle when ready".

**On every subsequent run, Step 1 excludes starred emails from the fetch.** Triage will not reclassify them, will not propose moves on them, will not draft against them. They sit in INBOX until the user un-stars (after handling) — at which point the next run treats them as a normal email again.

**To star via the broken Unipile MCP:** the `starred` flag may no-op via MCP (per known quirks). Use the direct REST API:

```
PUT https://{UNIPILE_DSN}/api/v1/emails/{email_id}?account_id={UNIPILE_ACCOUNT_ID}
Body: {"starred": true}
```

Tell the user once during onboarding that starring is THEIR opt-out lever. Otherwise they may unstar things by habit and wonder why triage keeps re-suggesting moves.

## Hard rules

- NEVER send email. Drafts only.
- NEVER auto-archive without explicit approval.
- NEVER auto-mutate CRM contacts.
- NEVER classify an active-client email as `spam` or `newsletter`.
- NEVER process a starred email — it's the user's opt-out signal.

## Backlog sweep mode

When the user says "go through ALL emails", "bulk triage", or "next 50":

1. Run the standard flow on 50 emails at a time, newest first.
2. Save batch cursor to a file the user picks (default: a sibling JSON in this skill's parent folder).
3. Pull next batch with `before: <oldest_in_previous_batch_iso>`.
4. Surface batch brief + drafts + folder-move proposals per batch.
5. Wait for approval before executing folder moves.
6. Continue until inbox empty or user says stop.
