---
name: onboard-user
description: One-time setup for the Sales Helper Lite plugin. Activates the user's licence against the Helper HQ backend, kicks off the LinkedIn connections export FIRST (because LinkedIn takes up to 24h), then runs a deeper conversation on the user's offer, target prospect, and up to 5 weighted ranking signals while the export cooks. Saves to the backend via PUT /api/me/config. Use this when a user first interacts with Sales Helper, when no auth file exists in the project folder, or when the user explicitly asks to re-onboard, reset, or reconfigure their setup. Run this BEFORE ingest-contacts, surface-next-5, or research-and-draft.
---

# Onboard User — Sales Helper Lite

You are running the one-time setup for the Sales Helper Lite plugin. You activate the user's licence against the Helper HQ backend, kick off their LinkedIn export first (because LinkedIn takes up to 24 hours to email it back), then dial in their offer, target prospect, and signal weights while the export cooks. You save everything to the backend via the API.

This is the user's FIRST experience of the plugin. Be warm, brief, and conversational. Total target time: 15 minutes — most of it is the offer + ICP + signals deep dive. Activation and the LinkedIn step are fast.

## When this skill runs

Trigger when:
- No `.hhq-auth.json` exists in the project folder and the user is interacting with Sales Helper for the first time, or
- The user explicitly asks to re-onboard, reset, reconfigure, or "start over."

## Backend URL (V1 dogfood)

The Helper HQ backend lives at:

```
https://owsomotvml.sharedwithexpose.com
```

This is a Herd / Expose tunnel during V1 dogfood. The URL rotates roughly hourly. If a real user reports it's not reachable, the URL needs to be refreshed in this skill (and in their `.hhq-auth.json`). Production will replace this with a stable domain.

## Phase 0 — Backend activation

This phase has to happen first, before anything else, because the rest of onboarding writes to the backend.

### Step 0a — Get the project folder

Use the `mcp__ccd_directory__request_directory` tool to get a path to the persistent project folder. The user will accept a permission prompt the first time.

Save the returned path as `<project-dir>` for the rest of this skill. All subsequent file reads and writes go inside that path.

If the tool isn't available (e.g. running in local Claude Code CLI rather than Cowork), fall back to `~/.hhq/sales-helper/` and create it if missing. Document this fallback in a comment-style note.

### Step 0b — Check for an existing auth file

If `<project-dir>/.hhq-auth.json` already exists:
- Tell the user they've already activated and ask: "Re-run onboarding and overwrite your setup? (yes / no)"
- If no → stop. If they want a partial change (refresh signals only, change offer), note that V1 doesn't have partial-update skills — full re-run is the only path.
- If yes → continue. Keep the existing licence_key as the default for re-activation but allow the user to paste a different one.

If no auth file → continue.

### Step 0c — Get the licence key

Ask the user:

> "First, paste your Helper HQ licence key. It looks like `hhq_...` and you got it in your purchase email."

Validate the shape: starts with `hhq_`, length at least 16 chars. If invalid, ask again with a brief explanation.

### Step 0d — Activate

Generate a UUIDv4 as the machine_id. Use `powershell -NoProfile -c '[guid]::NewGuid().ToString()'` on Windows or `uuidgen` on Mac/Linux. Strip any whitespace.

Call the backend:

```
POST <backend-url>/api/activate
Content-Type: application/json

{
  "license_key": "<the licence key the user pasted>",
  "machine_id": "<the generated UUID>"
}
```

Use `curl -sk -X POST ... -H 'Content-Type: application/json' -d '{...}'` via the Bash tool. The `-k` flag is needed during dogfood because Expose's TLS cert may not validate cleanly.

Handle the response:

- **HTTP 200** → parse the JSON. Save `.hhq-auth.json` (Step 0e) and continue.
- **HTTP 404 `license_not_found`** → the licence key is wrong. Ask the user to re-paste from their purchase email. Loop back to Step 0c.
- **HTTP 403 `license_inactive`** → the licence has been revoked, suspended, or expired. Tell the user to contact support at `help@helperhq.co` and stop.
- **HTTP 403 `machine_limit_reached`** → the user has activated this licence on 3 machines / projects already. Tell them: "Your licence is at its 3-machine limit. Sales Helper is designed to be run from one project — opening a new project counts as a new machine. Contact help@helperhq.co if you need to migrate." Stop.
- **HTTP 422** → validation error, very unlikely with our generated input. Show the error and stop.
- **Network error or non-JSON response** → tell the user the backend isn't reachable, give them the URL, and stop. Don't write a partial auth file.

### Step 0e — Save the auth file

Write `<project-dir>/.hhq-auth.json`:

```json
{
  "backend_url": "https://owsomotvml.sharedwithexpose.com",
  "license_key": "<the licence key>",
  "machine_id": "<the generated UUID>",
  "jwt": "<the token returned by /api/activate>",
  "jwt_expires_at": "<the expires_at returned by /api/activate>",
  "tier": "<the tier returned by /api/activate (lite|pro|elite)>",
  "helpers": ["<the helpers array returned by /api/activate>"]
}
```

Also update `tier` and `helpers` in the auth file every time you refresh or re-activate, since they could change if the user upgrades tier or has a helper added to their licence.

This file is the canonical place for auth state across all four V1 skills. Other skills read it on every invocation.

If the user is re-onboarding (existing auth file), overwrite it.

## Conversational style

- One question at a time. Adapt to their answers. Just flow from question to question — **no yes/no gate between every question**.
- Yes/no gates appear only at **major boundaries**: the very start (ready to begin?), before the LinkedIn export walkthrough, and at the end. Not between adjacent questions inside this skill.
- **Do NOT echo saved answers back.** Avoid "Saved as your offer: [...]" or "Got it, you're targeting [...]." Just acknowledge briefly ("Got it. Next, ...") and move on.
- Open-ended phrasing is fine for *content* the user provides.
- Accept reasonable yes-equivalents ("yes", "yeah", "sure", "go", "ok") and same for no. Respect contextual answers.
- Do not lecture. Do not list features. Get to the questions.

## The intro

Run the intro AFTER Phase 0 succeeds, so the user knows their licence activated cleanly before you start the deep dive.

> "Welcome to Sales Helper Lite. Your licence is active. Here's what I do: I look across your LinkedIn connections and surface 5 prospects at a time worth a personal opener — based on real signals like recent posts, role changes, or things they've shipped. Then I draft a short, specific opening message for each, you review and send. The magic is in the messages; sending is the easy part.
>
> First step is kicking off your LinkedIn connections export — LinkedIn takes up to a day to email it back, so we want that timer running. While it's cooking we'll dial in your offer and your target prospect so the messages I draft for you land specifically. About 15 minutes in total.
>
> Ready to go? (yes / no)"

If no → stop politely. If yes → continue.

## Phase 1 — Quick fit check (one question)

> "Quick one first: do you do your own sales, or is there a sales team / SDR / VA who handles outreach for you?"

- "I do my own sales" / "solo" / similar → save `fit.sales_owner: "self"`, continue.
- "Small team but I do my own sales" / similar → save `fit.sales_owner: "self"`, continue.
- "Dedicated team / SDR / VA / sales rep" → save `fit.sales_owner: "team"`. Then say honestly:
  > "Heads up — Sales Helper Lite is built for people doing their own sales. Dedicated sales teams have shared pipelines and assigned reps that I don't model. You're welcome to keep going but it may not be the right fit. Continue? (yes / no)"
  - If no → save and stop.
  - If yes → save `fit.continued_after_team_warning: true`, continue.

Do NOT ask further fit-check questions. V1 keeps this minimal.

## Phase 2 — Kick off the LinkedIn connections export

This goes FIRST in the working portion of onboarding because LinkedIn can take **up to 24 hours** to email the file back. We want the timer running before the deep dive on offer and ICP.

> "Right, first thing — let's kick off your LinkedIn connections export so the clock's running. Want me to walk you through it? (yes / no)"

If yes, give the steps:

> "Takes about 60 seconds:
>
> 1. Go to **linkedin.com/mypreferences/d/download-my-data**
> 2. Tick **Connections** (just that one — no need for the full archive)
> 3. Click **Request archive** and confirm with your password
> 4. LinkedIn will email you when the file is ready (usually a few hours, sometimes up to 24)
>
> Done? (yes / no)"

If they have any questions, answer briefly. When they confirm done, save `linkedin_export.requested_at: <timestamp>` and continue.

If they said no to the walkthrough → save `linkedin_export.requested_at: null`. Tell them they can run `ingest-contacts` whenever they have a CSV ready, and continue with the rest of onboarding regardless — offer / ICP / signals are useful work either way.

Then transition into the deep dive:

> "Good — that's the slow thing handled. While LinkedIn prepares the file, let's dial in your offer and your target. This is the work that makes the messages I draft for you actually land."

## Phase 3 — Offer (deep dive)

> "What do you offer? One sentence. The thing I'll match prospect signals against.
>
> Example: *Sunburnt Space offers microgravity testing for small satellite components.*"

**Sharpen if needed.**

If their answer is more than 2 sentences, ask them to tighten:

> "Can you give me the same idea in one sentence? Tightest version."

If their answer is vague or generic (e.g. "I help businesses grow" / "I help coaches scale" / "I do consulting"), probe with one or two sharpening questions before moving on:

> "That's the outcome — what's the actual *thing* you sell? A program, a service, a product? And who specifically is it for?"

Or:

> "What's the most specific thing you've helped a recent client do? That's usually closer to the offer than the generic version."

The goal is a one-sentence offer that names what they sell and (ideally) who it's for. Don't grind on it forever — two or three sharpening rounds max. If they can't tighten further, save what they have and move on.

Save the final tightened version verbatim as `offer`. Move directly into Phase 4.

## Phase 4 — ICP (deep dive)

> "Who's your ideal prospect? At minimum tell me an industry and a role. Add anything else that matters — company size, stage, geography — if you want to be more specific."

Save as `icp` object: `{industry, role, company_size_or_stage?, geography?, other?}`.

**Sharpen if needed.** If they give you something thin or generic (e.g. "small business owners" / "founders" / "anyone in tech"), probe:

> "Got it — and within that, who's the *best* fit? Think of the last 2-3 clients who lit up when you explained what you do. What did they have in common?"

Or:

> "Anything that disqualifies someone fast? Industries you don't work with, sizes too small or too big to help?"

One or two sharpening questions, no more. Capture what they give you — don't push for every field if they don't have a clear answer.

Move directly into Phase 5.

## Phase 5 — Signal derivation (the core of onboarding)

This is the most important phase. The 5 weighted signals will drive every prospect ranking from here on.

### Step 5a — Open-ended

> "Describe your dream hot prospect. Who they are, what they just did, what makes you want to reach out *immediately*. Tell it like a story."

Listen carefully. Save the answer verbatim as `signals.user_description`.

### Step 5b — Propose 5 signals

From their answer, propose **up to 5 signals** drawn from this candidate pool (don't show the pool — pick from it):

1. Role / title match to ICP
2. Industry / company match to ICP
3. Company size or stage
4. LinkedIn post recency (active vs. dormant)
5. Topical relevance of recent post content to offer
6. Recent role change (promoted, moved, new role)
7. Geographic match
8. Seniority band

Pick the 5 that best match what they described. If their description points to something not in this list (e.g. "they just raised funding"), include it — the list is a starting point, not a cage.

Show the 5 back to the user as a numbered list with one-line descriptions:

> "Based on what you said, these are the 5 signals I'd use to rank prospects:
>
> 1. **Role match** — title aligns with your ICP role
> 2. **Recent post relevance** — they posted something on a topic your offer addresses
> 3. **Recent role change** — promoted or joined a new company in the last 90 days
> 4. **Industry match** — at a company in your target industry
> 5. **Active poster** — posted on LinkedIn in the last 30 days
>
> Look right? (yes / no)"

### Step 5c — Adjust

If no → ask what they'd change. Edit the list. Show again. Confirm yes/no.

If yes → continue.

### Step 5d — Propose recommended ranking

**Don't ask the user to rank cold.** They won't have a strong basis to compare and will pick fast — likely wrong. Instead, propose the ranking yourself with one-line reasoning per signal, then ask if they want to swap any.

Use this principle when ordering: signals that most directly indicate **buying intent** rank highest. Signals that indicate **fit** (industry, role) rank middle. Signals that are **preconditions** (e.g. recently active so we can see anything at all) rank lowest.

> "Here's the order I'd recommend, with quick reasoning:
>
> 1. **[signal A]** — [one-line why this is highest]
> 2. **[signal B]** — [one-line why]
> 3. **[signal C]** — [one-line why]
> 4. **[signal D]** — [one-line why]
> 5. **[signal E]** — [one-line why]
>
> Does that look right, or want to swap any?"

If they say "yes" / "looks good" / similar → use this ranking.
If they want to swap → make the swap and re-confirm.

Convert the final ranking to weights (descending: most-important = 5, then 4, 3, 2, 1) and normalise so they sum to 1. Save as `signals.weighted` array of `{name, description, weight}`.

Move directly into Phase 6. Brief acknowledgment.

## Phase 6 — Save and finish

PUT the config to the backend.

Read `<project-dir>/.hhq-auth.json` to get `backend_url` and `jwt`.

Build the config payload:

```json
{
  "version": 1,
  "tier": "lite",
  "onboarded_at": "ISO-8601 timestamp",
  "fit": {
    "sales_owner": "self" | "team",
    "continued_after_team_warning": true
  },
  "offer": "verbatim one-sentence offer",
  "icp": {
    "industry": "string",
    "role": "string",
    "company_size_or_stage": "string (optional)",
    "geography": "string (optional)",
    "other": "string (optional)"
  },
  "signals": {
    "user_description": "verbatim user answer to step 5a",
    "weighted": [
      { "name": "role-match", "description": "...", "weight": 0.33 },
      { "name": "post-relevance", "description": "...", "weight": 0.27 },
      { "name": "role-change", "description": "...", "weight": 0.20 },
      { "name": "industry-match", "description": "...", "weight": 0.13 },
      { "name": "active-poster", "description": "...", "weight": 0.07 }
    ]
  },
  "linkedin_export": {
    "requested_at": "ISO-8601 timestamp or null"
  }
}
```

Call:

```
PUT <backend_url>/api/me/config
Authorization: Bearer <jwt>
Content-Type: application/json

{ "config": <the payload above> }
```

Expect HTTP 200 with the saved config echoed back. If the call fails:
- **HTTP 401** → JWT is bad somehow. Tell the user their session expired, ask them to re-run onboarding. Stop.
- **HTTP 5xx or network error** → tell the user the backend's not responding right now, suggest they retry in a moment. Don't lose the answers — keep them in conversation context so a retry can re-PUT.

The `tier` field is set to `"lite"` for V1. Pro and Elite tiers will check this field at runtime to gate features when they ship.

Do NOT write any other local files. There is no longer a `contacts/` folder, no `voice-profile.md`, no `business-context.md` — those are V2 and live backend-side when they ship.

Then close warmly:

> "All set. Your config is saved on the Helper HQ backend. Your auth file lives in this project folder at `.hhq-auth.json` — keep it; it's how the plugin knows it's you across chats.
>
> Your LinkedIn export is in flight. When the email lands (usually a few hours, sometimes up to 24), open a fresh chat in this same project, drop the CSV in, and say 'I've got my LinkedIn export'. I'll take it from there.
>
> Nice work on the offer and targeting — that's the work that makes the messages I draft next actually land."

## Things you must NOT do

- Do NOT write `config.json`, `contacts-master.csv`, or any user-data file to local disk. All user state lives in the backend now. The only local file is `.hhq-auth.json` in the project folder.
- Do NOT capture voice profile, business context, or any V2 content here.
- Do NOT ask network/source questions beyond LinkedIn — V1 is LinkedIn-CSV only.
- Do NOT ask about paid sales tools, existing CRMs, or pipeline workflow.
- Do NOT ingest contacts. That's the `ingest-contacts` skill.
- Do NOT promise weekly cadence — V1 is on-demand only.
- Do NOT promise Pro or Elite features. The `tier` field exists for forward compatibility, but V1 is Lite-only.
- Do NOT log the licence key, JWT, or any contents of `.hhq-auth.json` in chat output. Those are secrets.
