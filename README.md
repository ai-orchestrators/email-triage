# email-triage

A Claude Code / Cowork **meta-skill** that generates a fully-working per-mailbox email triage skill via interactive setup. Run it once for each Gmail or Outlook mailbox you want triaged; answer 10 questions; it writes a tailored `SKILL.md` you can keep, version, and run on demand.

The generated skill reads your inbox, classifies every email into 14 categories, drafts replies for the categories you flag, files everything to the right label or folder, and excludes anything you've starred or flagged. **It never sends. It never auto-archives without approval.**

## Why this exists

Most AI email-triage tutorials either (a) only work for one specific provider, (b) hardcode the author's personal categories and identity, or (c) silently break when an MCP endpoint is broken. This skill solves all three:

- **Multi-provider:** Gmail and Outlook, across 5 MCP combinations (native, Unipile, Microsoft Graph, custom APIs)
- **Personal-data-free templates:** brand, mailbox, clients, voice, signature all collected per-run
- **Production-tested:** ships with the documented workarounds for the two known-broken Unipile MCP endpoints (verified 2026-04-30)

## What you get after one setup run

A per-mailbox skill that:

- Pulls the last 24 hours of unread mail (configurable)
- Classifies each email into one of **14 categories** (Client Request, Project Update, New Lead, Lead Nurture, Internal, Industry News, Events Calendar, Support, Platform Notifications, Personal, Accounting, Spam, Partners, Other)
- Scores **urgency 1-5**
- **Drafts replies** for the categories you flagged at setup
- **Files emails** to the right Gmail label / Outlook folder
- Surfaces a **brief grouped by urgency** so you action only what matters
- **Stars / flags** anything urgent or pending a reply, leaves it in your inbox so you see it
- **Skips starred / flagged emails** on every future run — your per-email opt-out signal

## First use — what to expect

The setup flow is **interactive**. The skill asks you 10 questions, then writes the new skill at the path you choose. **Plan for 10-15 minutes the first time.**

### What to have ready before you start

1. **The email address** you want to triage (e.g. `triage@yourcompany.com`)
2. **An idea of which provider/MCP combination you'll use** — see [Provider matrix](#provider-matrix) below. The skill explains the trade-offs at Q1 if you're unsure
3. **A short list of your active clients and prospects** (free-text, just names)
4. **Your internal team's email domains** (e.g. `yourcompany.com, yourcompany.io`)
5. **An email signature** (HTML preferred; plain text also fine; the skill builds basic HTML for you if you don't have one)
6. **A Slack channel ID** (optional — only if you want briefs DM'd to you)

### The 10 questions, in order

| # | Question | What it does |
|---|---|---|
| 1 | Provider + MCP? | Picks the right template (Gmail/Outlook × Native/Unipile/API) |
| 2 | Mailbox identity | Email, skill name, brand, one-liner |
| 3 | Aliases? | Group / shared addresses routing into this mailbox |
| 4 | People & relationships | Active clients, prospects, internal team, voice cues |
| 5 | Categories | Use all 14, or trim |
| 6 | Per-category routing | Which categories should draft / notify / hit your CRM |
| 7 | Signature | Paste yours, build basic HTML, or skip |
| 8 | Notifications | Slack DM, console, or both |
| 9 | CRM | GoHighLevel, HubSpot, Salesforce, Notion, custom, or none |
| 10 | Output path | Where to save the generated skill |

After Q10 it writes your new `SKILL.md` (and `signature.html` if you provided one) and prints a test instruction.

### Run it twice if you have both Gmail and Outlook

Different MCPs, different folder/label systems, different filing call patterns. **One mailbox per skill.** The setup flow flags this upfront.

## Provider matrix

| Provider | MCP | Filing | Drafts | Best for |
|---|---|---|---|---|
| Gmail | Native (Anthropic-built) | ✅ Labels | ✅ | Read-only safety, official MCP |
| Gmail | Unipile | ⚠️ via REST workaround | ⚠️ via native MCP | Multi-account, third-party convenience |
| Gmail | Custom (Google API wrapper) | ✅ Labels | ✅ | Full control, no MCP dependency |
| Outlook | Microsoft Graph (ms365) | ✅ Folders | ✅ | M365 native |
| Outlook | Unipile | ⚠️ via REST workaround | ⚠️ via ms365 MCP | Mixed Gmail/Outlook setup |

⚠️ rows ship with the documented workarounds inside the generated skill.

## Installation

### Claude Code (CLI)

User-level (available across all projects):

```bash
git clone https://github.com/ai-orchestrators/email-triage.git ~/.claude/skills/email-triage
```

Project-level (this project only):

```bash
git clone https://github.com/ai-orchestrators/email-triage.git .claude/skills/email-triage
```

### Claude Cowork (Desktop)

Drop the `email-triage/` directory into `~/.claude/skills/`.

### First run

In any Claude Code session, type "set up email triage" or "build an email triage skill". The meta-skill walks through 10 questions, then writes the per-mailbox skill at your chosen path.

## Trigger phrases

- "set up email triage"
- "create a triage skill for [mailbox]"
- "build email triage for [client]"
- "scaffold a triage skill"
- "I want to triage my Gmail/Outlook"

## Critical guarantees

- **Never sends.** Drafts only. The generated skill has no send capability.
- **Never auto-archives without approval.** Filing rules are explicit and reviewed.
- **Per-email opt-out.** Star (Gmail) or flag (Outlook) any email to exclude it from future runs forever.

## License

MIT
