---
name: onboard-user
description: One-time setup for the Helper HQ Sales Helper plugin. TRIGGERS on phrases like "set up helper hq", "onboard me to helper hq", "set up sales helper", "start sales helper", "activate my helper hq licence", "activate my sales helper licence", "I have a helper hq licence key", "set me up for sales helper", "onboard me", "re-onboard", "reset my setup", "start over", or when the user explicitly invokes /hhq:onboard. Activates the user's licence against the Helper HQ backend, kicks off the LinkedIn export FIRST (Connections + Messages, because LinkedIn takes up to 24h), then runs a deeper conversation on the user's offer (one-sentence + hook + URLs read inline to distil an offer_profile), target prospect, up to 5 weighted ranking signals, and voice samples (brand guide as text/URL/PDF/DOCX, articles, LinkedIn message URLs — read inline to distil a voice_profile the user reviews and tunes). Optionally captures up to 5 LinkedIn profile URLs as a quick-start batch so the user can get openers drafted before the bulk LinkedIn export arrives. Saves to the backend via PUT /api/me/config. Run this BEFORE ingest-contacts, surface-next-5, or research-and-draft. For ongoing voice tuning AFTER onboarding, use tune-voice instead.
---

# Onboard User — Sales Helper Lite

You are running the one-time setup for the Sales Helper Lite plugin. You activate the user's licence against the Helper HQ backend, kick off their LinkedIn export first (because LinkedIn takes up to 24 hours to email it back), then dial in their offer, target prospect, and signal weights while the export cooks. You save everything to the backend via the API.

This is the user's FIRST experience of the plugin. Be warm, brief, and conversational. Total target time: 15 minutes — most of it is the offer + ICP + signals deep dive. Activation and the LinkedIn step are fast.

## When this skill runs

Trigger when:
- No `~/.hhq/machine.json` exists on this machine (and no legacy `<project-dir>/.hhq-auth.json` to migrate from) and the user is interacting with Helper HQ for the first time, or
- The user explicitly asks to re-onboard, reset, reconfigure, or "start over."

If the user wants a *new campaign* (different offer / ICP from what's already set up), route them to `/hhq:new-campaign` instead — this skill is for first-time setup or full reset.

## Backend URL (V1 dogfood)

The Helper HQ backend lives at:

```
https://hhq.ngrok.dev
```

This is a stable ngrok subdomain pointing at the local Herd backend during V1 dogfood. The URL doesn't rotate, but the tunnel is only up when the admin (Brad) has ngrok running. If activation or any API call returns a network error, the tunnel is probably down — tell the user to retry in a few minutes; do not change the URL or write a partial auth file. Production will replace this with a stable Helper HQ domain.

## Phase 0 — Backend activation

This phase has to happen first, before anything else, because the rest of onboarding writes to the backend.

Auth is stored at the **machine level** in `~/.hhq/machine.json` (so opening a new Cowork project for a new campaign does NOT consume a machine slot — same machine, same auth). Each project pins itself to a specific campaign via `<project-dir>/.hhq-campaign.json`.

### Step 0a — Get the project folder

Use the `mcp__ccd_directory__request_directory` tool with no `path` argument to get the persistent project folder. The user will accept a permission prompt the first time.

Save the returned path as `<project-dir>` for the rest of this skill. The campaign-pin file lives there.

If the tool isn't available (local Claude Code CLI), fall back to `~/.hhq/sales-helper/` and create it if missing.

### Step 0a.5 — Request access to `~/.hhq/` (per-machine auth folder) — REQUIRED CALL

**You MUST call `mcp__ccd_directory__request_directory({"path": "~/.hhq"})` at this step.** Do not skip it. Do not assume Cowork has already granted or denied access — the call IS the mechanism by which Cowork shows the user a permission prompt (*"Helper HQ wants access to ~/.hhq"*). Without making the call there is no prompt, no grant, no way to read or write the per-machine auth file. Skipping the call is the bug that breaks v0.10's per-machine semantics — every Cowork project would consume its own machine slot.

This folder holds `machine.json` — the per-machine licence + cached JWT. Sharing it across all the user's Cowork projects on the same laptop is what lets a single licence cover multiple campaigns without burning a machine slot per project.

Three possible outcomes from making the call:

- **Tool returns success (user approves the prompt OR Cowork has already granted access from a prior project / skill invocation)** → continue to Step 0b. Reads and writes to `~/.hhq/` work for the rest of the session.
- **Tool returns an error indicating the user declined** → set `home_hhq_unavailable = true` in your skill memory. Continue to Step 0b — the skill will fall back to per-project auth at `<project-dir>/.hhq-auth.json`. Tell the user once, briefly: "OK, I'll use a per-project auth file instead. Each Cowork project will need its own licence activation, which uses one of your machine slots."
- **Tool itself is not registered in this session** (rare CLI-only case where the CCD MCP isn't loaded) → treat as approved. CLI doesn't sandbox `~/.hhq/`. Continue.

Do NOT short-circuit this step with assumptions like *"the shared ~/.hhq folder isn't accessible from Cowork"* — that conclusion can only be reached *after* making the call and getting back a denial. If you haven't made the call, you don't know.

### Step 0b — Check for existing machine auth

If `home_hhq_unavailable` is set, skip directly to checking `<project-dir>/.hhq-auth.json` (the per-project fallback) below.

Otherwise, read `~/.hhq/machine.json`.

- **Found and you are NOT explicitly re-onboarding** → tell the user: "Your machine is already activated. To create a new outbound campaign in this project, run `/hhq:new-campaign`. To redo your full setup (overwrite voice + default-campaign offer/ICP/signals), say 'redo onboarding' or 'start over'. (new campaign / redo onboarding / cancel)". Route accordingly. If "new campaign" → stop and tell them to run `/hhq:new-campaign`. If "redo onboarding" → continue, reuse machine_id and licence_key from the existing file. If "cancel" → stop.
- **Found and user IS re-onboarding** → continue, reuse machine_id and licence_key.
- **Not found, but `<project-dir>/.hhq-auth.json` exists** → legacy auth file from before v0.10. Migrate it inline:
  1. `mkdir -p ~/.hhq`.
  2. Copy `<project-dir>/.hhq-auth.json` to `~/.hhq/machine.json`.
  3. Delete `<project-dir>/.hhq-auth.json`.
  4. Then proceed as if "Found and you are NOT explicitly re-onboarding" above.
- **Not found and no legacy file** → genuine first-time setup. Continue to Step 0c.

### Step 0c — Get the licence key

Skip if reusing an existing licence key from the machine.json (re-onboarding case).

Otherwise ask:

> "First, paste your Helper HQ licence key. It looks like `hhq_...` and you got it in your purchase email."

Validate the shape: starts with `hhq_`, length at least 16 chars. If invalid, ask again with a brief explanation.

### Step 0d — Activate

For first-time setup: generate a UUIDv4 as the `machine_id`. Use `powershell -NoProfile -c '[guid]::NewGuid().ToString()'` on Windows or `uuidgen` on Mac/Linux. Strip whitespace.

For re-onboarding: reuse the existing `machine_id` from `~/.hhq/machine.json`.

Call the backend:

```
POST <backend-url>/api/activate
Content-Type: application/json

{
  "license_key": "<the licence key>",
  "machine_id": "<the UUID>"
}
```

Use `curl -sk -X POST ... -H 'Content-Type: application/json' -d '{...}'` via the Bash tool.

Handle the response:

- **HTTP 200** → parse the JSON. Backend auto-creates a `default` campaign for new users on activation. Save `~/.hhq/machine.json` (Step 0e) and continue.
- **HTTP 404 `license_not_found`** → ask the user to re-paste from their purchase email. Loop back to Step 0c.
- **HTTP 403 `license_inactive`** → revoked / suspended / expired. Tell user to contact `help@helperhq.co` and stop.
- **HTTP 403 `machine_limit_reached`** → "Your licence is at its 3-machine limit. Auth is per-machine in v0.10+ — running multiple campaigns from the same laptop does NOT consume extra slots. If you really need more machines, contact `help@helperhq.co`." Stop.
- **HTTP 422 / network error** → show the error, don't write a partial auth file, stop.

### Step 0e — Save the auth and campaign files

**If `home_hhq_unavailable` is NOT set** (canonical path — user approved access to `~/.hhq/`):

Create `~/.hhq/` if missing (`mkdir -p ~/.hhq`). Write `~/.hhq/machine.json`:

```json
{
  "backend_url": "https://hhq.ngrok.dev",
  "license_key": "<the licence key>",
  "machine_id": "<the UUID>",
  "jwt": "<the token returned by /api/activate>",
  "jwt_expires_at": "<the expires_at returned>",
  "tier": "<lite|pro|elite>",
  "helpers": ["<the helpers array>"]
}
```

Update `tier` and `helpers` in this file every refresh/re-activation in case they changed.

**If `home_hhq_unavailable` IS set** (user declined `~/.hhq/` access in Step 0a.5):

Write the auth payload to `<project-dir>/.hhq-auth.json` instead, with the exact same shape (`backend_url`, `license_key`, `machine_id`, `jwt`, `jwt_expires_at`, `tier`, `helpers`). This is the per-project fallback. Each new Cowork project run will need its own activation under this path, consuming one machine slot per project.

Either way, also write `<project-dir>/.hhq-campaign.json`:

```json
{
  "campaign_slug": "default"
}
```

This pins the current Cowork project to the `default` campaign. Subsequent skills (surface-next-5, research-and-draft, etc.) read this file to know which campaign's context to operate in. To run a parallel outbound effort with a different offer/ICP, the user opens a new Cowork project and runs `/hhq:new-campaign` there.

If re-onboarding (machine.json already existed), overwrite both files.

## Phase 0.5 — Gmail connector prereq check

Run this immediately after activation succeeds. **Hard gate.** Helper HQ uses Claude's Gmail connector to ingest the user's address book and recent correspondence — without it, the Gmail-driven flows (auto-update "In conversation" stage when active back-and-forth is detected, surface bidirectional correspondents as warm prospects) won't work, and bringing it up *after* 15 minutes of setup is a worse experience than checking now.

### Step 0.5a — Tool introspection

Look at the tools available in your current session. If you can see ANY tools whose names map to Gmail operations — `search_threads`, `list_drafts`, `get_thread`, `list_labels`, `create_draft`, `create_label` (typically prefixed with `mcp__<some-uuid>__` in the function list) — the connector is installed. Skip to "If installed" below.

If you cannot see any Gmail-prefixed tools, the connector is not loaded for this session — go to Step 0.5b.

### Step 0.5b — Halt and tell the user

Tell the user honestly and briefly:

> "Quick prereq before we go further — Helper HQ uses Claude's Gmail connector to ingest your address book and recent correspondence. I can't see it loaded in this session.
>
> Could you install it? In Cowork: Settings → Connectors → Gmail. In Claude.ai: Settings → Connectors → Gmail.
>
> Once it's connected, start a new chat in this project and say 'set me up' again."

Then STOP. Do NOT continue with the intro, fit check, LinkedIn export walkthrough, or anything else. The connector is required for this product.

Do NOT try to work around the absence (no "we'll skip Gmail for now" path). The Gmail-driven stage transitions and warm-correspondent surfacing are core to how the product works as of v0.9.

### If installed

Acknowledge briefly and continue:

> "Gmail connector detected — good. Continuing."

(Or just move into the intro without ceremony — the acknowledgment is optional. What matters is the gate fails closed when the connector is missing.)

## Conversational style

- One question at a time. Adapt to their answers. Just flow from question to question — **no yes/no gate between every question**.
- Yes/no gates appear only at **major boundaries**: the very start (ready to begin?), before the LinkedIn export walkthrough, and at the end. Not between adjacent questions inside this skill.
- **Do NOT echo saved answers back.** Avoid "Saved as your offer: [...]" or "Got it, you're targeting [...]." Just acknowledge briefly ("Got it. Next, ...") and move on.
- Open-ended phrasing is fine for *content* the user provides.
- Accept reasonable yes-equivalents ("yes", "yeah", "sure", "go", "ok") and same for no. Respect contextual answers.
- Do not lecture. Do not list features. Get to the questions.

## The intro

Run the intro AFTER Phase 0 succeeds, so the user knows their licence activated cleanly before you start the deep dive.

> "Welcome to Sales Helper Lite. Your licence is active. Here's what I do: I look across your contacts — LinkedIn connections, business cards you've scanned, spreadsheets and CRM exports you've imported, and recent Gmail correspondents — and surface 5 prospects at a time worth a personal opener, based on real signals like recent posts, role changes, or things they've shipped. Then I draft a short, specific opening message for each. You review and send. The magic is in the messages; sending is the easy part.
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

## Phase 2 — Kick off the LinkedIn export (full archive)

This goes FIRST in the working portion of onboarding because LinkedIn can take **up to 24 hours** to email the file back. We want the timer running before the deep dive on offer and ICP.

LinkedIn no longer lets users pick individual files — they only ship a **full archive** (a `.zip` with everything). That archive contains both `Connections.csv` (drives prospect ranking) and `messages.csv` (drives recent-conversation flagging + voice sampling from outgoing messages), plus a bunch of files we don't care about. The user extracts the two files we want from the zip when the email lands.

> "Right, first thing — let's kick off your LinkedIn data export so the clock's running. LinkedIn ships everything as one big archive these days. Want me to walk you through it? (yes / no)"

If yes, give the steps:

> "Takes about 30 seconds:
>
> 1. Go to **linkedin.com/mypreferences/d/download-my-data**
> 2. Choose **Want a copy of all your data?** — LinkedIn no longer lets you pick individual files, so the full archive is the only option.
> 3. Click **Request archive** and confirm with your password.
> 4. LinkedIn emails you a `.zip` when it's ready (usually a few hours, sometimes up to 24).
>
> When it lands, you'll only need two files from the zip — `Connections.csv` and `messages.csv`. The rest can be ignored.
>
> Done? (yes / no)"

If they have any questions, answer briefly. When they confirm done, save `linkedin_export.requested_at: <timestamp>` and continue.

If they're hesitant about messages (privacy, legal, etc.), respect it — tell them to extract only `Connections.csv` from the zip when the time comes, skipping `messages.csv`. Save `linkedin_export.messages_requested: false`. The plugin works without messages; voice will improve more slowly without them.

If they said no to the walkthrough entirely → save `linkedin_export.requested_at: null`. Tell them they can run `ingest-contacts` whenever they have CSVs ready, and continue with the rest of onboarding regardless — offer / ICP / signals are useful work either way.

## Phase 2.5 — Existing pipeline import (optional)

While LinkedIn is preparing their archive, ask whether the user already tracks customers or pipeline somewhere we can import now. This lets the user land in the deep dive with their `/pipeline` view non-empty from day one rather than waiting for the LinkedIn export and then scrolling through 1000+ Lead-stage strangers.

> "Quick one before the deep dive — do you already track your existing customers or pipeline somewhere? If you have a spreadsheet, CSV, or CRM export (HubSpot, Salesforce, Pipedrive — any of those), drop it now and I'll bring it in so your pipeline is complete from day one. (yes — drop a file / no / skip)"

**If yes (file dropped or path provided):**

Invoke `ingest-contacts` inline. The Spreadsheet / CRM branch will run — column mapping, stage-label mapping if there's a status column, then POST to import. When it returns, save `existing_pipeline_imported: true` and the result counts to skill memory, and continue.

If the import fails partway (network, validation), tell the user briefly and offer to skip and continue — don't grind on it. They can re-run `ingest-contacts` later.

**If no / skip:**

Save `existing_pipeline_imported: false`. Tell the user briefly:

> "All good — you can drop a spreadsheet or CRM export any time later by saying 'import my spreadsheet' or 'I have a CRM export'."

And continue.

Then transition into the deep dive:

> "Good — that's the slow stuff handled. While LinkedIn prepares the file, let's dial in your offer and your target. This is the work that makes the messages I draft for you actually land."

## Phase 3 — Offer

### Step 3.0 — Quick or deep gate

Ask up front so the user picks depth once:

> "Quick offer pass here, or want the deeper offer review? Quick is one sentence + URLs (3 min). Deep walks through hook, outcomes, proof, differentiation, pricing band, and objections, reads your URLs, and pulls out canonical language for openers (~10 min). You can always run the deeper one later by saying 'review my offer'. (quick / deep)"

Accept natural-language equivalents ("quick" / "fast" / "simple" → quick; "deep" / "deeper" / "full" / "the long one" → deep).

**If "deep":**

Invoke the `offer-review` skill inline. It will GET the current config (which has `offer`/`offer_hook`/`offer_profile` all null at this point in onboarding, so no overwrite gate fires), run its full walkthrough, and PUT the result. When it returns, mark `_deep_skills_used.offer_review = true` in your skill memory and **skip the rest of Phase 3** — jump straight to Phase 4. Do not re-ask the same questions.

**If "quick" (or skipped):**

Continue the existing quick path below.

### Step 3.1 — Quick offer

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

Save the final tightened version verbatim as `offer`. Move directly into Phase 3b.

## Phase 3b — Offer hook (the angle)

The one-sentence offer says *what* it is. The hook says *why people are leaning in right now*. Capture this — it's the difference between a generic message and one that lands.

> "What's the angle prospects are getting excited about right now? The thing that makes them lean in when you talk about it. One or two sentences."
>
> Example: *Cost and cadence — flights are cheap and frequent enough that microgravity testing is suddenly viable on a real timeline. People are starting to think about what they could fly soon.*

**Sharpen if needed.** If they give you something generic ("our quality" / "great service" / "we care about customers"), probe once:

> "More specific — what's *changed recently* about your offer or your market that's making people pay attention? What are they saying when they get on a call with you?"

If they can't articulate it, save what they have and move on. One sharpening round max — don't grind.

Save verbatim as `offer_hook`. Move directly into Phase 3c.

## Phase 3c — Offer URLs (read + distil now)

> "Drop any URLs that explain your offer better than you'd be able to in chat — product page, pricing, a brochure, recent launch post, anything. I'll read them now and pull out the key benefits and canonical phrases so I can echo your language when I draft openers. Skip if you don't have any."

Accept space, comma, or newline-separated URLs. Validate each starts with `http://` or `https://`. Trim whitespace, dedupe.

If the user pastes nothing / "skip" / "none" → save `offer_profile: null` and move directly into Phase 4.

If they give URLs, run the synthesis pass before moving on.

### Step 3c.1 — Tell the user what's happening

> "Reading those now — back in a minute."

### Step 3c.2 — Fetch each URL

For each URL:
- Use `WebFetch` (or `mcp__Claude_in_Chrome__navigate` + `read_page` if a site blocks plain HTTP fetches — Sunburnt-style marketing sites are usually fine with WebFetch).
- If a fetch fails (404, timeout, blocked), note the failure and continue with the rest. Don't crash the phase.

### Step 3c.3 — Distil into offer_profile

Pull out, from everything you read:
- **`summary`** — 1–3 sentences on what the offer is and the angle that lands.
- **`key_benefits`** — short phrases the company uses, verbatim where possible.
- **`objection_patterns`** — common concerns the pages address (price, timing, complexity, etc.).
- **`canonical_phrases`** — distinctive wording worth echoing in openers.

Save as `offer_profile`:

```json
{
  "summary": "...",
  "key_benefits": ["...", "..."],
  "objection_patterns": ["...", "..."],
  "canonical_phrases": ["...", "..."],
  "sources": ["https://...", "..."],
  "failed_sources": ["https://..."],
  "generated_at": "ISO-8601 timestamp"
}
```

**Do NOT save the page contents themselves anywhere.** Distil and forget. The profile is the only persistent artifact.

Move directly into Phase 4. **No yes/no gate.** Brief acknowledgment of what you found — one line, naming the angle and a benefit or two.

## Phase 4 — ICP

### Step 4.0 — Quick or deep gate

> "Same call for ICP — quick version here (industry + role + size, 2 min) or the deeper ICP discovery (walks through stage, geography, triggers, pain points, disqualifiers, and reads up to 3 example client LinkedIn profiles, ~10 min)? Run the deeper one later anytime with 'discover my ICP'. (quick / deep)"

**If "deep":**

Invoke the `icp-discovery` skill inline. It will GET current config (with `icp`/`icp_profile` null at this point), run its walkthrough, and PUT the result. When it returns, mark `_deep_skills_used.icp_discovery = true` in skill memory and **skip the rest of Phase 4** — jump straight to Phase 5.

**If "quick" (or skipped):**

Continue the existing quick path below.

### Step 4.1 — Quick ICP

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

## Phase 6 — Voice (gather + synthesise + tune)

This is the difference between openers that sound like *you* and openers that sound like generic LLM output. We gather samples, synthesise them into a structured voice profile, and let you tune it before saving. The user can also retune anytime later via the `tune-voice` skill.

> "Last bit — voice. The more I know how *you* talk, the better drafts will sound like you instead of a generic LLM. I'll gather a few examples, then read everything and pull out a voice profile you can review. About 2-3 minutes once we hit the read step."

### Step 6a — Brand or voice guide

> "Got a brand guide or voice doc? Paste the text directly, drop a URL, attach a PDF or Word file, or skip."

Three branches based on what they give you:

- **Pasted prose / multiple lines** — capture the text verbatim. Hold in skill memory as `gathered.brand_guide_text`. Don't persist yet.
- **Single `http(s)://` URL** — hold as `gathered.brand_guide_url` (will be fetched in Step 6.5).
- **Attached file (`.pdf` / `.docx` / `.txt`)** — read the file inline using the `pdf` skill (for PDFs), `docx` skill (for Word), or direct read (for txt). Extract the text. Hold as `gathered.brand_guide_text` (the extracted content). **Do NOT save the file anywhere — content only.**
- **Skip** — hold as null.

### Step 6b — Articles you've written

> "Drop 2–3 URLs of blog posts or articles you've written that sound like you — the pieces where you'd say 'yeah, that's how I talk.' Skip if you don't have any."

Accept space/comma/newline-separated URLs. Validate `http(s)://`. Dedupe. Hold as `gathered.article_urls: [...]`.

### Step 6c — LinkedIn messages that landed

> "Last one — drop 2–3 LinkedIn message URLs where *you* opened a conversation that went well. I'll read them in your logged-in Chrome session and learn your opening style. Skip if you'd rather not share."

Accept URLs containing `linkedin.com`. Dedupe. Hold as `gathered.linkedin_message_urls: [...]`.

If the user gives nothing across all three steps → save `voice_profile: null`. Move directly to Phase 7.

### Step 6.5 — Synthesise

If anything was gathered, run the synthesis pass. Tell the user:

> "Reading everything now — back in 2-3 minutes."

For each item:

- **Brand-guide URL or article URL** — `WebFetch` the page. If blocked, fall back to `mcp__Claude_in_Chrome__navigate` + `read_page`.
- **LinkedIn message URL** — `mcp__Claude_in_Chrome__navigate` to the thread, `read_page` or `get_page_text`. Extract only the user's outgoing messages from the thread.
- **Brand guide text** (already in hand) — use as-is.

Note any source that fails (404, blocked, can't extract). Capture in `failed_sources` — don't crash.

From everything successfully read, distil:

```json
{
  "summary": "1-2 sentence description of how the user talks. e.g. 'Direct, plain, no jargon. Single-line questions. Curious, not selling.'",
  "tone": ["direct", "warm", "curious"],
  "do": [
    "Use first names",
    "Reference specific things they posted",
    "Ask one short question"
  ],
  "dont": [
    "Use the word 'leverage'",
    "Use exclamation marks",
    "Open with 'hope this finds you well'"
  ],
  "phrases": [
    "Hey <Name> — saw your <thing>. <observation>. <one short question>?"
  ],
  "sources": {
    "articles": ["https://...", "..."],
    "linkedin_messages": ["https://...", "..."],
    "brand_guide": "text" | "pdf" | "docx" | "url" | null
  },
  "failed_sources": ["https://..."],
  "generated_at": "ISO-8601"
}
```

Aim for: 2–4 tone words, 4–8 do items, 4–8 dont items, 1–3 example phrases. Keep them tight — this is structure the user (and `research-and-draft`) will scan repeatedly.

**Do NOT save the source contents.** Only the distilled profile. Throw away the raw page text once synthesis is done.

### Step 6.6 — Review and tune

Show the synthesised voice nicely:

```
Here's your voice based on what you gave me:

Summary
  Direct, plain, no jargon. Single-line questions. Curious, not selling.

Tone
  · direct  · warm  · curious

Do
  · Use first names
  · Reference specific things they posted
  · Ask one short question
  · ...

Don't
  · Use the word "leverage"
  · Use exclamation marks
  · Open with "hope this finds you well"
  · ...

Sound check
  "Hey Toufiq — saw the lidar testing post. Towing behind a car? Any plans to go higher?"

[Sources used: 2 articles, 1 LinkedIn message]
[Failed: 1 article (404)]
```

Then ask:

> "Want to add or remove anything? You can say things like 'remove the leverage rule', 'add: never end with looking forward to hearing from you', 'change the summary to ...', or 'looks good'. (edit / done)"

Loop:
- If user requests an edit → apply it (string-match on the item to remove, append to the relevant array for adds, replace summary verbatim if asked). Re-show the updated voice. Ask again.
- If user says "done" / "looks good" / "yes" / similar → save `voice_profile` and continue.
- If after 5 edit rounds the user is still tweaking, gently nudge: "We can keep tuning, or save what we've got and refine later via 'tune my voice'. (continue / done)"

Save the final tuned profile as `voice_profile` (the JSON shape above). Move directly into Phase 7.

## Phase 7 — Quick start (optional)

LinkedIn's export takes hours to a day. While the user waits, they can hand-pick up to 5 prospects they already know they want to reach out to and queue them for immediate research + draft. **Strictly optional** — if they skip, save runs as-is.

> "One last thing before we wrap — LinkedIn's export takes hours. If you've already got people in mind, paste up to 5 LinkedIn profile URLs and I'll queue them for research right after we close out — you'll have openers ready before LinkedIn even ships the bulk export. Or skip and we'll wait for the import. (paste URLs / skip)"

### Parsing

- Accept space, comma, or newline-separated URLs.
- Validate each starts with `http(s)://` AND contains `linkedin.com/in/`. Reject anything that doesn't (company pages, post URLs, message threads — those aren't profiles).
- Trim whitespace, dedupe within the list.
- **Cap at 5.** If the user pastes more, take the first 5 and say honestly: "Saved the first 5 — paste the rest after we work through this batch."

### Save

If at least one valid URL was provided:

```json
"quick_start": {
  "urls": ["https://www.linkedin.com/in/...", "..."],
  "requested_at": "ISO-8601 timestamp",
  "status": "queued"
}
```

If user skips / "no" / "skip" / "none" → save `quick_start: null`. Continue.

If user pastes URLs but none are valid LinkedIn profile URLs → tell them honestly: "None of those looked like LinkedIn profile URLs (they need to look like `linkedin.com/in/<handle>`). Want to retry, or skip?" One retry max — if still nothing valid, save `quick_start: null` and move on.

Do NOT attempt research or drafting in this phase — just queue. The actual research happens when the user invokes `research-and-draft` with a quick-start trigger phrase (handled in that skill).

Move directly into Phase 8. **No yes/no gate.**

## Phase 8 — Save and finish

Save the answers to the backend in **two PUTs**:

1. User-level config (voice, fit, linkedin_export, quick_start, tier, onboarded_at, version) → `PUT /api/me/config`.
2. Default campaign config (offer*, icp*, signals) → `PUT /api/me/campaigns/default/config`.

Read `~/.hhq/machine.json` to get `backend_url` and `jwt`.

### Step 8.0 — GET current configs first (merge, don't clobber)

If `_deep_skills_used.offer_review` or `_deep_skills_used.icp_discovery` is true in skill memory, those skills already PUT their fields to the campaign during onboarding. Don't overwrite them.

```
GET <backend_url>/api/me/config                       → existing_user_config
GET <backend_url>/api/me/campaigns/default/config     → existing_campaign_config
```

For any deep-skill-owned fields, prefer the `existing_campaign_config` values over what's in skill memory:

- If `_deep_skills_used.offer_review` → use `existing_campaign_config.offer`, `.offer_hook`, `.offer_profile`.
- If `_deep_skills_used.icp_discovery` → use `existing_campaign_config.icp`, `.icp_profile`.
- Otherwise → use the values you captured in Phase 3 / Phase 4.

All other fields come from skill memory.

### Step 8.1 — Build the user-level payload

```json
{
  "version": 1,
  "tier": "lite",
  "onboarded_at": "ISO-8601 timestamp",
  "fit": {
    "sales_owner": "self" | "team",
    "continued_after_team_warning": true
  },
  "voice_profile": {
    "summary": "1-2 sentence description of how the user talks",
    "tone": ["direct", "warm", "curious"],
    "do": ["..."],
    "dont": ["..."],
    "phrases": ["..."],
    "sources": {
      "articles": ["https://..."],
      "linkedin_messages": ["https://..."],
      "brand_guide": "text | pdf | docx | url | null"
    },
    "failed_sources": ["https://..."],
    "generated_at": "ISO-8601 timestamp"
  },
  "linkedin_export": {
    "requested_at": "ISO-8601 timestamp or null",
    "messages_requested": true
  },
  "quick_start": {
    "urls": ["https://www.linkedin.com/in/...", "..."],
    "requested_at": "ISO-8601 timestamp",
    "status": "queued"
  }
}
```

`PUT <backend_url>/api/me/config` with `{"config": <payload>}`. Expect HTTP 200.

### Step 8.2 — Build the default-campaign payload

```json
{
  "offer": "verbatim one-sentence offer",
  "offer_hook": "verbatim 1-2 sentence hook from Phase 3b",
  "offer_profile": {
    "summary": "...",
    "key_benefits": ["...", "..."],
    "objection_patterns": ["...", "..."],
    "canonical_phrases": ["...", "..."],
    "sources": ["https://...", "..."],
    "failed_sources": ["https://..."],
    "generated_at": "ISO-8601 timestamp"
  },
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
  }
}
```

`PUT <backend_url>/api/me/campaigns/default/config` with `{"config": <payload>}`. Expect HTTP 200.

If either call fails:
- **HTTP 401** → JWT is bad. Tell the user their session expired, ask them to re-run onboarding. Stop.
- **HTTP 5xx or network error** → tell the user the backend's not responding, suggest they retry in a moment. Keep the answers in conversation context so a retry can re-PUT.

The `tier` field is set to `"lite"` for V1. Pro and Elite tiers will check this field at runtime to gate features when they ship.

Do NOT write any other local files. The only local files are `~/.hhq/machine.json` (per-machine auth) and `<project-dir>/.hhq-campaign.json` (per-project campaign pin). Everything else lives backend-side.

Then close warmly. **Two close variants depending on whether Phase 7 captured quick-start URLs:**

**Variant A — quick-start URLs were saved (`config.quick_start.urls.length > 0`):**

> "All set — everything's saved. Your machine licence is at `~/.hhq/machine.json` and this project is pinned to your `default` campaign via `.hhq-campaign.json`. Keep both around.
>
> Your LinkedIn export is in flight (usually a few hours, sometimes up to 24). And — your `<N>` quick-start prospects are queued. Whenever you're ready, say 'let's go on quick start' and I'll research each, draft an opener, and have them ready to send in a few minutes.
>
> When the LinkedIn email lands, open a fresh chat in this same project, drop the CSV in, and say 'I've got my LinkedIn export'.
>
> Nice work on the offer and targeting — that's the work that makes the messages I draft next actually land."

**Variant B — no quick-start URLs (`config.quick_start` is null):**

> "All set — everything's saved. Your machine licence is at `~/.hhq/machine.json` and this project is pinned to your `default` campaign via `.hhq-campaign.json`. Keep both around.
>
> Your LinkedIn export is in flight. When the email lands (usually a few hours, sometimes up to 24), open a fresh chat in this same project, drop the CSV in, and say 'I've got my LinkedIn export'. I'll take it from there.
>
> Nice work on the offer and targeting — that's the work that makes the messages I draft next actually land."

## Things you must NOT do

- Do NOT write `config.json`, `contacts-master.csv`, or any user-data file to local disk. All user state lives in the backend now. The only local files are `~/.hhq/machine.json` (per-machine auth) and `<project-dir>/.hhq-campaign.json` (per-project campaign pin).
- Do NOT save raw page contents, PDFs, DOCX text, or LinkedIn message bodies anywhere. Synthesis is the only persistent artifact — distil and forget the source content.
- Do NOT save uploaded files to disk or to the backend. PDF/DOCX inputs in Phase 6a are read inline; the file itself is the user's responsibility to keep.
- Do NOT pressure for voice samples. Phase 6 is gravy — if the user skips, save `voice_profile: null`; they can build it later with `tune-voice`.
- Do NOT loop on Phase 6.6 review forever. After 5 edit rounds, gently nudge them to "save and refine later" rather than grinding.
- Do NOT ask network/source questions beyond LinkedIn — V1 is LinkedIn-CSV only.
- Do NOT ask about paid sales tools, existing CRMs, or pipeline workflow.
- Do NOT ingest contacts. That's the `ingest-contacts` skill.
- Do NOT promise weekly cadence — V1 is on-demand only.
- Do NOT promise Pro or Elite features. The `tier` field exists for forward compatibility, but V1 is Lite-only.
- Do NOT log the licence key, JWT, or any contents of `~/.hhq/machine.json` in chat output. Those are secrets.
