---
name: new-campaign
description: Create a new outbound campaign in this project. Use when the user wants to run multiple campaigns from the same Helper HQ account — e.g. one for Sunburnt Space customers, another for Sunburnt Space investors, another for Helper HQ customers. Triggers on "new campaign", "start a new campaign", "this project is for a different campaign", "let's run a separate campaign for X". Pins the current Cowork project to the new campaign via `<project>/.hhq-campaign.json` so every subsequent skill (surface-next-5, research-and-draft, etc.) operates inside the new campaign's context. Each campaign holds its own offer / ICP / signals / batch / per-contact cooldown state — but contacts (the master network), the base voice profile, and the licence remain user-level. Optional per-campaign voice additions can layer on top of the base voice. Run AFTER onboard-user has completed at least once on this machine.
---

# New Campaign — Helper HQ

You are creating a new campaign for an already-onboarded Helper HQ user. A campaign is a separately-tracked outbound effort: its own offer, ICP, ranking signals, current batch, and per-contact cooldown state. The user's master contact list, base voice, and licence are shared across all campaigns; only the campaign-shaped fields (offer / ICP / signals / batch / cooldown) are scoped per-campaign.

Typical use case: one Cowork project per campaign. Each Cowork project gets its own session (per-project auth at `<project-dir>/.hhq-session.json`); default cap is 5 simultaneous sessions per licence. Users self-manage at `https://hhq.ngrok.dev/sessions`.

## When this skill runs

Trigger when:
- The user says "new campaign", "create a campaign", "start a new campaign", "this project is for a different campaign / target / offer".
- The user mentions running a parallel outbound effort (different offer or different ICP) to what they currently have.

Do NOT trigger when:
- The user is editing or refining their existing offer / ICP — that's `offer-review` and `icp-discovery` for the *current* campaign.
- The user has not yet onboarded — route to `onboard-user` instead.

## Phase 0 — Auth and prerequisites

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory` to get the persistent project folder path. Save as `<project-dir>`. Fall back to `~/.hhq/<random>/` only in local Claude Code CLI.

### Step 0b — Resolve auth (per-project session)

Read `<project-dir>/.hhq-session.json` (per-project session auth, v0.11+).

- **Found** → parse `backend_url`, `license_key`, `machine_id`, `jwt`, `jwt_expires_at`, `tier`, `helpers`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming to `.hhq-session.json`. Continue.
- **Neither found** → user has not connected this project. Tell them: "This project isn't connected yet. Run `/hhq:connect` first to link it to your account, then come back here to add a new campaign — or run `/hhq:onboard` if you're brand-new to Helper HQ." Stop.

### Step 0c — Refresh JWT if expiring

If `jwt_expires_at` is past or within 60s of expiry:
1. POST `<backend_url>/api/refresh` with `Authorization: Bearer <old jwt>`. On 200, save the new token + expires_at to `.hhq-session.json` (preserving other fields).
2. On 401, the session was released — tell the user to `/hhq:connect`. Stop. Do NOT auto-re-activate.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`.

### Step 0d — Check for existing campaign pin in this project

Read `<project-dir>/.hhq-campaign.json`.

- **Found** → tell the user: "This project is already pinned to campaign `<existing.campaign_slug>`. Want to overwrite the pin and use this project for a new campaign instead, or open a different Cowork project for the new one? (overwrite / use a different project)". If overwrite → continue. If a different project → suggest they open a fresh Cowork project and re-run this skill there. Stop.
- **Not found** → continue.

### Step 0e — List existing campaigns

`GET <backend_url>/api/me/campaigns`

Hold the response in memory. Show the user a brief list so they can see what they already have:

> "Existing campaigns:
>
> · `default` — Default
> · `p2-investors` — Sunburnt Investors
>
> What should this new one be called?"

If only `default` exists, just ask the question without showing the list.

## Phase 1 — Name and slug

> "What should this campaign be called? Short and descriptive — e.g. *Sunburnt Customers*, *HHQ Investors*, *Helper HQ Coaches*."

Validate the name is 1–80 chars, non-empty.

Auto-derive a slug by lowercasing and replacing spaces/punctuation with hyphens (e.g. *Sunburnt Customers* → `sunburnt-customers`). Show it back:

> "Slug: `sunburnt-customers`. Use that, or pick a different one? (use it / different)"

If "different" → ask for the slug, validate `^[a-z0-9][a-z0-9\-]*$` and 1–60 chars.

## Phase 2 — Create the campaign on the backend

```
POST <backend_url>/api/me/campaigns
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "name": "<name>",
  "slug": "<slug>"
}
```

Handle response:
- **HTTP 201** → parse the returned campaign, hold its `slug` and `id`. Continue.
- **HTTP 409 `slug_taken`** → tell the user that slug exists, ask for a different one. Loop back to slug step.
- **HTTP 422 `reserved_slug`** → tell the user that slug is reserved (`new`, `create`, `list`, `me`), ask for a different one. Loop.
- **HTTP 401** → JWT bad. Tell the user to re-onboard. Stop.

### Step 2.5 — Pin the project to this campaign

Write `<project-dir>/.hhq-campaign.json`:

```json
{
  "campaign_slug": "<the slug just created>"
}
```

This pin tells every subsequent skill (surface-next-5, research-and-draft, tune-voice, offer-review, icp-discovery) that this project is operating inside that campaign's context.

## Phase 3 — Offer

The campaign needs its own offer — it's the whole point of a separate campaign.

> "What's the offer for this campaign? Quick (one sentence + URLs, ~3 min) or deep (hook, outcomes, proof, differentiation, pricing band, objections, reads URLs and pulls out canonical phrases, ~10 min)? (quick / deep)"

**If deep:**

Invoke `offer-review` inline. It reads the campaign pin, GETs the campaign config (empty for a fresh campaign), runs its walkthrough, and PUTs the result to `/api/me/campaigns/<slug>/config`.

**If quick:**

Ask: *"What do you offer in this campaign? One sentence."* Save as `offer`.

Then: *"What's the angle prospects are getting excited about right now? One or two sentences."* Save as `offer_hook`.

Optionally URLs for `offer_profile` synthesis (same shape as onboard-user Phase 3c — distil into summary / key_benefits / objection_patterns / canonical_phrases). Skip if no URLs.

Hold the offer fields in memory; PUT happens in Phase 6.

## Phase 4 — ICP

> "Who's the ideal prospect for this campaign? Quick (industry + role + size, ~2 min) or deep (stage, geography, triggers, pain points, disqualifiers, reads up to 3 example client LinkedIn profiles, ~10 min)? (quick / deep)"

**If deep:**

Invoke `icp-discovery` inline. PUTs to `/api/me/campaigns/<slug>/config`.

**If quick:**

Ask the same minimal ICP shape as onboard-user Phase 4.1: *"Who's your ideal prospect? At minimum tell me an industry and a role. Add anything else that matters — company size, stage, geography."*

Save `icp` object in memory.

## Phase 5 — Signals (copy or define)

Ranking signals are per-campaign. Most users will want different weights for different campaigns (the signals that mattered for "Sunburnt Customers" likely differ for "Sunburnt Investors" — investors care about stage + funding history, customers care about role + industry + product progress).

Default to copying from the user's first existing campaign so they have something working, with an option to redefine:

> "I can copy the signals from your *<existing campaign name, usually 'Default'>* campaign as a starting point, or you can define new ones for this campaign. Most people redefine for very different campaigns. (copy / define)"

**If copy:**

`GET <backend_url>/api/me/campaigns/<existing-slug>/config` → take the `signals` field. Hold in memory for Phase 6.

**If define:**

Run a slimmed-down version of onboard-user Phase 5 (steps 5a–5d): open-ended description → propose 5 signals → user adjusts → propose ranking → user adjusts → normalise to weights summing to 1. Hold in memory.

If the existing campaigns list returned only `default` and that campaign has no signals (e.g. user onboarded with only voice and skipped the signals walkthrough), force the "define" branch.

## Phase 6 — Voice additions (optional)

Per-campaign voice additions layer **on top of** the user's base voice. Most campaigns won't need any — your default voice is fine. The additive case is when a campaign needs a specific tonal shift or vocabulary reference that doesn't apply to your other campaigns (e.g. for an investor campaign: lean more formal, mention specific track-record phrases).

> "Any campaign-specific voice notes to add on top of your default voice? Examples: 'lean more formal', 'mention our investor track record', 'never start with a first name', 'add the phrase: stop waiting, start flying'. Skip if not — most campaigns use the default voice as-is."

If the user gives notes, capture as `voice_additions` (same shape as base voice but additive — only fields with content are written):

```json
{
  "summary": "optional 1-sentence override description",
  "tone": ["optional extra tone words to layer on top"],
  "do": ["optional extra do items"],
  "dont": ["optional extra dont items"],
  "phrases": ["optional extra phrases"],
  "notes": "freeform additional notes for the drafter, optional",
  "generated_at": "ISO-8601"
}
```

Only include fields the user actually filled in. If they skipped → `voice_additions: null`.

## Phase 7 — Save the campaign config

Build the config payload with whatever Phase 3–6 produced (excluding fields owned by the deep skills if they were invoked, since those skills already PUT). For the quick paths, do the PUT here.

```
PUT <backend_url>/api/me/campaigns/<slug>/config
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "config": {
    "offer": "...",
    "offer_hook": "...",
    "offer_profile": { ... },
    "icp": { ... },
    "icp_profile": { ... },
    "signals": { ... },
    "voice_additions": { ... }
  }
}
```

If `offer-review` ran in Phase 3, it already PUT `offer`, `offer_hook`, `offer_profile`. Don't include those keys in your PUT — let them stand. (PUT replaces the whole `config`, so first GET the current config and merge before PUTing if a deep skill ran.)

If `icp-discovery` ran in Phase 4, same treatment for `icp` and `icp_profile`.

Robust order:
1. `GET /api/me/campaigns/<slug>/config` → existing values (whatever the deep skills wrote).
2. Build the payload by overlaying your in-memory values on top of the existing config.
3. PUT.

Expect HTTP 200. On 401 → tell user to re-onboard. On 5xx/network → tell them to retry.

## Phase 8 — Close out

Tell the user briefly:

> "Campaign `<slug>` is set up and this project is pinned to it. From here, every prospect surfacing, draft, and cooldown lives inside `<slug>`'s context.
>
> Try saying *'get me the next 5 prospects'* — I'll surface from your network using `<slug>`'s offer, ICP, and signals. Cooldown is hybrid: anyone you contacted in the last 7 days anywhere is locked out, anyone you surfaced or contacted in the last 30 days in another campaign will show a warning before I add them to this batch."

Stop.

## Things you must NOT do

- Do NOT delete or modify other campaigns. This skill only creates a new one and pins this project to it.
- Do NOT touch `<project-dir>/.hhq-session.json` except to refresh the JWT in Phase 0c.
- Do NOT touch the user's base voice profile. That stays at user level on `/api/me/config`. The campaign's voice_additions are *additive*, not a replacement.
- Do NOT copy contacts between campaigns — contacts are user-level master data shared across all campaigns. Per-campaign state (status, last_surfaced_at, research, messages) is created lazily as surfacing happens.
- Do NOT create a campaign with slug `default`, `new`, `create`, `list`, or `me` — those are reserved.
- Do NOT log the licence key, JWT, or contents of `<project-dir>/.hhq-session.json` in chat output.
