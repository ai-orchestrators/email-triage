---
name: {{SKILL_SLUG}}
description: Triage and classify emails for the {{BRAND_NAME}} mailbox ({{PRIMARY_EMAIL}} — Outlook via Microsoft Graph / ms365 MCP). Use when the user says "triage {{BRAND_NAME}}", "what's in {{BRAND_NAME}}", "check {{BRAND_NAME}} email", "any new leads", or any prompt referencing the {{BRAND_NAME}} Outlook inbox. Pulls Inbox via ms365 MCP. Routes emails to category folders. Drafts replies. Notifies the user. Never auto-sends.
---

# {{BRAND_NAME}} Email Triage

Inbox-specific triage for **{{PRIMARY_EMAIL}}** (Microsoft 365 / Outlook).

## Provider & MCP

| Field | Value |
|---|---|
| Provider | Microsoft 365 (Outlook) |
| MCP | ms365 (Microsoft Graph) |
| Filing capability | ✓ (folder moves via `move-mail-message`) |
| Draft capability | ✓ (HTML drafts via `create-draft-email`) |
| Account | `{{MS365_ACCOUNT}}` |

**Why ms365 over Unipile for Outlook:** ms365 returns trimmed metadata fields and folder filtering works correctly. Unipile dumps full bodies and the folder filter is unreliable on M365.

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
**Active clients** (auto-tag as `project_update` or `client_request`):
{{ACTIVE_CLIENTS_CSV}}
<!-- END: clients_block -->

<!-- BEGIN: prospects_block -->
**Active prospects** (auto-tag as `lead_nurture`):
{{PROSPECTS_CSV}}
<!-- END: prospects_block -->

**Internal team domains** (auto-tag as `internal`): {{INTERNAL_DOMAINS_CSV}}

<!-- BEGIN: internal_named_senders_block -->
**Internal named senders**: {{INTERNAL_NAMED_SENDERS_CSV}}
<!-- END: internal_named_senders_block -->

<!-- BEGIN: voice_cues_block -->
**Voice cues** — phrases that boost confidence in `client_request` / `lead_nurture`:
{{VOICE_CUES_CSV}}
<!-- END: voice_cues_block -->

## ms365 MCP call patterns

### Fetch inbox
```
mcp__ms365__list-mail-folder-messages
  mailFolderId: inbox
  top: 50
  select: id,subject,from,toRecipients,receivedDateTime,bodyPreview,isRead,hasAttachments,conversationId
  orderby: receivedDateTime desc
```

If response is large (>25K chars typical for 50 emails with bodyPreview), pipe through `jq` to TSV before reading.

### Read full body (when bodyPreview isn't enough)
```
mcp__ms365__get-mail-message
  messageId: <id>
```

### Move to folder
```
mcp__ms365__move-mail-message
  messageId: <id>
  destinationId: <folder_provider_id>
```

### Create draft (HTML)
```
mcp__ms365__create-draft-email
  body: { contentType: "html", content: "<full HTML reply>" }
  toRecipients: [...]
  subject: "Re: <original>"
```

**WARNING:** `mcp__ms365__reply-mail-message` SENDS — never use it. There's no native `createReply` call; compose threaded body manually with quoted text + signature.

## Triage flow

### Step 1 — Fetch (excludes flagged — see flag semantics below)

ms365 `list-mail-folder-messages` with `mailFolderId: inbox`, last 24h, top 50, ordered by `receivedDateTime desc`, **filtered to exclude flagged emails**:

```
$filter=isRead eq false and flag/flagStatus ne 'flagged'
```

**Why:** a flagged email = "the user (or a previous run) has opted this email out of triage". See "Flag semantics" below.

### Step 2 — Pre-classify (no LLM, first match wins)

- **Calendar subject patterns** → `events_calendar`
  - `^Accepted:`, `^Declined:`, `^Tentative:`, `^Proposed new time:`
  - `^Invitation:`, `^Updated invitation:`, `^Canceled:` / `^Cancelled:`
  - `\bRSVP\b`, `\bcalendar invite\b`
- **Calendar sender addresses** → `events_calendar`
  - `calendar-notification@google.com`, `noreply@google.com`, `outlook.live.com`, `calendar.google.com`
- **Internal domains** → `internal`: {{INTERNAL_DOMAINS_CSV}}
- **No-reply prefixes** → `platform_notifications`: `noreply@`, `no-reply@`, `notifications@`
- **Common platform domains** → `platform_notifications`: `stripe.com`, `github.com`, `slack.com`, `notion.so`, `vercel.com`, `zoom.us`, `calendly.com`, `apple.com`, `microsoft.com`, `office365.com`

If sender is on the user's `important-senders` list, bump urgency one level (never downgrade).

### Step 3 — Local sender match

<!-- BEGIN: clients_block -->
Match sender against active clients → classify as `project_update` / `client_request` based on subject cues.
<!-- END: clients_block -->
<!-- BEGIN: prospects_block -->
Match against active prospects → classify as `lead_nurture`.
<!-- END: prospects_block -->

### Step 4 — LLM classify the rest

Use the Email Classifier prompt below. Returns:

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

```
From — [name]
Urgency — [1-5]
Summary — [1-2 sentence summary of latest email]
Required Action — [1 sentence]
Part of a Thread — [Yes/No]
Thread Context — [if Yes, summary of full conversation]
```

### Step 6 — Draft replies (where the routing table says draft = ✓)

<!-- BEGIN: signature_block -->
Drafts MUST follow this 3-block HTML structure:

1. **Reply text** (the new content)
2. **Signature** (full HTML from `{{SIGNATURE_FILENAME}}` in this skill folder)
3. **Quoted thread underneath** (with From/Sent/To/Subject header)

Template:
```html
<div>
<p>[reply paragraphs]</p>
</div>

[INSERT FULL CONTENTS OF {{SIGNATURE_FILENAME}}]

<br>
<div style="border-top: 1px solid #ccc; padding-top: 10px; margin-top: 10px;">
<p><b>From:</b> [Sender Name] &lt;[sender email]&gt;<br>
<b>Sent:</b> [date in human format]<br>
<b>To:</b> [recipients]<br>
<b>Subject:</b> [original subject]</p>
</div>
<blockquote style="margin: 0 0 0 0.8ex; border-left: 1px solid #ccc; padding-left: 1ex;">
[original body, HTML-escaped, line breaks preserved with <br>]
</blockquote>
```

The reply text does NOT include sign-off — the signature handles that. Save via `mcp__ms365__create-draft-email` with `contentType: "html"`. Never auto-send.
<!-- END: signature_block -->

### Step 7 — Folder discovery (run once during setup)

Outlook folder IDs are immutable Graph IDs that look like `AQMkAD...`. To collect them for the routing table below:

```
mcp__ms365__list-mail-folders
  top: 50
```

Find each folder by `displayName`, copy the `id` field, and paste it into the routing table below. Do this once during initial setup, not per triage run.

If a folder doesn't exist yet, the user must create it in Outlook first (or this skill can fall back to leaving it in Inbox and just classifying without moving).

### Step 7.5 — Apply filing rules (LOCKED — read before Step 8)

Apply in order BEFORE deciding to move. Rules 1-3 cause the email to stay in INBOX with a flag; only Rule 5 actually files.

1. **Urgent → flag, don't file.** If `urgency_level >= 4`: flag the email via `mcp__ms365__update-mail-message` with body `{"flag": {"flagStatus": "flagged"}}`. Leave in INBOX. Surface in the brief under 🔴 URGENT. Skip Step 8.
2. **Pending reply → flag, don't file.** If a draft was created this run for this thread, OR an existing draft for the same `conversationId` already lives in Drafts: flag and leave in INBOX. Surface. Skip Step 8.
3. **SPAM with safe unsubscribe → trigger then file.** If category is `spam` AND the body has an `RFC 2369 List-Unsubscribe` header or visible unsubscribe link AND the sender is NOT phishing: fire HTTP GET / POST on the unsubscribe URL first, then proceed to Step 8 (move to Junk Email). NEVER click unsubscribe from suspected phishing.
4. **Mark-as-read on move.** Every move in Step 8 is preceded by `mcp__ms365__update-mail-message` with `{"isRead": true}`.
5. **Otherwise** → proceed to Step 8 (move to category folder).

### Step 8 — Move to category folder

For each email passing Rule 5 above, look up the destination folder in the routing table and call `mcp__ms365__move-mail-message`.

Sequence per email: `update-mail-message {isRead: true}` → `move-mail-message {destinationId}`.

Batch all moves per category at end of triage so the user can approve in groups.

### Step 9 — Notify the user

{{NOTIFY_CHANNEL}}

<!-- BEGIN: slack_block -->
Slack DM the brief to the channel ID configured in the inbox config above.
<!-- END: slack_block -->

### Step 10 — CRM action

<!-- BEGIN: crm_block -->
For `new_lead` (and any other category flagged `crm = ✓`): propose adding the contact to {{CRM_LABEL}} ({{CRM_DETAILS}}). ALWAYS require approval before write.
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
Conversations about ongoing projects in active development.

### Internal
From team / contractors. Domains: {{INTERNAL_DOMAINS_CSV}}.

### New Lead
First-time inquiry from a prospect who has never engaged before.

### Lead Nurture
Ongoing communications with existing prospects (not yet client).

### Industry News
Industry / domain newsletters the user actively subscribes to.

### Events Calendar
Event invitations, webinars, meeting confirmations, RSVPs.

### Support
Technical support for previously deployed work. Complaints, escalations.

### Platform Notifications
Automated SaaS notifications: Stripe, hosting, GitHub, Slack, etc.

### Personal
Family, personal banking, non-business.

### Accounting
Invoices, receipts, payment reminders, tax docs.

### Spam
Unsolicited commercial / cold outreach selling TO us.

### Partners
Business partners, referral partners, suppliers, recruitment.

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

Folder ID column gets filled in during the discovery step (Step 7). Until then, skill falls back to classifying-only without moving.

| Category | Outlook folder | Folder ID (fill in via discovery) | Draft? | Notify? | CRM write? |
|---|---|---|:-:|:-:|:-:|
{{ROUTING_TABLE_ROWS}}

## Email Classifier prompt

System prompt:

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
1. Analyse the entire email — subject and body.
2. Sender domain matters for Internal classification.
3. <0.7 confidence = uncertain; 0.7-0.9 = probable; >0.9 = clear.
4. Cold outreach selling TO us → spam, NEVER new_lead.
5. Calendar acceptances/declines → ALWAYS events_calendar.
```

User content per call:

```
Email From: <sender>
Subject: <subject>
Message: <body or first 2KB if long>
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

📝 DRAFTS — M (in Outlook Drafts, threaded)
  - [draft body inline + the original message it's replying to]

📁 FOLDER MOVES PROPOSED — K
  - X to Project Updates · Y to Internal · Z to Platform notifications · ...
  Approve all / Approve [list] / Skip [list]

🆕 CRM PROPOSALS — L (New Leads only)
  - [contact details + suggested pipeline stage]
  Approve / Skip
```

## Approvals

- **Drafts saved silently** — live in Outlook Drafts, user reviews before sending.
- **Folder moves require approval** — propose in batch.
- **CRM writes always require approval**.

## Flag semantics (the flag is the opt-out signal)

A flagged email = "this email is OUT of triage". Two ways an email becomes flagged:

1. **The user flags it manually** in Outlook (right-click → Flag, or the flag icon). They want to handle it themselves and don't want triage suggesting moves on it again.
2. **This skill flags it** when filing rules 1-2 in Step 7.5 fire (urgent, or pending reply). The flag marks "left in INBOX intentionally — handle when ready".

**On every subsequent run, Step 1 excludes flagged emails from the fetch** via `flag/flagStatus ne 'flagged'`. Triage will not reclassify, will not propose moves, will not draft against flagged emails. They sit in INBOX until the user un-flags (after handling) — at which point the next run treats them as a normal email again.

**To flag via ms365:**
```
mcp__ms365__update-mail-message
  messageId: <id>
  body: {"flag": {"flagStatus": "flagged"}}
```

To clear a flag (rarely needed — the user usually does this manually): `{"flag": {"flagStatus": "notFlagged"}}` or `{"flag": {"flagStatus": "complete"}}`.

Tell the user once during onboarding that flagging is THEIR opt-out lever.

## Hard rules

- NEVER send email. Drafts only. (`reply-mail-message` SENDS — never call it.)
- NEVER auto-archive without explicit approval.
- NEVER auto-mutate CRM contacts.
- NEVER classify an active-client email as `spam` or `newsletter`.
- NEVER process a flagged email — it's the user's opt-out signal.

## Backlog sweep mode

When the user says "go through ALL emails", "bulk triage", or "next 50":

1. Run the standard flow on 50 emails at a time, newest first.
2. Save batch cursor to a file the user picks.
3. Pull next batch via `filter: receivedDateTime lt '<oldest_in_previous_batch>'`.
4. Surface batch brief + drafts + folder-move proposals per batch.
5. Wait for approval before executing moves.
6. Continue until inbox empty or user says stop.
