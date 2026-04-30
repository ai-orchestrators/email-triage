---
name: {{SKILL_SLUG}}
description: Triage and classify emails for the {{BRAND_NAME}} mailbox ({{PRIMARY_EMAIL}} — Gmail via Google Cloud APIs / custom integration). Use when the user says "triage {{BRAND_NAME}}", "what's in {{BRAND_NAME}}", "check {{BRAND_NAME}} email". Surfaces drafts for approval. Never auto-sends.
---

# {{BRAND_NAME}} Email Triage

Inbox-specific triage for **{{PRIMARY_EMAIL}}** using a custom Google Cloud APIs integration.

## Provider & MCP

| Field | Value |
|---|---|
| Provider | Gmail (Google Workspace) |
| Integration | Google Cloud Gmail API (custom) |
| Filing capability | ✓ (label add/remove) |
| Draft capability | ✓ |

**Setup prerequisites (the user must have these):**
- A Google Cloud project with the Gmail API enabled
- An OAuth client for the user's Google account
- A long-lived refresh token stored securely (env var or secrets manager)
- A Python or Node script that wraps the Gmail API calls (the skill calls these via `Bash`)

If any of these are missing, pause this skill and tell the user to set them up. They could also switch to Unipile (no setup beyond the MCP install) or the native Cowork Gmail MCP (read/draft only).

## Inbox config

| Field | Value |
|---|---|
| Primary email | {{PRIMARY_EMAIL}} |
<!-- BEGIN: aliases_block -->
| Aliases routing into this mailbox | {{ALIASES_CSV}} |
<!-- END: aliases_block -->
| Script wrapper | `<USER_FILLS_IN_PATH>` (path to their Gmail API wrapper script) |
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

## Gmail API call patterns (via wrapper script)

The user's wrapper script should expose these subcommands. Example skeleton (Python):

```python
# wrapper.py — user owns and maintains this
import sys, json
from googleapiclient.discovery import build
from google.oauth2.credentials import Credentials

CREDS = Credentials(...)  # load from secrets manager

def list_unread(query, max_results=50):
    svc = build('gmail', 'v1', credentials=CREDS)
    return svc.users().messages().list(userId='me', q=query, maxResults=max_results).execute()

def get_message(mid):
    svc = build('gmail', 'v1', credentials=CREDS)
    return svc.users().messages().get(userId='me', id=mid, format='full').execute()

def add_label(mid, label_name):
    # ... lookup label_id, then modify
    pass

def remove_inbox(mid):
    # remove INBOX label = archive
    pass

def create_draft(to, subject, html_body, thread_id=None):
    # build raw RFC 2822, base64-encoded
    pass

if __name__ == '__main__':
    cmd, *args = sys.argv[1:]
    print(json.dumps(globals()[cmd](*args)))
```

The skill invokes this script via `Bash`:

```bash
python3 wrapper.py list_unread "is:unread newer_than:1d to:{{PRIMARY_EMAIL}}" 50
python3 wrapper.py get_message <id>
python3 wrapper.py add_label <id> <label_name>
python3 wrapper.py remove_inbox <id>
python3 wrapper.py create_draft <to> <subject> <html_body_file> <thread_id>
```

## Triage flow

### Step 1 — Fetch (excludes starred — see flag semantics below)

Run `list_unread` with the To-filter and 24h cutoff, **excluding starred emails**. Gmail query syntax: `is:unread -is:starred newer_than:1d to:{{PRIMARY_EMAIL}}`. Pull message IDs.

**Why:** starred = "the user (or a previous run) has opted this email out of triage". See "Flag semantics" below.

### Step 2 — Pull message bodies
For each message ID, call `get_message`. Cache locally if running in batches.

### Step 3 — Pre-classify (no LLM)
Same rules:
- Calendar subjects / senders → `events_calendar`
- Internal domains → `internal`: {{INTERNAL_DOMAINS_CSV}}
- No-reply prefixes → `platform_notifications`
- Common platform domains → `platform_notifications`

### Step 4 — Local sender match
<!-- BEGIN: clients_block -->
Match against active clients → `project_update` / `client_request`.
<!-- END: clients_block -->
<!-- BEGIN: prospects_block -->
Match against prospects → `lead_nurture`.
<!-- END: prospects_block -->

### Step 5 — LLM classify
Use the Email Classifier prompt below.

### Step 6 — Per-email summary
Standard structure.

### Step 7 — Draft replies
<!-- BEGIN: signature_block -->
3-block HTML: reply + signature from `{{SIGNATURE_FILENAME}}` + quoted thread.
Save via `wrapper.py create_draft`. Never auto-send.
<!-- END: signature_block -->
<!-- IF NOT signature_block -->
Plain reply text via `wrapper.py create_draft`. No signature configured.
<!-- END IF -->

### Step 7.5 — Apply filing rules (LOCKED — read before Step 8)

Apply in order BEFORE deciding to file. Rules 1-3 cause the email to stay in INBOX with a star; only Rule 5 actually files.

1. **Urgent → star, don't file.** If `urgency_level >= 4`: call `wrapper.py add_label <id> STARRED`. Leave INBOX label intact. Surface in 🔴 URGENT. Skip Step 8.
2. **Pending reply → star, don't file.** If a draft was created this run for this thread, OR an existing draft for the same `thread_id` already lives in Drafts: star and leave in INBOX. Surface. Skip Step 8.
3. **SPAM with safe unsubscribe → trigger then file.** If category is `spam` AND the body has an `RFC 2369 List-Unsubscribe` header or visible unsubscribe link AND the sender is NOT phishing: fire HTTP GET / POST on the unsubscribe URL first, then proceed to Step 8 (file as SPAM). NEVER click unsubscribe from suspected phishing.
4. **Mark-as-read on file.** When filing, also call `wrapper.py modify <id> --remove UNREAD`.
5. **Otherwise** → proceed to Step 8 (file).

### Step 8 — File / archive
For each email passing Rule 5 above, call `wrapper.py add_label <id> <label>` then `wrapper.py remove_inbox <id>`. The label add + INBOX remove together = "label and archive" in Gmail.

### Step 9 — Notify
{{NOTIFY_CHANNEL}}

### Step 10 — CRM action
<!-- BEGIN: crm_block -->
Propose adding `new_lead` contacts to {{CRM_LABEL}} ({{CRM_DETAILS}}). Always require approval.
<!-- END: crm_block -->
<!-- IF NOT crm_block -->
No CRM integration configured.
<!-- END IF -->

## Categories

{{CATEGORIES_KEPT_LIST}}

(Definitions identical to other variants.)

## Urgency scale

| Level | Meaning |
|---|---|
| 1 | FYI / no action |
| 2 | Respond within a week |
| 3 | Respond within 2-3 days |
| 4 | Action within 24 hours |
| 5 | Urgent |

## Per-category routing table

| Category | Gmail label | Draft? | Notify? | CRM write? |
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

## Output brief

Standard format (see gmail-unipile variant).

## Flag semantics (the star is the opt-out signal)

A starred email = "this email is OUT of triage". Two ways an email becomes starred:

1. **The user stars it manually** in Gmail. They want to handle it themselves and don't want triage suggesting moves on it again.
2. **This skill stars it** when filing rules 1-2 in Step 7.5 fire (urgent, or pending reply). The star marks "left in INBOX intentionally — handle when ready".

**On every subsequent run, Step 1 excludes starred emails from the fetch** via `-is:starred` in the Gmail search query. Triage will not reclassify, will not propose moves, will not draft against starred emails. They sit in INBOX until the user un-stars (after handling) — at which point the next run treats them as a normal email again.

**To star via the Gmail API wrapper:** add the `STARRED` label.

```bash
python3 wrapper.py add_label <id> STARRED
```

Tell the user once during onboarding that starring is THEIR opt-out lever.

## Hard rules

- NEVER send email. Drafts only.
- NEVER auto-archive without approval.
- NEVER classify an active-client email as `spam` or `newsletter`.
- NEVER process a starred email — it's the user's opt-out signal.
- The Gmail API rate limit is per-user-per-second; back off on 429s.
