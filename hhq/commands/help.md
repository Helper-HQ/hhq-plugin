---
description: Lists every Helper HQ skill the user has, grouped by helper. V1 — all activated users get all helpers; this is a pure rendering command and makes no backend calls.
---

# /hhq:help — Helper HQ skill directory

When invoked, render a clean directory of every Helper HQ skill, grouped by helper. V1 ships with all helpers active for every user — there are no per-helper or per-tier gates yet, so the full directory always renders.

## Read auth state

Use `mcp__ccd_directory__request_directory` to get the project folder (fall back to `~/.hhq/sales-helper/` in local Claude Code CLI). Save as `<project-dir>`.

Try to read `<project-dir>/.hhq-session.json` (per-project session auth, v0.11+). If missing, also check the legacy `<project-dir>/.hhq-auth.json` so users on pre-v0.11 installs aren't shown the activation prompt incorrectly.

Do NOT call any backend API for this command — it's pure rendering. Even if the JWT has expired, the help directory still renders.

## Determine onboarding state

If neither `<project-dir>/.hhq-session.json` nor legacy `<project-dir>/.hhq-auth.json` exists, prepend at the top of the output:

> "⚠ This project isn't connected to Helper HQ yet — say `/hhq:connect` if you've already onboarded elsewhere, or `/hhq:onboard` to start fresh."

Don't block the help listing — still show it. The note is just informational.

If either exists, treat as "started" and skip the warning. The onboard-user skill itself handles half-finished setups.

## Render the directory

Use this exact shape.

```
**Helper HQ**

═══ Sales Helper ═══

Active skills:

• **onboard-user** — One-time first-time setup. Captures your offer + hook + URLs (read inline to build an offer profile), ICP, 5 weighted signals, and voice samples (brand guide, articles, LinkedIn message URLs — read inline to build a voice profile you review and tune); activates your licence on this project (one of 5 default session slots); kicks off your LinkedIn export (Connections + Messages). Triggers when you're new or say "set me up", "re-onboard", "start over".

• **connect** — Already onboarded somewhere else? Connect this Cowork project to your account without re-doing the full setup. Paste your licence key, optionally name the project, pick which campaign to pin this project to (or create a new one). Each Cowork project gets its own session — manage your 5 active session slots at `https://hhq.ngrok.dev/sessions`. Triggers on "connect", "connect this project", "I have a licence key, just connect", or `/hhq:connect`.

• **ingest-contacts** — Imports your contacts from any of four file-based sources: LinkedIn export (Connections.csv + messages.csv), generic spreadsheet (xlsx/csv/xls), CRM export (HubSpot/Salesforce/Pipedrive), or business card scan (PDF or image). Auto-detects file type and routes to the right branch. For spreadsheets / CRM exports, walks you through column mapping and stage-label mapping. For card scans, runs vision extraction with per-card review for misreads or gaps. Triggers when you drop any of those files in chat or say "I've got my LinkedIn export", "import my contacts", "import my spreadsheet", "I have a CRM export", "scan my cards".

• **sync-gmail** — Pulls your last 30 days of Gmail correspondents through Claude's Gmail connector, builds a per-correspondent digest from message headers (never bodies), and folds it into Helper HQ — refreshes "last messaged" data on contacts you already have, advances the ones you're actively emailing back and forth with to "In conversation" stage, and creates Lead-stage contacts for new bidirectional correspondents. Triggers on "sync my gmail", "pull my gmail contacts", "refresh from gmail", "ingest gmail".

• **review-imports** — Walks you through pending fuzzy-match items in your import review queue — incoming contacts that look similar to ones you already have (same name + company) but neither email nor LinkedIn URL confirmed they're the same person. For each one: shows existing vs incoming side by side, you say merge / new / skip. Mostly populated by business card scans and CRM imports — LinkedIn-only users will rarely see anything in the queue. Triggers on "review my imports", "show review queue", "any contacts to review".

• **tune-voice** — Manage your voice profile after onboarding. View, edit do/dont/tone/phrases lists in plain language, add a new source (URL, paste, PDF, DOCX) and re-synthesise, or regenerate from scratch. Triggers on "tune my voice", "show my voice", "add this article to my voice", "regenerate my voice".

• **surface-next-5** — Filters your contacts by your weighted signals and surfaces the top 5 with one-line reasoning. Triggers on "get me the next 5 prospects", "who should I reach out to", "next 5".

• **research-and-draft** — Researches each prospect via their LinkedIn profile + recent posts, drafts a Greg-style opener using your voice + offer profiles, saves research and message history to the backend. Two entry points: normal flow after surface-next-5 ("let's go", "draft them"), and quick-start flow after onboarding ("let's go on quick start", "research my 5") for the up-to-5 prospects you hand-picked.

═══ Marketing Helper ═══

Active skills:

• **offer-review** — Deep guided walkthrough to define your offer for a campaign — one-sentence offer, hook, outcomes, proof points, differentiation, pricing band, common objections, source URLs read inline. Saves a structured `offer_profile` Sales Helper uses when drafting openers. Run any time to refresh — V1 stores one active offer; running again replaces it (overwrite gate confirms first). Triggers on "review my offer", "offer review", "offer discovery", "redo my offer", "tighten my offer".

• **icp-discovery** — Deep guided walkthrough to define your ideal customer profile — industries, roles, size/stage, geography, triggers, pain points, disqualifiers, example client LinkedIn URLs read inline. Saves a structured `icp_profile` Sales Helper uses when ranking prospects. V1 stores one active ICP; running again replaces it. Triggers on "discover my ICP", "define my ICP", "review my ICP", "who's my ideal customer".

The two skills are also offered inline during onboarding — pick "deep" at the offer or ICP step to run them then instead of the quick version.

═══ Storage ═══

Your contact data, config, and current batch live on the Helper HQ backend. Each Cowork project has its own `.hhq-session.json` (this project's session token, one of 5 default active sessions per licence) and `.hhq-campaign.json` (pinning the project to a specific campaign). Manage your sessions across all projects at `https://hhq.ngrok.dev/sessions`. Your per-prospect freeform notes live in the project folder at `contacts/<slug>/notes.md` — open and edit those any time, the plugin reads them when drafting.

═══ Need a hand? ═══

If you're stuck on what to say, just describe what you want to do in plain language. The skills above auto-trigger from natural phrasing — you almost never need to invoke them by name.
```

## What this command does NOT do

- Does NOT modify any files.
- Does NOT call any backend API.
- Does NOT trigger any other skill — it's pure rendering.
- Does NOT speculate about future helpers or skills not listed above.
- Does NOT show internal auth values (jwt, license_key, machine_id, raw signal weights).

## Edge cases

- **No `<project-dir>/.hhq-session.json` and no legacy `<project-dir>/.hhq-auth.json`** → render the help block with the activation prompt prepended.
- **Auth file present but malformed** → render the help block as normal; auth contents don't affect the listing in V1.
