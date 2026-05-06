---
name: offer-review
description: Marketing Helper — guided offer review. Walks the user through a deeper definition of what they sell — one-sentence offer, hook/angle, outcomes, proof points, differentiation, pricing band, common objections, and source URLs read inline — and synthesises a structured `offer_profile` plus the simple `offer` and `offer_hook` fields used by Sales Helper. Use when the user says "review my offer", "offer review", "redo my offer", "update my offer", "let's nail down the offer", "offer discovery", "tighten my offer", or invokes /hhq:offer-review. Can be run standalone any time, or invoked from `onboard-helperhq` Phase 3 when the user picks the "deep" option. Loads existing offer fields from the backend; if any exist, prompts the user before overwriting (V1 only stores ONE active offer — multi-offer per campaign is a V2 backlog item).
---

# Offer Review — Marketing Helper

You are running a deeper guided walkthrough to dial in the user's offer. This skill is the long-form counterpart to `onboard-helperhq` Phase 3 — it asks more questions, captures more structure (proof, differentiation, pricing band), and reads any URLs the user gives you to extract canonical language about what they sell.

The output feeds `research-and-draft` so openers can echo the user's own words back when referencing what they do.

**V1 stores ONE active offer.** Multi-offer per campaign is a V2 backlog item — be explicit with the user when overwriting an existing offer.

## When this skill runs

Trigger when the user says:
- "review my offer", "offer review", "tighten my offer"
- "redo my offer", "update my offer"
- "offer discovery", "let's nail down the offer", "deep offer"
- "deep offer" (from inside `onboard-helperhq` Phase 3)

Or when invoked inline from `onboard-helperhq` Phase 3 ("deep" branch).

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

Offer is per-campaign in v0.10+. This skill operates on `campaign_slug`'s offer, not user-level.

## Phase 1 — Load current campaign config

`GET <backend_url>/api/me/campaigns/<campaign_slug>/config`

Hold the full campaign config in skill memory — needed intact for the final PUT (GET → merge → PUT pattern).

Read `offer`, `offer_hook`, `offer_profile` from the response.

**If all three are null / missing:** continue straight into Phase 3 (no overwrite gate needed).

**If any exist:** go to Phase 2.

## Phase 2 — Show existing + confirm overwrite

Show the existing offer cleanly:

```
Your current offer

  Offer
    <offer or "—">

  Hook
    <offer_hook or "—">

Profile (from last review: <offer_profile.generated_at or "—">)

  Summary
    <offer_profile.summary or "—">

  Key benefits        · <key_benefits ...>
  Proof points        · <proof_points ...>
  Differentiation     · <differentiation ...>
  Pricing band        <pricing_band or "—">
  Objection patterns  · <objection_patterns ...>
  Canonical phrases   · <canonical_phrases ...>

[Sources used: <N> URLs]
```

If `offer_profile` is null, show only `offer` and `offer_hook`, and note "No deep profile yet."

Then say:

> "V1 only stores one active offer — running review again replaces this. (Per-campaign offers are coming in V2.) Run a fresh review? (yes / no)"

If no → stop politely. Tell them they can come back any time with "review my offer".
If yes → continue.

## Phase 3 — Review walkthrough

Conversational style (same as `onboard-helperhq`):

- One question at a time. Adapt to their answers. **No yes/no gate between adjacent questions.**
- **Do NOT echo saved answers back.** Brief acknowledgment ("Got it. Next, ...") and move on.
- One or two sharpening rounds per question max — capture what they give you.

### Step 3a — One-sentence offer

> "Start with one sentence — what do you offer? The thing prospect signals will be matched against.
>
> Example: *Sunburnt Space offers microgravity testing for small satellite components.*"

If their answer is more than 2 sentences, ask them to tighten:

> "Same idea in one sentence. Tightest version."

If vague ("I help businesses grow" / "I help coaches scale"), probe once:

> "That's the outcome — what's the actual *thing* you sell? A program, a service, a product? And who specifically is it for?"

Save the tightened version verbatim as `offer`.

### Step 3b — The hook (why-now)

> "What's the angle prospects are getting excited about *right now*? The thing that makes them lean in. One or two sentences.
>
> Example: *Cost and cadence — flights are cheap and frequent enough that microgravity testing is suddenly viable on a real timeline.*"

If generic ("our quality" / "great service"), probe once:

> "More specific — what's *changed recently* about your offer or your market that's making people pay attention? What are they saying when they get on a call with you?"

Save verbatim as `offer_hook`.

### Step 3c — Outcomes the buyer cares about

> "What's the buyer trying to get done? The job-to-be-done, in their words — not your words. List 2–4 outcomes."

Hold as `offer_profile.outcomes: [...]`.

**Sharpen if the user gives features instead of outcomes** — "That's a feature. What does the buyer get because of that feature? What changes for them on Monday morning?"

### Step 3d — Proof points

> "What proof do you have? Case studies, named clients, metrics, awards, anything that makes a sceptical prospect believe you. Drop the headline version of each — I'll get URLs in a minute."

Hold as `offer_profile.proof_points: [...]`. Cap at 6.

### Step 3e — Differentiation

> "What's the alternative they'd compare you to — another vendor, doing it in-house, doing nothing? And what's specifically different about how *you* do it?"

Capture both alternative and differentiator. Hold as `offer_profile.differentiation: [...]` (one entry per alternative).

### Step 3f — Pricing band

> "Rough pricing band — under $1k, $1-10k, $10-50k, $50-250k, $250k+ — or skip if you'd rather not say. This helps me read fit in research; I won't quote it back to prospects."

Hold as `offer_profile.pricing_band: "..."` or null.

### Step 3g — Common objections

> "Top 2-3 things that come up on a discovery call when a prospect's interested but not sold. The hesitations you've heard a lot."

Hold as `offer_profile.objection_patterns: [...]`.

### Step 3h — Source URLs

> "Last bit — drop URLs that explain your offer better than chat would. Product page, pricing page, case study, recent launch post, podcast where you described it well. I'll read them and pull out canonical phrases. Skip if nothing comes to mind."

Accept space/comma/newline-separated URLs. Validate `http(s)://`. Dedupe. Hold as `offer_profile.source_urls: [...]`. Cap at 6.

If user gives nothing → skip Phase 4 fetch step. Move directly to Phase 5 (the profile will have everything from Phase 3 but no URL-derived fields).

## Phase 4 — Read sources and synthesise (remote prompt)

The synthesis prompt is **server-controlled** — fetched at runtime from the backend, admin-tunable without a plugin release. This is the "remote skill" pattern shared with `research-and-draft`.

### Step 4a — Read source pages

If URLs were captured in 3h:

> "Reading those now — back in 1-2 minutes."

For each URL:
- `WebFetch` the page. Fall back to `mcp__Claude_in_Chrome__navigate` + `read_page` if blocked.

If a fetch fails (404, blocked, timeout), capture the URL in a local `failed_urls` list and continue. Don't crash.

Hold the raw page text per URL in skill memory — needed for the synthesis call below, then discarded.

If user gave NO URLs → skip Step 4a's fetch step but still run Step 4b to synthesise from Phase 3 answers alone (the prompt handles the empty-sources case).

### Step 4b — Fetch the synthesis prompt

```
GET <backend_url>/api/mcp/prompts/synthesise_offer_profile
Authorization: Bearer <jwt>
```

Returns:

```json
{
  "name": "synthesise_offer_profile",
  "template": "<the prompt with {{placeholders}}>",
  "version": <int>,
  "updated_at": "ISO-8601"
}
```

**Fallback:** if the fetch fails (404 / network / 5xx), use the embedded baseline at Step 4d. Don't block — keep the user moving.

### Step 4c — Substitute and run

Substitute the user's Phase 3 answers + raw page reads into the template:

- `{{user_offer}}` — Phase 3a verbatim
- `{{user_offer_hook}}` — Phase 3b verbatim
- `{{user_outcomes}}` — Phase 3c list as numbered lines
- `{{user_proof_points}}` — Phase 3d list as numbered lines, "—" if empty
- `{{user_differentiation}}` — Phase 3e list, "—" if empty
- `{{user_pricing_band}}` — Phase 3f, or "—" if skipped
- `{{user_objection_patterns}}` — Phase 3g list as numbered lines
- `{{source_urls_raw}}` — concatenated raw page reads from 3h with URL headers (`### URL\n<text>`); "—" if none
- `{{generated_at}}` — current ISO-8601 timestamp

Run the substituted prompt. The model returns the structured `offer_profile` JSON.

Parse the JSON. Validate top-level keys exist. If parse fails, retry once with "Return only valid JSON, no preamble." If retry also fails, fall back to Step 4d.

**Don't save the raw page contents anywhere.** The prompt has consumed them; throw them away.

### Step 4d — Embedded baseline (fallback)

Used only if Step 4b's fetch fails OR Step 4c's parse fails twice. Keeps the user unblocked.

Build `offer_profile` manually from Phase 3 answers — `summary` = a synthesis of `offer` + `offer_hook` + `outcomes` you write inline, copy other lists 1:1 from user input, set `key_benefits` and `canonical_phrases` to empty arrays. Tell the user honestly: "Heads up — synthesis prompt unreachable, saved your answers as-is. Retry later with 'review my offer' once the backend's back."

## Phase 5 — Show the profile + tune

Display it nicely:

```
Your offer (generated <generated_at>)

  Offer
    <offer>

  Hook
    <offer_hook>

  Summary
    <summary>

  Outcomes
    · <outcome>
    · <outcome>

  Key benefits
    · <benefit>
    · <benefit>

  Proof points
    · <proof>
    · <proof>

  Differentiation
    · <vs alternative>
    · <vs alternative>

  Pricing band
    <pricing_band or "—">

  Common objections
    · <objection>
    · <objection>

  Canonical phrases (echoed in openers)
    · "<phrase>"
    · "<phrase>"

[Sources: N URLs · Failed: M]
```

Then ask:

> "Want to add or remove anything? Edits in plain language work — 'remove the X benefit', 'add: proof points include the Series A close', 'change the hook to ...'. Or 'looks good'. (edit / done)"

Loop:
- Edit → apply (string-match for removals, append for adds, replace verbatim for summary/offer/hook). Re-show. Ask again.
- "done" / "looks good" / "yes" → save.
- After 5 edit rounds without "done", nudge: "We can keep tuning, or save what we've got and refine later. (continue / save and done)"

## Phase 6 — Save

GET the current campaign config fresh, splice in `offer`, `offer_hook`, `offer_profile`, PUT to the same campaign endpoint.

```
PUT <backend_url>/api/me/campaigns/<campaign_slug>/config
Authorization: Bearer <jwt>
Content-Type: application/json

{ "config": <existing campaign config with offer + offer_hook + offer_profile replaced> }
```

Expected HTTP 200. On 401, auth fallback once and retry. On 5xx / network error, tell the user honestly and don't lose their answers.

## Close

**If invoked standalone:**

> "Offer saved. The next opener I draft will reference your offer language — canonical phrases, key benefits, the hook. Run 'discover my ICP' next if you haven't already, or 'get me the next 5 prospects' to put it to work."

**If invoked from inside `onboard-helperhq`:**

Return control to onboard-helperhq without the close message. Onboard-user will close at Phase 8.

## Things you must NOT do

- Do NOT save raw page contents or PDF/DOCX text. Distil and forget.
- Do NOT touch other config fields. This skill only writes `offer`, `offer_hook`, `offer_profile`.
- Do NOT promise per-campaign offers — that's a V2 backlog item. If asked, tell the user honestly: "V1 stores one active offer. Multi-offer per campaign is on the roadmap."
- Do NOT log the JWT or licence key in chat.
- Do NOT loop forever in Phase 5. After 5 rounds, nudge to save.
- Do NOT skip the overwrite confirmation in Phase 2.
- Do NOT quote the user's pricing band back to prospects. The field is for internal-fit reading only.

## Edge cases

- **`offer` exists but `offer_profile` is null** — treat as "exists" for Phase 2; show what's there.
- **All URL fetches fail** in Phase 4 → tell the user, build `offer_profile` from Phase 3 only, move to Phase 5.
- **User says they have no proof yet** (e.g. very early-stage) — capture honestly as empty array. Don't push.
- **User asks to clear the whole offer** → confirm explicitly: "That'll wipe your offer entirely. Sure? (yes / no)" If yes, set `offer: null`, `offer_hook: null`, `offer_profile: null` and save.
- **User describes a campaign-specific offer** ("for this LinkedIn push I'm leading with X, but my main offer is Y") → tell them V1 is single-offer and ask which to save now. Capture the other in conversation only.
