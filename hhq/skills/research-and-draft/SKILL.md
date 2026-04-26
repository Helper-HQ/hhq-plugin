---
name: research-and-draft
description: Researches each prospect in the current batch and drafts a Greg-style opener for each. Two entry points — (1) normal flow after surface-next-5 ("let's go", "draft them", etc.) reads /api/me/current-batch, (2) quick-start flow after onboard-user ("let's go on quick start", "research my 5") reads config.quick_start.urls and creates contacts on the fly via POST /api/me/contacts/import after profile reads. Both paths use the Chrome connector to read each prospect's LinkedIn profile + recent posts, save findings to the prospect's `research` field via PUT /api/me/contacts/{slug}, draft a short signal-referenced opener, append to the `messages` field, set status to `drafted`, then present all openers cleanly in chat. Run AFTER surface-next-5 (normal) or AFTER onboard-user with quick-start URLs queued.
---

# Research and Draft — Sales Helper Lite

You are doing pass-2 enrichment and opener drafting for the 5 prospects the user just confirmed in `surface-next-5`. For each prospect: read their public LinkedIn profile + recent posts via the Chrome connector, capture findings to the backend, draft a short signal-referenced opener in the Greg style, save artifacts, present the openers for copying.

This is the heaviest V1 skill and it's where the actual product value lives. Work through it carefully — quality of research and quality of drafts is the whole product.

## When this skill runs

Two entry points: the **normal flow** (after `surface-next-5`) and the **quick-start flow** (after `onboard-user` Phase 7 captured up to 5 LinkedIn profile URLs the user wants to reach out to immediately).

### Normal flow triggers

Trigger when the user says any variant of:
- "let's go"
- "draft them"
- "start drafting"
- "go for it"
- "do it"
- "yes draft"
- "give me the openers"

…AND `/api/me/current-batch` returns a non-empty batch.

### Quick-start flow triggers

Trigger when the user says any variant of:
- "let's go on quick start"
- "do the quick start"
- "research my 5"
- "research my quick start"
- "quick start"

…AND `config.quick_start.urls` is non-empty AND `config.quick_start.status` is `"queued"` (not yet completed).

### Disambiguation

If a phrase like "let's go" is ambiguous (current_batch non-empty AND quick_start.status is "queued"), prefer the **quick-start path** — those URLs were captured most recently and the user is more likely to mean them. After they complete, the user will reach for "let's go" on the normal flow.

If the user says one of the normal-flow phrases but the current batch is empty AND quick_start is unavailable, route them to surface-next-5 first ("There's no current batch — let's surface 5 prospects first. Say 'get me the next 5'.").

## Phase 0 — Auth

Use `mcp__ccd_directory__request_directory` to get the project folder (fall back to `~/.hhq/sales-helper/` in local Claude Code CLI). Save as `<project-dir>`.

Read `<project-dir>/.hhq-auth.json`. If missing → "No auth file — say 'set me up' to onboard." Stop.

Parse `backend_url`, `jwt`, `jwt_expires_at`, `license_key`, `machine_id`.

If `jwt_expires_at` is past or within 60s of expiry:
1. `POST <backend_url>/api/refresh` with `Authorization: Bearer <old jwt>`. On 200, save the new token + expires_at to `.hhq-auth.json`.
2. On 401, re-activate via `POST /api/activate` with the saved `license_key` + `machine_id`. Save the new token. On 403, tell the user and stop.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

## Phase Q — Quick-start branch (alternative entry point)

If the user invoked this skill via a quick-start trigger phrase (see "When this skill runs"), run **Phase Q instead of the normal Pre-flight + Phase 1 loop**. Phase Q uses the same research methodology (Phase 2) and drafting methodology (Phase 3) as the normal flow — the difference is the input (URLs, no contacts yet) and what we do at the end (create contacts after research, since they don't exist yet).

### Step Q1 — Read the queued URLs

`GET <backend_url>/api/me/config`

- Read `quick_start.urls` (array of LinkedIn profile URLs).
- Read `quick_start.status`. If `"completed"` → tell the user "You've already completed quick start — say 'get me the next 5' to surface from your full contact list." Stop.
- Read `offer`, `offer_hook`, `icp`, `voice_samples`, `signals.weighted` — same fields the normal flow uses for drafting.
- If `quick_start.urls` is empty/null → tell the user "No quick-start URLs queued. Want to onboard or surface a normal batch?" Stop.

### Step Q2 — Tell the user what's happening

> "Right — researching your `<N>` quick-start prospects and drafting openers. Same methodology as the normal flow. Takes a few minutes; I'll work through them one at a time."

### Step Q3 — Per-URL loop

For each URL in `quick_start.urls`, do the following sequentially:

**Q3a — Read profile via Chrome.** Same as Step 2c in the normal flow. Use `mcp__Claude_in_Chrome__navigate` then `mcp__Claude_in_Chrome__read_page`.

Capture from the profile:
- First name + last name
- Headline / current role + company
- Location
- About section (only if offer-relevant)
- Featured / pinned content (if any)

**Q3b — Read recent activity.** Same as Step 2d. URL: `<profile_url>/recent-activity/all/`. Read most recent 5–10 posts.

**Q3c — Analyse (in-context, same as Phase 2 Step 2e).** Apply the methodology in Phase 2 — what to look for, what counts as a signal, how to weight a recent post against a role change. The user picked these prospects intentionally, so the *fit* is presumed; lean into the *recent signal* in the opener.

**Q3d — Draft opener (in-context, same as Phase 3).** Apply the Greg-style methodology in Phase 3 below. Use the user's `voice_samples` (if any have been processed by the background worker) to shape phrasing. Use `offer_hook` for the angle; keep the actual ask soft.

**Q3e — Create the contact via POST.** Now that we have name, company, position from the profile read, create the contact:

```
POST <backend_url>/api/me/contacts/import
Content-Type: application/json
Authorization: Bearer <jwt>

{
  "contacts": [
    {
      "first_name": "...",
      "last_name": "...",
      "company": "...",
      "position": "...",
      "linkedin_url": "<the URL>",
      "connected_on": null,
      "raw_csv": null
    }
  ]
}
```

The endpoint upserts by `linkedin_url`, so if the prospect later appears in the bulk LinkedIn export, the existing record will be updated rather than duplicated. The response includes the contact's slug — use it for the next step.

**Q3f — Save research and message.**

```
PUT <backend_url>/api/me/contacts/{slug}
{
  "research": { ...the structured research from Q3c... },
  "messages": [
    {
      "kind": "drafted_opener",
      "drafted_at": "ISO-8601",
      "body": "<the drafted opener>",
      "source": "quick_start"
    }
  ],
  "status": "drafted"
}
```

If profile read fails for a URL (private, 404, rate-limited), DO NOT crash the batch. Save what you can (URL only as the linkedin_url, status "research_failed"), draft a *generic* opener using only what's visible (the URL itself contains the handle), flag the issue clearly when presenting, and continue.

### Step Q4 — Present openers

Use the same presentation format as Phase 5 in the normal flow. List all `<N>` openers with research summaries, LinkedIn URLs, and notes-file paths. Same Greg-style block per prospect.

### Step Q5 — Mark quick_start completed

`PUT <backend_url>/api/me/config` with the existing config + `quick_start.status` updated to `"completed"` and `quick_start.completed_at` set to the ISO timestamp. The URLs themselves stay in config as a record.

### Step Q6 — Close

> "Done — `<N>` quick-start openers ready. When you've sent any, mark them `contacted` like usual. When your LinkedIn bulk export lands, drop the CSV in and I'll work through the rest of your network from there."

Phase Q ends here. Do NOT continue into Phase 1 — the normal flow doesn't apply.

## Hollow-skill seams (V1 dogfood)

This skill has **two** hollow-skill seams that move to server-side MCP calls in V2. The endpoints already exist as stubs:

**Seam A — Research analysis** (`POST /api/mcp/research_analyze`):
Takes the raw page-scrape data and returns a structured `research`-shaped finding. The IP is the analysis prompt — *what to look for, what counts as a signal, how to weight a recent post against a role change*, etc.

**Seam B — Opener drafting** (`POST /api/mcp/draft_opener`):
Takes the research findings + offer + ICP and returns a Greg-style opener. The IP is the drafting prompt — the style guide, the structural rules, the anti-patterns to avoid.

For V1 dogfood you do both yourself in-context. The Greg-style methodology and the analysis heuristics are written below. Treat them as the high-IP content that will live behind the MCP API in V2. Do not call the stub endpoints in V1 — they return placeholder text not yet useful for real drafting.

## Pre-flight checks

### Step A — Fetch the batch

`GET <backend_url>/api/me/current-batch`

- HTTP 200 with non-empty `batch` → continue.
- HTTP 200 with empty `batch` → tell the user "No active batch — say 'get me the next 5' first." Stop.
- HTTP 401 → run the auth fallback once and retry.

Hold the batch in memory: it's a list of `{contact_id, surfaced_at, drafted_at, reasoning}`.

### Step B — Fetch the user's config

`GET <backend_url>/api/me/config` — you need `offer` and `icp` for the drafting heuristics. If config is missing or incomplete, route to onboard-user.

### Step C — Map contact_ids to slugs

`GET <backend_url>/api/me/contacts?per_page=500` (paginate if total > 500). Build a map `{id → {slug, first_name, last_name, company, position, linkedin_url}}` for the prospects in the batch. You'll use the slug for per-prospect detail GET/PUT.

### Tell the user what's about to happen

Briefly tell the user so they're not staring at a silent screen for 5 minutes:

> "Got it — researching <N> prospects and drafting openers. This takes a few minutes; I'll work through them one at a time and show you everything when I'm done."

Then start the per-prospect loop.

## Phase 1 — Per-prospect loop (sequential, one at a time)

For each prospect in the batch, do Phases 2–4. Sequential is fine for V1 — 5 prospects × ~60-90s each is acceptable. Don't try to parallelise.

After each prospect, give a brief progress note in chat ("✓ 1/5 — Greg Coleman done. Next: Marina Park."). No emoji except the check mark for progress. Keeps the user oriented during the wait.

If a single prospect's research fails (page gone, rate-limit, network error), DO NOT crash the whole batch. Mark that prospect's `research` JSON with a `failure_reason` field, attempt the opener using only stored contact data (it'll be more generic — flag this in the message log), and move on to the next.

## Phase 2 — Research a single prospect

For the current prospect:

### Step 2a — Fetch full contact record

`GET <backend_url>/api/me/contacts/{slug}` to get the latest fields including `linkedin_url`, `email`, `headline`, etc.

### Step 2b — Read local notes if any

Check if `<project-dir>/contacts/<slug>/notes.md` exists. If yes, read it — these are the user's own freeform notes ("Greg's away until Jan", "don't pitch testing — they have an in-house lab"). Use them to shape the opener.

If the file doesn't exist, that's fine — most prospects won't have notes.

Notes live as local files (not on the backend) for V1. The user owns this file; never overwrite it. If you create a placeholder later in this skill (Step 2e), only do so if the file is absent.

### Step 2c — Navigate to LinkedIn profile

Use the Chrome connector tools (`mcp__Claude_in_Chrome__navigate`, then `mcp__Claude_in_Chrome__read_page` or `mcp__Claude_in_Chrome__get_page_text`).

URL: prospect's `linkedin_url` field. If empty/missing, fall back to a LinkedIn search by name+company — but accept that this is less reliable.

What you read from the profile:
- Current role + tenure (when did they start the current role?)
- Previous role (recency of any change)
- Location
- "About" section — only if it gives offer-relevant context
- Featured / pinned content — if any

Do NOT read: skills section, endorsements, recommendations, education history beyond current/latest, profile photo, banner. None are useful for the opener and they bloat context.

### Step 2d — Navigate to recent activity

URL pattern: `<linkedin_url>/recent-activity/all/`.

Read the most recent 5–10 posts. For each:
- Date posted
- Post body (full text — these are usually short)
- Post type (their own / repost / comment)

If they have no public activity → note that ("No recent posts visible publicly").

### Step 2e — Analyse and store research (hollow seam A)

This is **seam A**. In V2 the analysis becomes a server-side MCP call. In V1 you do the analysis yourself.

Heuristics for analysis:

- **Recency cliff**: a post in the last 7 days is hot. 7-30 days is warm. 30-90 days is lukewarm. >90 days is cold.
- **Topical hits**: does any recent post touch on the user's offer space? Be specific — "post about scaling sales teams" matches a sales-coach offer, "post about a recent funding round" matches a B2B SaaS offer. Generic "leadership lessons" posts are weak signals.
- **Role-change signals**: did they start their current role recently (last 3 months)? New roles = high openness to vendor conversations. Long tenure (>3 years) = harder to dislodge from their current setup.
- **Shipping signals**: did they announce something they built / shipped / launched? Strong hook.
- **Vulnerability signals**: did they post about a problem they're trying to solve? Strongest possible hook — they've publicly named the pain.

Build a research JSON object with this shape:

```json
{
  "researched_at": "ISO timestamp",
  "profile": {
    "current_role": "Founder",
    "current_company": "Magnetorquer Pty Ltd",
    "tenure": "since Jan 2023",
    "location": "Brisbane, Australia",
    "previous_role": "Engineer at SatCo (2020-2022)"
  },
  "recent_activity": {
    "most_recent_post": { "date": "2026-04-22", "summary": "Announced shipping the Magnetorquer prototype" },
    "themes": ["satellite hardware", "team hiring"],
    "activity_level": "hot"
  },
  "signals_matched": [
    { "signal": "post-relevance", "evidence": "Recent post about Magnetorquer shipping aligns directly with microgravity testing offer" },
    { "signal": "shipping-signal", "evidence": "Just announced launch — open window for vendor conversations" }
  ],
  "hook_for_opener": "Just shipped the Magnetorquer prototype — congratulate, then offer microgravity validation before launch.",
  "notes_for_draft": "User's local notes mention Greg is travelling — keep ask soft, no pressure on timeline."
}
```

If you couldn't get a profile read at all, set:

```json
{
  "researched_at": "ISO timestamp",
  "failure_reason": "<short reason>",
  "fallback": true
}
```

PUT the research field for this prospect:

```
PUT <backend_url>/api/me/contacts/{slug}
{
  "research": <the JSON above>
}
```

If `research` already has prior data (re-surfaced after the 30-day cooldown), the PUT overwrites. If you want to preserve history, prepend a `previous_runs` array — but for V1 simplicity just overwrite.

### Step 2f — Create notes placeholder (only if absent)

If `<project-dir>/contacts/<slug>/notes.md` does NOT exist, create the directory + placeholder:

```markdown
# Notes: <Full Name>

*Your own notes go here. Examples:*
*- "Greg's away until Jan, come back then."*
*- "Met at the SmallSat conference 2025."*
*- "Don't pitch testing — they have an in-house lab."*
```

If notes.md already exists, leave it completely untouched. The user owns this file.

## Phase 3 — Draft a Greg-style opener (hollow seam B)

This is **seam B**. In V2 the drafting becomes a server-side MCP call. In V1 you do it yourself using the methodology below.

### What a Greg-style opener IS

- **3-5 sentences max.** Often shorter is better.
- **Opens with a specific reference** — a post, a milestone, a role change, something they shipped. NOT "I noticed you..." or "I came across your profile..." or "Hope you're well!"
- **States offer relevance in one line** — what you do, why it connects to *their specific situation*. Not a pitch, not a value prop deck.
- **Soft ask** — "happy to chat", "lmk if useful", "no pressure". Never "let's set up a call" or "15 minutes of your time".
- **Reads like a smart peer noticed something** — not like an SDR running a sequence.

### What a Greg-style opener IS NOT

- ❌ "Hi Greg, hope you're well!" — generic greeting filler
- ❌ "I help small satellite companies with microgravity testing." — generic value prop, not personal
- ❌ "I came across your profile and was impressed by..." — fake personalisation
- ❌ "Would love to set up 15 minutes to discuss how we can help you." — pushy ask
- ❌ Any emoji
- ❌ Any "!" exclamation (unless replicating user's voice and they use them)
- ❌ Buzzword stack ("synergy", "leverage", "scale", "transform")
- ❌ Long preamble before the actual point

### Structure (3 lines, often less)

1. **Specific opener** — refers to the hook from the research
2. **Bridge to relevance** — one sentence connecting their thing to your offer
3. **Soft ask** — leave the door open, don't push

### Examples

**Good** (illustrative):
> Hey Greg — saw your post about the attitude control work you're shipping next quarter. We do microgravity component validation at Sunburnt Space; if you're after a final shake-down before launch, happy to chat. No pressure either way.

**Good** (a "they shipped something" hook):
> Hey Marina — congrats on the launch of FlightDeck. The demo video looks slick. We work with founders on outbound right around your stage — if you're thinking about a more deliberate go-to-market motion, lmk and I'll send something useful.

**Good** (a "recent role change" hook):
> Hey Tim — saw you joined Gilmour Space as Senior Propulsion Engineer last month, congrats. We work with propulsion teams on flight-readiness testing in microgravity; if it's relevant for your roadmap I'd love to compare notes. No pitch unless asked.

**Bad** (avoid this):
> Hi Greg, hope you're well! I saw you're a Founder at Magnetorquer. I help small satellite companies with microgravity testing solutions. Would love to set up a quick 15-minute call to discuss how Sunburnt Space can help you scale your operations. Let me know what works in your calendar!

### Drafting process

1. Re-read the `hook_for_opener` field from the research blob.
2. Re-read the user's `offer` from config.
3. Read local `notes.md` for the prospect if any — let it shape the draft (e.g. soften the ask if user noted travel; skip the offer if user noted "in-house lab").
4. Draft using the structure above.
5. **Self-edit pass:** check it against the IS / IS NOT lists. Cut anything weak. If it could have been written about anyone, rewrite — it must reference *this specific prospect's situation*.

### If the research failed

Use the stored contact fields (position, company, headline). Be honest in the draft — don't fabricate posts or activity. The opener will be more generic. Example fallback:

> Hey Tim — noticed we connected a while back. We work with propulsion teams on flight-readiness testing — if it's relevant for your work at Gilmour, happy to share more. Otherwise no pressure.

### Append the opener to messages

GET the current `messages` array on the contact (you already pulled the full contact in Step 2a; reuse that).

Append a new entry:

```json
{
  "drafted_at": "ISO timestamp",
  "channel": "linkedin",
  "kind": "opener",
  "draft": "<the opener text>",
  "research_failed": false,
  "user_notes_present": true | false
}
```

PUT the updated messages array:

```
PUT <backend_url>/api/me/contacts/{slug}
{
  "messages": <the appended array>
}
```

Preserve all prior message entries — append, don't replace.

## Phase 4 — Update status after each prospect

For the prospect just drafted:

```
PUT <backend_url>/api/me/contacts/{slug}
{
  "status": "drafted"
}
```

Don't touch `last_surfaced_at` — surface-next-5 already set it. Don't mark `contacted` — V1 has no automated send-tracking.

## Phase 5 — After all prospects done — clear the batch and present

Once the loop completes:

### Step 5a — Clear the batch

```
PUT <backend_url>/api/me/current-batch
{ "batch": [] }
```

This way the next `surface-next-5` call doesn't see a stale batch.

### Step 5b — Present all openers in chat

Show all openers cleanly so the user can copy them. Format:

```
All done. Here are your <N> openers — copy, tweak if you want, send from LinkedIn:

═══ 1. Greg Coleman — Founder, Magnetorquer Pty Ltd ═══

<opener text>

LinkedIn: <url>
Notes file: <project-dir>/contacts/<slug>/notes.md

═══ 2. Marina Park — CEO, FlightDeck ═══

<opener text>

LinkedIn: <url>
Notes file: <project-dir>/contacts/<slug>/notes.md

... etc
```

If any prospect failed research, mark it clearly:

```
═══ 4. Tim Reyes — Senior Propulsion Engineer, Gilmour Space ═══

⚠ Research failed (rate-limited). Opener uses stored contact data only — more generic than usual.

<fallback opener text>

LinkedIn: <url>
Notes file: <project-dir>/contacts/<slug>/notes.md
```

### Step 5c — Close

> "When you've sent any of these, you can mark them `contacted` by saying 'mark Greg as contacted' — V1 doesn't track sends automatically, but the next surface skips anything in `drafted` for 30 days regardless.
>
> When you're ready for the next 5, just say 'get me the next 5'."

## Things you must NOT do

- Do NOT do anything beyond the prospect's profile + recent posts. No company page deep-dive, no news search, no LinkedIn graph traversal. Narrow.
- Do NOT log into LinkedIn or attempt any authenticated actions. Public profile reads only.
- Do NOT fabricate posts, role history, or anything that wasn't actually visible. If research is thin, say so honestly.
- Do NOT overwrite local `notes.md`. Ever. The user owns it.
- Do NOT replace the contact's `messages` array — always append.
- Do NOT mark prospects `contacted` — V1 has no send tracking.
- Do NOT use emoji in opener drafts.
- Do NOT use exclamation marks in opener drafts (unless mirroring the user's voice in a future tier).
- Do NOT pitch in the opener. Soft ask only.
- Do NOT call the `/api/mcp/*` endpoints in V1 — leave them as documented seams. V1 does research + drafting in-context.
- Do NOT modify `.hhq-auth.json` except to update `jwt` / `jwt_expires_at` after a refresh / re-activate.
- Do NOT log the JWT, licence key, or auth file contents.
- Do NOT touch the user's config.

## Edge cases to handle gracefully

- **Profile is private / "Out of network"** → research fails, fall back to stored contact data, flag in the opener block.
- **No recent posts** → activity level = silent, hook from role/company match instead. Be honest in the opener — no fake "saw your recent post" reference.
- **Profile URL is wrong / 404** → mark research failed, fall back, suggest the user verify the LinkedIn URL.
- **Prospect's company in the contact record differs from current LinkedIn profile** (they changed jobs since the export) → research finds a new company. PUT a single update with both the research and the corrected `company` / `position` fields.
- **Rate-limited mid-batch** → mark remaining prospects as research-failed, fall back, finish the batch. Don't retry mid-flight (it'll just re-rate-limit).
- **Single-name profile** (no last name) → use what's available. Slug already handled by ingest-contacts (which delegates to backend).
- **Two prospects in batch with the same name** (unlikely) → slugs differ (server-side disambiguation), so notes files don't collide.
- **`/api/me/current-batch` is empty mid-flow** → another session may have cleared it. Tell the user honestly and suggest re-surfacing.
- **User interrupts mid-batch** ("stop", "pause", "wait") → finish the current prospect cleanly (don't leave a half-PUT contact), then stop. The current-batch on the backend still reflects the original 5 with `drafted_at` set on the ones you finished. The user can re-trigger this skill — check each contact's `messages` field for a recent opener entry to skip already-done prospects.
- **Backend down mid-batch** → if Chrome research succeeded but the PUT fails, you have research data you can't persist. Hold it in conversation context, surface a partial result honestly, suggest retry.
