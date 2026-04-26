---
description: Lists every Helper HQ skill the user's licence entitles them to, grouped by helper, plus locked skills available on higher tiers. Reads tier from the project's `.hhq-auth.json` (V1) or the signed token (V2).
---

# /hhq:help — Helper HQ skill directory

When invoked, render a clean directory of every skill the user's licence currently entitles them to, plus a brief list of locked skills they could unlock by upgrading.

## Read auth state

Use `mcp__ccd_directory__request_directory` to get the project folder (fall back to `~/.hhq/sales-helper/` in local Claude Code CLI). Save as `<project-dir>`.

Try to read `<project-dir>/.hhq-auth.json`:

- **File exists and parses** → pull `tier` and `helpers`. If those fields are missing (auth file from an older onboarding), default `tier = "lite"` and `helpers = ["sales-helper"]`.
- **File missing or unreadable** → render the help block but prepend the activation prompt below. Use defaults `tier = "lite"`, `helpers = ["sales-helper"]` for the listing.

Do NOT call any backend API for this command — it's pure rendering. Even if the JWT has expired, you can still render the help directory based on the cached tier/helpers in the auth file.

## Determine onboarding state

If `<project-dir>/.hhq-auth.json` doesn't exist, prepend at the top of the output:

> "⚠ You haven't activated yet — say 'set me up' or 'onboard me' to get started."

Don't block the help listing — still show it. The note is just informational.

If the file exists but the user hasn't completed onboarding (you can detect this by NOT calling the backend; just trust the auth file's existence as a proxy for "started" — the help command shouldn't make API calls), no warning needed. The onboard-user skill itself handles half-finished setups.

## Render the directory

Use this exact shape. Adapt the active/locked split to the user's tier.

```
**Helper HQ** • Tier: <tier title-cased>

═══ Sales Helper (<tier title-cased>) ═══

Active skills:

• **onboard-user** — One-time setup. Captures your offer + hook + URLs (read inline to build an offer profile), ICP, 5 weighted signals, and voice samples (brand guide, articles, LinkedIn message URLs — read inline to build a voice profile you review and tune); activates your licence; kicks off your LinkedIn export (Connections + Messages). Triggers when you're new or say "set me up", "re-onboard", "start over".

• **ingest-contacts** — Imports your LinkedIn export into your master contact list and message history. Auto-detects Connections.csv and messages.csv. Triggers when you drop a CSV in chat or say "I've got my LinkedIn export", "import my contacts", "import my messages".

• **tune-voice** — Manage your voice profile after onboarding. View, edit do/dont/tone/phrases lists in plain language, add a new source (URL, paste, PDF, DOCX) and re-synthesise, or regenerate from scratch. Triggers on "tune my voice", "show my voice", "add this article to my voice", "regenerate my voice".

• **surface-next-5** — Filters your contacts by your weighted signals and surfaces the top 5 with one-line reasoning. Triggers on "get me the next 5 prospects", "who should I reach out to", "next 5".

• **research-and-draft** — Researches each prospect via their LinkedIn profile + recent posts, drafts a Greg-style opener using your voice + offer profiles, saves research and message history to the backend. Two entry points: normal flow after surface-next-5 ("let's go", "draft them"), and quick-start flow after onboarding ("let's go on quick start", "research my 5") for the up-to-5 prospects you hand-picked.

Locked — upgrade to unlock:

• **follow-up-drafter** (Pro) — Drafts timed follow-ups for prospects who haven't replied, aware of the original opener.
• **deep-research** (Pro) — Pulls company news, funding rounds, recent press alongside the LinkedIn profile read.
• **email-ingest** (Pro) — Ingests Gmail / Outlook contacts in addition to LinkedIn.
• **apollo-enrich** (Elite) — Finds NEW prospects from Apollo enrichment, not limited to your existing LinkedIn connections.
• **linkedin-discover** (Elite) — Discovers prospects matching your ICP from outside your warm network.

═══ Marketing Helper ═══

Coming in V2 — not yet available.

═══ Storage ═══

Your contact data, config, and current batch live on the Helper HQ backend. Your auth file lives in this project folder at `.hhq-auth.json` — keep it; it's how the plugin knows it's you across chats. Your per-prospect freeform notes live alongside it at `contacts/<slug>/notes.md` — open and edit those any time, the plugin reads them when drafting.

═══ Need a hand? ═══

If you're stuck on what to say, just describe what you want to do in plain language. The skills above auto-trigger from natural phrasing — you almost never need to invoke them by name.
```

## Tier-based locking rules

| Tier | Active | Locked (shown as upgrade) |
|---|---|---|
| `lite` | onboard-user, ingest-contacts, tune-voice, surface-next-5, research-and-draft | All Pro and Elite skills above |
| `pro` | All Lite skills + follow-up-drafter, deep-research, email-ingest | Elite skills above |
| `elite` | All Lite + Pro skills + apollo-enrich, linkedin-discover | (no locked skills) |

When rendering, only show the **Locked** section if there are actually locked skills for the user's tier. For an Elite user, omit the Locked section entirely.

## What this command does NOT do

- Does NOT modify any files.
- Does NOT call any backend API.
- Does NOT trigger any other skill — it's pure rendering.
- Does NOT show roadmap items beyond the Pro/Elite skills listed above. Don't speculate about V3 features.
- Does NOT pretend a locked skill is active. If the user's tier doesn't include a skill, list it as locked.
- Does NOT show internal auth values (jwt, license_key, machine_id, raw signal weights). Tier and helpers are fine to surface; secrets are not.

## Edge cases

- **No `.hhq-auth.json`** → render the help block with the activation prompt prepended. Default tier `lite`.
- **Malformed tier value** → default to `lite`, render help, show a discreet note: "_(tier read defaulted to lite — check `.hhq-auth.json`)_".
- **`helpers[]` does not include `sales-helper`** (future state) → omit the Sales Helper block. If no helpers at all, render an empty-state message: "No helpers active on your licence — visit helperhq.co to add one."
