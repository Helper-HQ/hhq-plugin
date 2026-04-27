---
description: Lists every Helper HQ skill the user has, grouped by helper. V1 — all activated users get all helpers; this is a pure rendering command and makes no backend calls.
---

# /hhq:help — Helper HQ skill directory

When invoked, render a clean directory of every Helper HQ skill, grouped by helper. V1 ships with all helpers active for every user — there are no per-helper or per-tier gates yet, so the full directory always renders.

## Read auth state

Use `mcp__ccd_directory__request_directory` to get the project folder (fall back to `~/.hhq/sales-helper/` in local Claude Code CLI). Save as `<project-dir>`.

Try to read `<project-dir>/.hhq-auth.json` (purely to detect onboarding status — the listing itself doesn't depend on its contents).

Do NOT call any backend API for this command — it's pure rendering. Even if the JWT has expired, the help directory still renders.

## Determine onboarding state

If `<project-dir>/.hhq-auth.json` doesn't exist, prepend at the top of the output:

> "⚠ You haven't activated yet — say 'set me up' or 'onboard me' to get started."

Don't block the help listing — still show it. The note is just informational.

If the file exists but the user hasn't completed onboarding (you can detect this by NOT calling the backend; just trust the auth file's existence as a proxy for "started" — the help command shouldn't make API calls), no warning needed. The onboard-user skill itself handles half-finished setups.

## Render the directory

Use this exact shape.

```
**Helper HQ**

═══ Sales Helper ═══

Active skills:

• **onboard-user** — One-time setup. Captures your offer + hook + URLs (read inline to build an offer profile), ICP, 5 weighted signals, and voice samples (brand guide, articles, LinkedIn message URLs — read inline to build a voice profile you review and tune); activates your licence; kicks off your LinkedIn export (Connections + Messages). Triggers when you're new or say "set me up", "re-onboard", "start over".

• **ingest-contacts** — Imports your LinkedIn export into your master contact list and message history. Auto-detects Connections.csv and messages.csv. Triggers when you drop a CSV in chat or say "I've got my LinkedIn export", "import my contacts", "import my messages".

• **tune-voice** — Manage your voice profile after onboarding. View, edit do/dont/tone/phrases lists in plain language, add a new source (URL, paste, PDF, DOCX) and re-synthesise, or regenerate from scratch. Triggers on "tune my voice", "show my voice", "add this article to my voice", "regenerate my voice".

• **surface-next-5** — Filters your contacts by your weighted signals and surfaces the top 5 with one-line reasoning. Triggers on "get me the next 5 prospects", "who should I reach out to", "next 5".

• **research-and-draft** — Researches each prospect via their LinkedIn profile + recent posts, drafts a Greg-style opener using your voice + offer profiles, saves research and message history to the backend. Two entry points: normal flow after surface-next-5 ("let's go", "draft them"), and quick-start flow after onboarding ("let's go on quick start", "research my 5") for the up-to-5 prospects you hand-picked.

═══ Marketing Helper ═══

Active skills:

• **offer-review** — Deep guided walkthrough to define your offer for a campaign — one-sentence offer, hook, outcomes, proof points, differentiation, pricing band, common objections, source URLs read inline. Saves a structured `offer_profile` Sales Helper uses when drafting openers. Run any time to refresh — V1 stores one active offer; running again replaces it (overwrite gate confirms first). Triggers on "review my offer", "offer review", "offer discovery", "redo my offer", "tighten my offer".

• **icp-discovery** — Deep guided walkthrough to define your ideal customer profile — industries, roles, size/stage, geography, triggers, pain points, disqualifiers, example client LinkedIn URLs read inline. Saves a structured `icp_profile` Sales Helper uses when ranking prospects. V1 stores one active ICP; running again replaces it. Triggers on "discover my ICP", "define my ICP", "review my ICP", "who's my ideal customer".

The two skills are also offered inline during onboarding — pick "deep" at the offer or ICP step to run them then instead of the quick version.

═══ Storage ═══

Your contact data, config, and current batch live on the Helper HQ backend. Your auth file lives in this project folder at `.hhq-auth.json` — keep it; it's how the plugin knows it's you across chats. Your per-prospect freeform notes live alongside it at `contacts/<slug>/notes.md` — open and edit those any time, the plugin reads them when drafting.

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

- **No `.hhq-auth.json`** → render the help block with the activation prompt prepended.
- **Auth file present but malformed** → render the help block as normal; auth contents don't affect the listing in V1.
