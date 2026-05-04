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

If either exists, treat as "started" and skip the warning. The onboard-helperhq skill itself handles half-finished setups.

## Render the directory

Use this exact shape.

```
**Helper HQ**

═══ Sales Helper ═══

Active skills:

• **onboard-helperhq** — One-time first-time setup. Captures your offer + hook + URLs (read inline to build an offer profile), ICP, 5 weighted signals, and voice samples (brand guide, articles, LinkedIn message URLs — read inline to build a voice profile you review and tune); activates your licence on this project (one of 5 default session slots); kicks off your LinkedIn export (Connections + Messages). Triggers when you're new or say "set me up", "re-onboard", "start over".

• **connect** — Already onboarded somewhere else? Connect this Cowork project to your account without re-doing the full setup. Paste your licence key, optionally name the project, pick which campaign to pin this project to (or create a new one). Each Cowork project gets its own session — manage your 5 active session slots at `https://helperhq.co/sessions`. Triggers on "connect", "connect this project", "I have a licence key, just connect", or `/hhq:connect`.

• **ingest-contacts** — Imports your contacts from any of four file-based sources: LinkedIn export (Connections.csv + messages.csv), generic spreadsheet (xlsx/csv/xls), CRM export (HubSpot/Salesforce/Pipedrive), or business card scan (PDF or image). Auto-detects file type and routes to the right branch. For spreadsheets / CRM exports, walks you through column mapping and stage-label mapping. For card scans, runs vision extraction with per-card review for misreads or gaps. Triggers when you drop any of those files in chat or say "I've got my LinkedIn export", "import my contacts", "import my spreadsheet", "I have a CRM export", "scan my cards".

• **sync-gmail** — Pulls your last 30 days of Gmail correspondents through Claude's Gmail connector, builds a per-correspondent digest from message headers (never bodies), and folds it into Helper HQ — refreshes "last messaged" data on contacts you already have, advances the ones you're actively emailing back and forth with to "In conversation" stage, and creates Lead-stage contacts for new bidirectional correspondents. Triggers on "sync my gmail", "pull my gmail contacts", "refresh from gmail", "ingest gmail".

• **review-imports** — Walks you through pending fuzzy-match items in your import review queue — incoming contacts that look similar to ones you already have (same name + company) but neither email nor LinkedIn URL confirmed they're the same person. For each one: shows existing vs incoming side by side, you say merge / new / skip. Mostly populated by business card scans and CRM imports — LinkedIn-only users will rarely see anything in the queue. Triggers on "review my imports", "show review queue", "any contacts to review".

• **remap-pipeline** — Edit your pipeline stages — rename, reorder, add custom stages, or delete custom stages. Use this when the seven Helper HQ defaults (Lead, Outreach sent, In conversation, Meeting booked, Proposal sent, Customer, Not a fit) don't match how you actually sell. Defaults can be renamed and reordered but never deleted (Gmail sync's stage-advance keys off them); custom stages get a `custom_<name>` slug and slot anywhere in the funnel. Pipeline locks after editing; rename stays open any time. Triggers on "remap my pipeline", "edit my pipeline stages", "add a stage", or `/hhq:remap-pipeline`.

• **tune-voice** — Manage your voice profile after onboarding. View, edit do/dont/tone/phrases lists in plain language, add a new source (URL, paste, PDF, DOCX) and re-synthesise, or regenerate from scratch. Triggers on "tune my voice", "show my voice", "add this article to my voice", "regenerate my voice".

• **surface-next-5** — Filters your contacts by your weighted signals and surfaces the top 5 with one-line reasoning. Triggers on "get me the next 5 prospects", "who should I reach out to", "next 5".

• **research-and-draft** — Researches each prospect via their LinkedIn profile + recent posts, drafts a Greg-style opener using your voice + offer profiles, saves research and message history to the backend. Two entry points: normal flow after surface-next-5 ("let's go", "draft them"), and quick-start flow after onboarding ("let's go on quick start", "research my 5") for the up-to-5 prospects you hand-picked.

• **surface-followups** — Daily companion to surface-next-5: shows up to 10 people who actually need a reply (manual reminder due, ball in your court, stale your court, going cold). You pick one at a time — the skill live-reads the Gmail thread (bodies in context, never persisted), distils dated bullets, refreshes the rolling user-level dossier, drafts a reply in your voice referencing the conversation, then pushes the draft into Gmail as a reply-in-thread for your final pass and send. Snooze / mark-handled / skip per pick. Triggers on "follow-ups for today", "who's waiting on me", "let's do follow-ups", or `/hhq:followups`.

• **log-touch** — Quick capture for offline interactions Gmail sync can't see — phone calls, meetings, in-person catch-ups, voicemails, SMS. Free-form: "log a call with Sarah — wants pricing for 50 seats, follow up Tuesday". Logs the touch, optionally sets a reminder for surface-followups to pick up, appends a bullet to her conversation notes, and bumps her last_contacted_at for outreach-counting types. Triggers on "log a call with X", "talked to X", or `/hhq:log-touch`.

• **contact** — View and manage one contact's full record: rolling user-level dossier, recent conversation bullets across all your campaigns, recent manual touches, pipeline status per campaign. Edit conversationally — "change her role to VP", "add: prefers WhatsApp", "drop the bullet from April 15", "clear her dossier and start fresh". Bullets and dossier edits are auto-versioned. Triggers on "show me Sarah", "what do I know about Marcus", "edit Greg's profile", or `/hhq:contact <name>`.

• **request-gmail-access** — Submit (or check) a request for **extended Gmail access** (beta) — the prerequisite for `/hhq:connect-gmail`. Standalone path so already-onboarded users don't need to re-run full onboarding just for this one step. Asks for the Gmail address, POSTs the request, tells you what happens next (admin reviews, you get an email when approved). Idempotent — checks current state first. Triggers on "request gmail access", "I need gmail access", "submit gmail request", or `/hhq:request-gmail-access`.

• **connect-gmail** — Finish the OAuth handshake for **extended Gmail access** (beta). Run this after you receive the "your access is approved" email. Drives the Google consent screen and verifies the connection landed. Required before Admin Helper inbox features. Pre-req: submit a request via `/hhq:request-gmail-access` (or `/hhq:onboard` Phase 7.5) and wait for admin approval. Triggers on "connect my gmail to helper hq", "finish gmail connection", "my gmail access was approved", or `/hhq:connect-gmail`.

═══ Admin Helper ═══

*Requires extended Gmail access (beta opt-in via `/hhq:onboard` Phase 7.5 + `/hhq:connect-gmail`). Standard Gmail connector users see these skills listed but can't run them — the skills route you to the Connect Gmail flow if you try.*

Active skills:

• **setup-gmail-labels** — One-time setup of 5 default HHQ Gmail labels (To Do / Awaiting Reply / FYI / Notifications / Newsletters) on your Gmail account, namespaced under `HHQ/`. Creates the labels in Gmail, persists the mapping. Auto-runs after `/hhq:connect-gmail` succeeds. Re-run any time to rename or disable individual labels. Required before triage-inbox and the background auto-rules can apply anything. Triggers on "set up gmail labels", "configure my labels", or `/hhq:setup-gmail-labels`.

• **triage-inbox** — Pulls your last 50 Gmail threads, AI-categorises into the 5 HHQ buckets (To Do stays in inbox; Awaiting Reply / FYI / Notifications / Newsletters get archived), presents the grouped list with sender context (prospects + customers flagged), and on one yes applies the labels + archives + marks read in bulk per the bucket settings. Your stored triage rules (see `/hhq:triage-rules`) and any custom labels run BEFORE the AI step — once you've taught a rule, it applies deterministically every run. Most newsletters / notifications are auto-handled by the background sync (every 15 min), so triage usually focuses on the To Do / Awaiting Reply / FYI judgment cases. Default UX is fast (one approval to apply all). When you say "always do X" mid-flow, the skill offers to save it as a rule. Can also run autonomously as a scheduled Cowork routine — invoke with the keyword "auto" to skip the approval gate. Falls back to a simpler 4-bucket flow if labels haven't been set up. Triggers on "triage my inbox", "clean up my inbox", "what needs my attention", or `/hhq:triage-inbox`.

• **triage-rules** — View, edit, reorder, disable, or delete the triage rules you've taught Helper HQ over time, plus manage your custom Gmail labels (anything beyond the 5 HHQ defaults like "To Be Paid" or "Investor Updates"). Rules apply BEFORE the AI step in `/hhq:triage-inbox` — deterministic, free, fast — so once you've taught one, every future run (manual or scheduled routine) applies it without asking. Most rules get added conversationally inside `/hhq:triage-inbox` ("always move missed-payment ones to To Be Paid"); this skill is for explicit management. Triggers on "show my triage rules", "edit my triage rules", "what rules do I have", "manage my custom labels", or `/hhq:triage-rules`.

• **draft-reply** — Drafts a reply to one specific Gmail thread in your voice and pushes it to Gmail as a draft for you to review + send. Identify the thread by description ("Sarah's pricing email"), Gmail URL, or thread ID — the skill searches your inbox if you describe it. Reads the thread, uses the same voice profile Sales Helper uses, references the conversation specifically, surfaces placeholders if it needs facts you didn't give it. Helper HQ never sends — you open Gmail and click send when ready. Triggers on "draft a reply to X", "reply to Sarah's email", "let's reply to that", or `/hhq:draft-reply <reference>`.

• **draft-email** — Drafts a *fresh* outbound email (with subject line) to one person and pushes it to Gmail as a new draft. Built for people who didn't come through the LinkedIn opener flow — follow-ups to non-responders, first-touch emails to people you met in person, referrals, conference contacts. Identify the recipient by name (looks them up in your contacts) or email; give a one-line purpose ("follow up after our coffee on Tuesday") and the skill drafts in your voice referencing what's already on their dossier. Same anti-LLM-tell rules as the rest of the toolkit (no em dashes, no "circling back", no "leverage"). Triggers on "email <name>", "draft an email to <name>", "write a follow-up to <name>", "send something to <name> about X", or `/hhq:draft-email <name>`.

• **manage-followups** — Surfaces threads where you sent the last message N+ days ago and no one's replied (default 5 days, override with `/hhq:manage-followups 10`). Sorted oldest-first, capped at 30. Pick one at a time — the skill reads the thread, drafts a light low-pressure follow-up in your voice referencing what you previously said, pushes as a Gmail draft for you to review + send. Skip / next / stop per pick. Broader than Sales Helper's `/hhq:followups` — covers any inbox thread, not just sales pipeline contacts. Triggers on "what am I waiting on", "manage my followups", "who haven't I heard back from", "follow up on emails", or `/hhq:manage-followups`.

═══ Marketing Helper ═══

Active skills:

• **offer-review** — Deep guided walkthrough to define your offer for a campaign — one-sentence offer, hook, outcomes, proof points, differentiation, pricing band, common objections, source URLs read inline. Saves a structured `offer_profile` Sales Helper uses when drafting openers. Run any time to refresh — V1 stores one active offer; running again replaces it (overwrite gate confirms first). Triggers on "review my offer", "offer review", "offer discovery", "redo my offer", "tighten my offer".

• **icp-discovery** — Deep guided walkthrough to define your ideal customer profile — industries, roles, size/stage, geography, triggers, pain points, disqualifiers, example client LinkedIn URLs read inline. Saves a structured `icp_profile` Sales Helper uses when ranking prospects. V1 stores one active ICP; running again replaces it. Triggers on "discover my ICP", "define my ICP", "review my ICP", "who's my ideal customer".

The two skills are also offered inline during onboarding — pick "deep" at the offer or ICP step to run them then instead of the quick version.

═══ Storage ═══

Your contact data, config, and current batch live on the Helper HQ backend. Each Cowork project has its own `.hhq-session.json` (this project's session token, one of 5 default active sessions per licence) and `.hhq-campaign.json` (pinning the project to a specific campaign). Manage your sessions across all projects at `https://helperhq.co/sessions`. Your per-prospect freeform notes live in the project folder at `contacts/<slug>/notes.md` — open and edit those any time, the plugin reads them when drafting.

═══ Need a hand? ═══

If you're stuck on what to say, just describe what you want to do in plain language. The skills above auto-trigger from natural phrasing — you almost never need to invoke them by name.
```

## What this command does NOT do

- Does NOT modify any files.
- Does NOT call any backend API.
- Does NOT trigger any other skill — it's pure rendering.
- Does NOT speculate about future helpers or skills not listed above.
- Does NOT show internal auth values (jwt, license_key, session_id, raw signal weights).

## Edge cases

- **No `<project-dir>/.hhq-session.json` and no legacy `<project-dir>/.hhq-auth.json`** → render the help block with the activation prompt prepended.
- **Auth file present but malformed** → render the help block as normal; auth contents don't affect the listing in V1.
