---
name: icp-discovery
description: Marketing Helper — guided ICP (ideal customer profile) discovery. Walks the user through a deeper definition of who their target prospect is for a campaign — industry, role, stage/size, geography, triggers, pain points, buying signals, disqualifiers, example clients — and synthesises a structured `icp_profile` plus the simpler `icp` fields used by Sales Helper. Use when the user says "discover my ICP", "define my ICP", "ICP discovery", "who's my ideal customer", "review my ICP", "redo my ICP", "let's nail down the target", or invokes /hhq:icp-discovery. Can be run standalone any time, or invoked from `onboard-helperhq` Phase 4 when the user picks the "deep" option. Loads existing `icp` / `icp_profile` from the backend; if one exists, prompts the user before overwriting (V1 only stores ONE active ICP — multi-ICP per campaign is a V2 backlog item).
---

# ICP Discovery — Marketing Helper

You are running a deeper guided walkthrough to define the user's ideal customer profile (ICP). This skill is the long-form counterpart to `onboard-helperhq` Phase 4 — it asks more questions, captures more structure, and reads any URLs the user gives you to extract canonical language about who their best clients are.

The output feeds Sales Helper's prospect ranking (via the simple `icp` fields) and gives `research-and-draft` richer context to reference in openers (via `icp_profile`).

**V1 stores ONE active ICP.** Multi-ICP per campaign is a V2 backlog item — be explicit with the user about this when overwriting an existing profile.

## When this skill runs

Trigger when the user says:
- "discover my ICP", "define my ICP", "ICP discovery"
- "who's my ideal customer", "let's nail down my target"
- "review my ICP", "redo my ICP", "update my ICP"
- "deep ICP" (from inside `onboard-helperhq` Phase 4)

Or when invoked inline from `onboard-helperhq` Phase 4 ("deep" branch).

Do NOT trigger if `<project-dir>/.hhq-session.json` is missing (and no legacy `<project>/.hhq-auth.json` to migrate from) — route to `/hhq:connect` first.

## Phase 0 — Auth and campaign

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory` (no arguments) to get the project folder. Save as `<project-dir>`. CLI fallback: `~/.hhq/sales-helper/`.

### Step 0b — Resolve auth (per-project session)

Read `<project-dir>/.hhq-session.json` (per-project session auth, v0.11+).

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming to `.hhq-session.json`. Continue.
- **Neither found** → "No auth — say `/hhq:connect` to link this project." Stop.

If `jwt_expires_at` is past or within 60s of expiry, proactively refresh: POST `<backend_url>/api/refresh` with the existing JWT (accepts expired tokens). Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json`.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key in chat.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key` from `.hhq-session.json`** (NOT a fresh UUID — reusing the same UUID keeps this idempotent and avoids burning a slot). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop (`session_limit_reached` includes the `/sessions` URL).

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

### Step 0c — Resolve current campaign

Read `<project-dir>/.hhq-campaign.json` to get `campaign_slug`. If missing, write `{"campaign_slug": "default"}` and use `default`.

ICP is per-campaign in v0.10+. This skill operates on `campaign_slug`'s ICP, not user-level.

## Phase 1 — Load current campaign config

`GET <backend_url>/api/me/campaigns/<campaign_slug>/config`

Hold the full campaign config in skill memory — you'll need it intact for the final PUT (we do GET → merge → PUT so we never overwrite fields owned by other skills).

Read `icp` and `icp_profile` from the response.

**If both are null / missing:** continue straight into Phase 3 (no overwrite gate needed).

**If either exists:** go to Phase 2.

## Phase 2 — Show existing + confirm overwrite

Show the existing ICP cleanly:

```
Your current ICP

  Industry      <icp.industry>
  Role          <icp.role>
  Size/stage    <icp.company_size_or_stage or "—">
  Geography     <icp.geography or "—">
  Other         <icp.other or "—">

Profile (from last discovery: <icp_profile.generated_at or "—">)

  Summary
    <icp_profile.summary or "—">

  Triggers          · <triggers ...>
  Pain points       · <pain_points ...>
  Disqualifiers     · <disqualifiers ...>
  Example clients   · <example_clients ...>

[Sources used: <N> URLs]
```

If `icp_profile` is null, show only the simple `icp` block and note "No deep profile yet."

Then say:

> "V1 only stores one active ICP — running discovery again replaces this. (Per-campaign ICPs are coming in V2.) Run a fresh discovery? (yes / no)"

If no → stop politely. Tell them they can come back any time with "review my ICP".
If yes → continue.

## Phase 3 — Discovery walkthrough

Conversational style — same rules as `onboard-helperhq`:

- One question at a time. Adapt to their answers. **No yes/no gate between adjacent questions.**
- **Do NOT echo saved answers back.** Brief acknowledgment ("Got it. Next, ...") and move on.
- Don't grind. One or two sharpening rounds per question max — capture what they give you.
- Open-ended phrasing for content the user provides.

### Step 3a — Industry / vertical

> "Start with industry — what verticals or industries are your best-fit clients in? You can list a few; I'll capture them all. If you serve more than 3, give me your top 3."

Hold as `icp_profile.industries: [...]`. Cap at 5 even if they list more (tell them honestly: "I'll work with the top 5 — if it matters which, put your strongest first.").

The first item also becomes `icp.industry` (the simple field Sales Helper ranks against).

**Sharpen if vague** ("any business", "B2B", "startups") → "Within that, who's the *best* fit? The clients who lit up when you explained what you do — what industry were they in?"

### Step 3b — Role / decision-maker

> "Who's the buyer? The role or title that signs off on bringing you in. Be specific — 'Head of Engineering' beats 'leadership'. If there's a champion who isn't the buyer, name them too."

Hold as `icp_profile.roles: [...]`. The buyer-role becomes `icp.role`.

**Sharpen if vague** ("decision-makers", "founders") → "What's the actual title on their LinkedIn?"

### Step 3c — Company size or stage

> "Size or stage? Pick whichever fits — employees, revenue band, funding stage, ARR — and skip if it doesn't matter."

Hold as `icp_profile.company_size_or_stage: "..."` and copy to `icp.company_size_or_stage`.

### Step 3d — Geography

> "Geography matter, or do you sell globally?"

Hold as `icp_profile.geography: "..."` and copy to `icp.geography`. "Global" / "anywhere" / "skip" → save as null.

### Step 3e — Triggers (the why-now)

This is the highest-value question for Sales Helper. Triggers are events that shift a prospect from "not in market" to "in market right now" — they drive the recency-weighted signals.

> "What's *changed* in their world right after they become a hot prospect? Recent funding, a new hire, a launch, a regulatory change, a public commitment to something — anything that flipped them from 'not now' to 'right now'. List as many as you can think of."

Hold as `icp_profile.triggers: [...]`. Cap at 8.

**Sharpen if generic** ("they need our product") → "More specific — what just *happened* in the world that put them in market? Think back to the last 3 deals you closed — what was going on at their company that month?"

### Step 3f — Pain points

> "What's the pain they're feeling before they buy? Not the outcome you deliver — the actual hurt that's making them look. The thing that's keeping someone up at night."

Hold as `icp_profile.pain_points: [...]`. Cap at 8.

### Step 3g — Disqualifiers

> "Who's *not* a fit? Industries, sizes, stages, or situations where you've learned not to take the meeting. Skip if there's nothing obvious."

Hold as `icp_profile.disqualifiers: [...]`.

### Step 3h — Example clients (optional)

> "Drop 1–3 LinkedIn URLs of dream-fit existing clients (or prospects) so I can read their profiles and pull out shared patterns. Skip if you'd rather not share."

Accept LinkedIn profile URLs (`linkedin.com/in/`). Validate, dedupe. Hold as `icp_profile.example_client_urls: [...]`. Cap at 3.

If they describe clients in prose instead of URLs, capture the prose as `icp_profile.example_clients_described: "..."` and skip the fetch step for those.

### Step 3i — Buying signals (URL sources)

> "Anything written down that captures who buys from you? Customer case studies, segment pages, pitch deck slides, podcast you did about your audience — drop URLs and I'll read them. Skip if nothing comes to mind."

Accept space/comma/newline-separated URLs. Validate `http(s)://`. Dedupe. Hold as `icp_profile.source_urls: [...]`. Cap at 5.

If user gives nothing across 3h and 3i → skip the synthesis fetch step. Move directly to Phase 5.

## Phase 4 — Read sources and synthesise (remote prompt)

The synthesis prompt is **server-controlled** — fetched at runtime from the backend, admin-tunable without a plugin release. This is the "remote skill" pattern shared with `research-and-draft`.

### Step 4a — Read source pages

If any URLs were captured in 3h or 3i:

> "Reading those now — back in 1-2 minutes."

For each URL:
- **LinkedIn profile (3h)** → `mcp__Claude_in_Chrome__navigate` + `read_page`. Pull headline, current company, role, recent posts (so we can spot patterns across example clients).
- **General URL (3i)** → `WebFetch`. Fall back to Chrome navigate + read_page if blocked.

If a fetch fails (404, blocked, timeout), capture the URL in a local `failed_urls` list and continue. Don't crash.

Hold the raw page text per URL in skill memory — needed for the synthesis call below, then discarded.

If user gave NO URLs across both 3h and 3i → skip Step 4b's fetch step but still run Step 4b to synthesise from Phase 3 answers alone (the prompt handles the empty-sources case).

### Step 4b — Fetch the synthesis prompt

```
GET <backend_url>/api/mcp/prompts/synthesise_icp_profile
Authorization: Bearer <jwt>
```

Returns:

```json
{
  "name": "synthesise_icp_profile",
  "template": "<the prompt with {{placeholders}}>",
  "version": <int>,
  "updated_at": "ISO-8601"
}
```

**Fallback:** if the fetch fails (404 / network / 5xx), use the embedded baseline at the bottom of this file (Step 4d) and continue. Don't block on the backend — the user has work in flight.

### Step 4c — Substitute and run

Substitute the user's Phase 3 answers + raw page reads into the template:

- `{{user_industries}}` — Phase 3a list, joined with " · "
- `{{user_roles}}` — Phase 3b list
- `{{user_company_size_or_stage}}` — Phase 3c, or "—" if blank
- `{{user_geography}}` — Phase 3d, or "—"
- `{{user_triggers}}` — Phase 3e list as numbered lines
- `{{user_pain_points}}` — Phase 3f list as numbered lines
- `{{user_disqualifiers}}` — Phase 3g list, or "—" if empty
- `{{example_client_profiles_raw}}` — concatenated raw page reads from 3h with URL headers (`### URL\n<text>`); "—" if none
- `{{source_urls_raw}}` — concatenated raw page reads from 3i with URL headers; "—" if none
- `{{generated_at}}` — current ISO-8601 timestamp

Run the substituted prompt. The model returns the structured `icp_profile` JSON.

Parse the JSON. Validate top-level keys exist. If parse fails, retry once with "Return only valid JSON, no preamble." If retry also fails, fall back to Step 4d.

**Don't save the raw page contents anywhere.** The prompt has consumed them; throw them away.

### Step 4d — Embedded baseline (fallback)

Used only if Step 4b's fetch fails OR Step 4c's parse fails twice. Keeps the user unblocked.

Build the profile manually from Phase 3 answers — set `summary` from offer/triggers, copy lists 1:1 from user input, set `key_benefits`/`canonical_phrases`/`buying_signals`/`example_clients` to empty arrays (no synthesis happened), record any failed URLs in `failed_sources`. Tell the user honestly: "Heads up — synthesis prompt unreachable, saved your answers as-is. Retry later with 'review my ICP' once the backend's back."

## Phase 5 — Show the profile + tune

Display the full ICP nicely:

```
Your ICP (generated <generated_at>)

  Summary
    <summary>

  Industries     · <industry> · <industry> ...
  Roles          · <role> · <role> ...
  Size/stage     <company_size_or_stage>
  Geography      <geography or "—">

  Triggers
    · <trigger>
    · <trigger>
    ...

  Pain points
    · <pain>
    · <pain>
    ...

  Buying signals
    · <signal>
    · <signal>
    ...

  Disqualifiers
    · <dq>
    · <dq>
    ...

  Example clients
    · <pattern>
    · <pattern>

[Sources: N URLs · Failed: M]
```

Then ask:

> "Want to add or remove anything? Edits in plain language work — 'remove the X disqualifier', 'add: triggers include hiring a CFO', 'change the summary to ...'. Or 'looks good'. (edit / done)"

Loop:
- If user requests an edit → apply (string-match for removals, append for adds, replace verbatim for summary). Re-show. Ask again.
- If "done" / "looks good" / "yes" / similar → save.
- After 5 edit rounds without "done", nudge: "We can keep tuning, or save what we've got and refine later. (continue / save and done)"

## Phase 6 — Save

GET the current campaign config fresh (in case anything else changed during the conversation), splice in the new `icp` and `icp_profile`, PUT to the same campaign endpoint.

```
PUT <backend_url>/api/me/campaigns/<campaign_slug>/config
Authorization: Bearer <jwt>
Content-Type: application/json

{ "config": <existing campaign config with icp + icp_profile replaced> }
```

Expected HTTP 200. On 401, run auth fallback once and retry. On 5xx / network error, tell the user honestly and don't lose their answers — keep them in conversation context for retry.

## Close

**If invoked standalone:**

> "ICP saved. Sales Helper will use this when it ranks your next batch of prospects. Run 'review my offer' next if you haven't already, or 'get me the next 5 prospects' to put it to work."

**If invoked from inside `onboard-helperhq`:**

Return control to onboard-helperhq without the close message. Onboard-user will close at Phase 8.

## Things you must NOT do

- Do NOT save raw page contents, LinkedIn profile bodies, or PDF/DOCX text. Distil and forget.
- Do NOT touch other config fields. This skill only writes `icp` and `icp_profile`.
- Do NOT promise per-campaign ICPs — that's a V2 backlog item. If asked, tell the user honestly: "V1 stores one active ICP. Multi-ICP per campaign is on the roadmap."
- Do NOT log the JWT or licence key in chat.
- Do NOT loop forever in Phase 5. After 5 rounds, nudge to save.
- Do NOT skip the overwrite confirmation in Phase 2 — overwriting an ICP is destructive (V1 has no version history).

## Edge cases

- **`icp_profile` exists but `icp` is null** (or vice versa) — treat as "exists" for Phase 2; show whatever's there.
- **All URL fetches fail** in Phase 4 → tell the user, save the profile from Phase 3 answers only (no `buying_signals` or `example_clients` derived from sources), still move to Phase 5.
- **User asks to clear the whole profile** → confirm explicitly: "That'll wipe your ICP entirely. Sure? (yes / no)" If yes, set both `icp: null` and `icp_profile: null` and save. They can rebuild via this skill.
- **User pastes more than the cap** (e.g. 10 industries, 6 example clients) — take the first N and say honestly: "Capped at <N>. Re-run discovery later if you want to swap any."
- **User describes campaign-style segmentation** ("for this campaign I'm targeting X, but normally Y") → tell them honestly that V1 is single-ICP and ask which one to save now. Capture the other in conversation only — they can swap by re-running the skill.
