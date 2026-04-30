---
name: email-triage
description: Use when the user wants to set up email triage for a Gmail or Outlook mailbox, build a per-mailbox triage skill, generate a triage skill from templates, or onboard a new business mailbox to triage. TRIGGERS like "set up email triage", "create a triage skill for [mailbox]", "build email triage for [client]", "scaffold a triage skill", "I want to triage my Gmail/Outlook". SKIP if the user already has a triage skill running for the target mailbox or only wants to run an existing triage (use the per-inbox skill instead).
whenToUse: "When creating a new email triage skill for a specific Gmail or Outlook mailbox. Walks the user through provider/MCP choice, mailbox config, brand context, categories, signature, then generates a fully working SKILL.md tailored to their setup."
context: inline
allowedTools: [AskUserQuestion, Read, Write, Edit, Bash, ToolSearch]
---

# Email Triage Setup

The meta-skill for generating per-mailbox email triage skills. Walks the user through provider, MCP, mailbox config, brand, categories, and signature. Outputs a fully working SKILL.md plus optional signature file at the location they choose.

**Core principle:** the skill itself contains zero personal data. All identity (brand, email, clients, voice, signature) is collected per run and rendered into the output via templates.

## Read this first — important caveats

**1. One mailbox per skill.** If the user has both a Gmail account AND an Outlook account, they need to run this skill twice. Different MCPs, different folder/label systems, different filing call patterns. Tell them this upfront.

**2. The MCP choice dictates capabilities.** Native Gmail (Cowork built-in) cannot file emails. If the user wants automatic labelling or filing, they MUST use Unipile or set up Google Cloud APIs. Same trade for Outlook — Unipile or ms365 MCP both work, but ms365 MCP gives better folder handling.

**3. Drafts only — never auto-send.** Every generated skill enforces this hard rule. Confirm with the user before generation.

**4. Bring-your-own MCP.** This skill does NOT install MCPs. If the user picks Unipile and doesn't have it connected, point them at the install docs and pause until they have it.

## Capability matrix (show this to the user)

| Provider | MCP option | Read | Draft | File / move | Best for |
|---|---|:-:|:-:|:-:|---|
| Gmail | Native (Cowork built-in) | ✓ | ✓ | ✗ | Read-only triage, simplest setup |
| Gmail | **Unipile** | ✓ | ⚠️ | ⚠️ | Full triage — MCP draft + file are broken, templates use direct API workaround |
| Gmail | Google Cloud APIs (custom) | ✓ | ✓ | ✓ | Advanced users with their own infra |
| Outlook | Unipile | ✓ | ⚠️ | ⚠️ | Cross-provider unified setup — same Unipile MCP issues as Gmail |
| Outlook | **ms365 MCP / Microsoft Graph** | ✓ | ✓ | ✓ | Best Outlook experience — fully working MCP |
| Outlook | Custom Graph API | ✓ | ✓ | ✓ | Advanced users with their own infra |

Bold = recommended. ⚠️ = templates include workarounds (direct REST API for filing, native Gmail/Outlook MCP for drafts).

**Unipile caveat (verified 2026-04-30):** the Unipile MCP's `update_email` and `create_email_draft` are broken — the templates auto-include the working direct-API and native-MCP fallbacks. The user does not need to write that code, but they DO need: a Unipile DSN + API key (read from a secrets file) for filing, AND a connected native Gmail or Outlook MCP for drafts. Confirm both before generating a Unipile-based skill.

## Run flow

This skill uses `AskUserQuestion` to collect answers, then renders one of the templates in `templates/`. If `AskUserQuestion` is not loaded, fetch it first via `ToolSearch` with `query: "select:AskUserQuestion"`.

Run the questions in order. Some answers gate later questions — for example, if the user picks Gmail + Native, skip the per-category routing question (no filing capability).

### Step 1 — Provider + MCP

Ask which mailbox + MCP combination the user is setting up. Show the capability matrix above first so they can make an informed choice.

```
Question: Which provider + MCP are you setting up triage for?
Options:
- Gmail + Native (read/draft only, no filing)
- Gmail + Unipile (full triage with label filing)
- Gmail + Google Cloud APIs (advanced custom integration)
- Outlook + Unipile (full triage with folder filing)
- Outlook + Microsoft Graph (ms365 MCP — best Outlook experience)
- Outlook + Custom Graph API (advanced custom integration)
```

Map the answer to a template file:
- Gmail + Native → `templates/gmail-native.tmpl.md`
- Gmail + Unipile → `templates/gmail-unipile.tmpl.md`
- Gmail + Google APIs → `templates/gmail-googleapi.tmpl.md`
- Outlook + Unipile → `templates/outlook-unipile.tmpl.md`
- Outlook + ms365 → `templates/outlook-ms365.tmpl.md`
- Outlook + Custom Graph → `templates/outlook-ms365.tmpl.md` (same shape, user adapts the call pattern themselves)

If the user does not have the required MCP connected, pause here and tell them how to install it before continuing.

### Step 2 — Mailbox identity

Free-text answers. Collect:

- **Primary email address** (e.g. `triage@acme.com`)
- **Skill slug** (kebab-case, suggest `<business>-email-triage` based on email domain)
- **Brand or business name** (display name used in the output skill body)
- **One-line description** of what the business does (used in the brand context block)

### Step 3 — Aliases / shared inboxes

```
Question: Are there alias or group addresses that route into the same mailbox?
Options:
- No, just the one primary address
- Yes — I'll list them
```

If yes, free-text the alias list. These become the To-filter so triage only picks up mail addressed to those addresses (avoids triaging mail bcc'd from other accounts).

### Step 4 — People & active relationships

Free-text, optional. Anything the user does not have, leave blank.

- **Active clients** (comma-separated list — used to auto-tag `project_update` / `client_request`)
- **Active prospects** (comma-separated — auto-tag `lead_nurture`)
- **Internal team domains** (comma-separated — auto-tag `internal`)
- **Internal team named senders** (e.g. `nancy@`, `tania@`)
- **Voice cues** — phrases that signal `new_lead` / `client_request` for this brand (boost classifier confidence)

### Step 5 — Categories

```
Question: Which categories do you want this skill to use?
Options:
- All 14 default categories (recommended)
- Select a subset
```

Default 14: Client Request, Project Update, Internal, New Lead, Lead Nurture, Industry News, Events Calendar, Support, Platform Notifications, Personal, Accounting, Spam, Partners, Other.

If subset: multi-select which to keep. Always keep `Spam` and `Platform Notifications` (no good reason to drop these).

### Step 6 — Per-category routing

Skip this step if the user picked Gmail + Native (no filing capability).

For each kept category, ask three booleans:
- Save a draft reply by default?
- Notify user (DM/console) when matched?
- CRM write (e.g. add new contact)?

Sensible defaults to pre-fill:

| Category | Draft | Notify | CRM |
|---|:-:|:-:|:-:|
| Client Request | ✓ | ✓ | — |
| Project Update | ✓ | ✓ | — |
| Internal | — | ✓ | — |
| New Lead | ✓ | ✓ | ✓ |
| Lead Nurture | ✓ | ✓ | — |
| Industry News | — | ✓ | — |
| Events Calendar | — | ✓ | — |
| Support | ✓ | ✓ | — |
| Platform Notifications | — | — | — |
| Personal | — | ✓ | — |
| Accounting | — | ✓ | — |
| Spam | — | — | — |
| Partners | ✓ | ✓ | — |
| Other | — | ✓ | — |

For Gmail + Unipile and Gmail + Google APIs: also collect the Gmail label name for each category (default = capitalised category name).
For Outlook variants: collect the destination folder name. Folder IDs (M365 immutable IDs) get filled in later via discovery — note this in the generated skill.

### Step 7 — Signature

```
Question: How do you want to handle the email signature?
Options:
- I have one ready, I'll paste the HTML
- I'll paste a plain-text signature, build me a basic HTML version
- Skip — I don't want a signature on drafts
- Help me build one from scratch
```

If "build from scratch": collect name, role/title, brand name, website, phone (optional), socials (optional). Render `templates/signature.tmpl.html` with those values.

Save the final signature as `signature.html` next to the generated SKILL.md. The generated skill body references it.

### Step 8 — Notifications

```
Question: Where should the skill send the triage brief?
Options:
- Slack DM to me (I'll provide the channel ID)
- Console output only (no Slack)
- Both
- A different channel (paste channel ID for now, we can add Discord/Teams later)
```

If Slack chosen: collect channel ID. Generated skill uses Slack MCP if available, otherwise notes it as TODO.

### Step 9 — CRM integration

Skip if user said no CRM writes in Step 6.

```
Question: Which CRM do you want triage to write to?
Options:
- GoHighLevel (GHL)
- HubSpot
- Salesforce
- Notion
- Other / custom
- None
```

For GHL: collect sub-account name + location ID + default pipeline name for new leads.
For others: note as a TODO in the generated skill — the user will fill in the call pattern.

### Step 10 — Output path

```
Question: Where should the generated skill be saved?
Options:
- Global Claude Code skills folder (~/.claude/skills/<slug>/)
- This project's .claude/skills/ folder
- Custom path (user types it)
```

Confirm path before writing.

## Generation

Once all answers are collected:

1. **Read the chosen template** from `templates/<provider>-<mcp>.tmpl.md`.
2. **Replace placeholders** using the answer map (see "Placeholder reference" below).
3. **Trim categories** the user dropped — remove the matching rows from the routing table and the corresponding lines in the categories list.
4. **Trim optional blocks** — each template has `<!-- BEGIN: <block_name> -->` / `<!-- END: <block_name> -->` markers. Keep blocks whose condition is true; remove the entire block (markers + contents) otherwise.
5. **Write `SKILL.md`** to the user's chosen output path.
6. **Write `signature.html`** if the user provided/built one.
7. **Print test instructions** — show the user how to invoke the new skill, and remind them of the hard rule (drafts only, never auto-send).

After write, ALWAYS:
- Tell the user the file path of the new SKILL.md.
- Tell the user one example trigger phrase they can use to test it (e.g. "triage acme" if slug is `acme-email-triage`).
- Suggest running it with the inbox empty first to verify the MCP connection before letting it loose on real mail.

If the user has another mailbox to set up (Step 1 indicated multi-mailbox), offer to run the flow again now.

## Placeholder reference

Every template uses these placeholders. Replace them all when rendering.

| Placeholder | Source | Example |
|---|---|---|
| `{{SKILL_SLUG}}` | Step 2 | `acme-email-triage` |
| `{{BRAND_NAME}}` | Step 2 | `Acme` |
| `{{BRAND_ONELINER}}` | Step 2 | `B2B SaaS for compliance teams.` |
| `{{PRIMARY_EMAIL}}` | Step 2 | `triage@acme.com` |
| `{{ALIASES_CSV}}` | Step 3 | `accounts@acme.com, hello@acme.com` (or empty) |
| `{{TO_FILTER}}` | Derived from primary + aliases | `to:triage@acme.com OR to:accounts@acme.com` |
| `{{ACTIVE_CLIENTS_CSV}}` | Step 4 | `Foo Co, Bar Inc` (or empty) |
| `{{PROSPECTS_CSV}}` | Step 4 | `Beta Corp` (or empty) |
| `{{INTERNAL_DOMAINS_CSV}}` | Step 4 | `acme.com, acme.io` |
| `{{INTERNAL_NAMED_SENDERS_CSV}}` | Step 4 | `nancy@, tania@` (or empty) |
| `{{VOICE_CUES_CSV}}` | Step 4 | `compliance audit, ISO 27001, SOC 2` |
| `{{CATEGORIES_KEPT_LIST}}` | Step 5 | bulleted markdown of kept categories |
| `{{ROUTING_TABLE_ROWS}}` | Step 6 | rendered markdown table rows |
| `{{SIGNATURE_FILENAME}}` | Step 7 | `signature.html` or `none` |
| `{{NOTIFY_CHANNEL}}` | Step 8 | `Slack DM (C1234ABCD)` or `console` |
| `{{CRM_LABEL}}` | Step 9 | `GoHighLevel` or `none` |
| `{{CRM_DETAILS}}` | Step 9 | `GHL location ID: abc123, default pipeline: 06 Cold Outreach Leads` |
| `{{UNIPILE_ACCOUNT_ID}}` | Step 1 sub-question (Unipile only) | `67LqKj...` |
| `{{MS365_ACCOUNT}}` | Step 1 sub-question (ms365 only) | `triage@acme.com` (default account) |

## Optional block markers

Each template wraps optional sections with HTML comments:

```
<!-- BEGIN: aliases_block -->
... only kept if {{ALIASES_CSV}} is non-empty ...
<!-- END: aliases_block -->
```

When rendering, drop the entire block (markers and contents) if the condition is false. Conditions:

| Block | Keep if |
|---|---|
| `aliases_block` | `{{ALIASES_CSV}}` non-empty |
| `clients_block` | `{{ACTIVE_CLIENTS_CSV}}` non-empty |
| `prospects_block` | `{{PROSPECTS_CSV}}` non-empty |
| `voice_cues_block` | `{{VOICE_CUES_CSV}}` non-empty |
| `crm_block` | `{{CRM_LABEL}}` is not `none` |
| `slack_block` | `{{NOTIFY_CHANNEL}}` includes Slack |
| `signature_block` | `{{SIGNATURE_FILENAME}}` is not `none` |
| `internal_named_senders_block` | `{{INTERNAL_NAMED_SENDERS_CSV}}` non-empty |

## Trigger this skill

The skill description above auto-triggers when a user says any of the trigger phrases. To run it manually, reference it by name (`/email-triage`) or ask Claude to "set up email triage for [mailbox]".

## Hard rules — bake into every generated skill

These are non-negotiable. The templates already contain them. Do not let the user toggle them off.

- NEVER send email. Drafts only.
- NEVER auto-archive without explicit approval.
- NEVER auto-mutate CRM contacts without approval.
- NEVER classify a known active client's mail as `spam` or `newsletter`.
- NEVER process a starred (Gmail) or flagged (Outlook) email — that's the user's per-email opt-out signal.

## Flag semantics — bake into every generated skill

Every template ships with this behaviour and it should be explained to the user during setup:

- **Star (Gmail) or Flag (Outlook) = "leave this alone"**. The skill's Step 1 fetch excludes starred/flagged emails. The skill's filing rules (Step 6.5 / 7.5 in the templates) auto-star/flag when an email is urgent or has a pending reply, so it stays in INBOX as a visual cue.
- The user can also star/flag manually at any time to opt an email out of future runs.
- To re-process an email later, the user un-stars / un-flags it.

This pattern is what stops triage from re-suggesting moves on the same emails every run after the user has decided to keep them in INBOX.

When walking through Step 7 (signature) or Step 10 (output), make sure the user understands this — otherwise they may unstar things by habit and wonder why triage keeps re-proposing moves.

## When NOT to use this skill

- The user already has a working triage for the target mailbox — just run that one.
- The user wants a one-off pass over their inbox without saving anything reusable — run an ad-hoc triage instead.
- The user's mailbox is on a provider not covered (e.g. Yahoo Mail, ProtonMail). Tell them and stop — don't fake it with a Gmail template.
