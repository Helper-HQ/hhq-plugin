---
name: surface-next-5
description: On-demand prospect surfacing — campaign-scoped. Triggers when the user says "get me the next 5 prospects", "who should I reach out to", "give me 5 leads", "find me some prospects", or similar. Resolves the current campaign from `<project-dir>/.hhq-campaign.json` (default `default`), reads that campaign's offer / ICP / signals plus eligible contacts (server-side filter on per-campaign cooldown, hard 7-day global lockout via last_contacted_at, recently-messaged 30-day filter, pipeline stage), ranks the rest by the campaign's weighted signals, surfaces the top 5 with one-line signal-referenced reasoning AND any cross-campaign warnings (someone surfaced in another campaign in the last 30 days), lets the user drop / swap, then persists the confirmed batch via PUT /api/me/campaigns/{slug}/current-batch and writes per-campaign contact status via PUT /api/me/campaigns/{slug}/contacts/{contactSlug}. Run AFTER onboard-user and ingest-contacts have run at least once.
---

# Surface Next 5 — Sales Helper Lite

You are surfacing the next 5 prospects worth a personal opener, ranked by the user's weighted signals using only what's already in the contact record. This is the on-demand pass-1 filter. Pass-2 enrichment (LinkedIn posts, recent activity, role changes) happens later in `research-and-draft` for the narrowed shortlist of 5.

Keep this fast and tight. One operation: rank, surface, confirm, persist, hand off.

## When this skill runs

Trigger when the user says any variant of:
- "get me the next 5 prospects"
- "who should I reach out to"
- "give me 5 leads"
- "find me some prospects"
- "let's prospect"
- "next 5"

Do NOT trigger if the user is asking for one specific person, asking general questions, or working with someone already drafted.

## Phase 0 — Auth and campaign

### Step 0a — Resolve auth (machine-level)

**Per-machine folder access.** First, call `mcp__ccd_directory__request_directory({"path": "~/.hhq"})` to request Cowork access to the per-machine auth folder. The user sees a one-time prompt per Cowork project; approve once and `~/.hhq/` is readable for the rest of the project's history. If declined, set `home_hhq_unavailable = true` and fall back to reading per-project `<project-dir>/.hhq-auth.json` instead. If the tool is unavailable (CLI), treat as approved.

Then read `~/.hhq/machine.json` (or `<project-dir>/.hhq-auth.json` if `home_hhq_unavailable`).

- **Found** → parse `backend_url`, `license_key`, `machine_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but `<project-dir>/.hhq-auth.json` exists** → legacy file from before v0.10. Migrate inline: `mkdir -p ~/.hhq`, copy `<project-dir>/.hhq-auth.json` → `~/.hhq/machine.json`, delete the legacy file. Continue.
- **Not found and no legacy file** → "Looks like you haven't onboarded yet — say 'set me up' to get started." Stop.

If `jwt_expires_at` is past or within 60s of expiry:
1. `POST <backend_url>/api/refresh` with `Authorization: Bearer <old jwt>`. On 200, save the new token + expires_at to `~/.hhq/machine.json` (preserving other fields).
2. On 401, re-activate via `POST /api/activate` with `license_key` + `machine_id`. Save the new token. On 403 license_inactive / machine_limit_reached, tell the user and stop.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk` (`-s` silent, `-k` is harmless and covers any unusual cert situations).

Never log the JWT or licence key.

### Step 0b — Resolve current campaign (project-level)

Use `mcp__ccd_directory__request_directory` to get the project folder. Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` in local Claude Code CLI.

Read `<project-dir>/.hhq-campaign.json` to get `campaign_slug`. If missing, write it with `{"campaign_slug": "default"}` and use `default`.

This skill operates inside that campaign's context — its offer, ICP, signals, batch, and per-contact cooldown state. The user can run a parallel campaign in a separate Cowork project; auth is shared, campaign context is not.

## Remote-skill seam (V1 dogfood)

The ranking step is structured to become a *remote skill* — turning weighted signals + 100 candidates into a top-5 list with reasoning is high-IP and prompt-iterating work, and it'll move to a server-side MCP call (`POST /api/mcp/rank_prospects`) in V2. The endpoint already exists as a stub on the backend.

For V1 dogfood, do the ranking in-context using your own reasoning over the supplied candidates. Treat the **"Rank candidates" phase below** as the remote boundary. Everything before that phase (config load, eligibility filter, pre-filter) and everything after (presentation, edit handling, persistence, handoff) stays local regardless.

## Pre-flight checks

### Step A — Fetch configs

User-level (voice): `GET <backend_url>/api/me/config` — hold the response as `user_config`. Mostly informational here; voice is used in research-and-draft, not surfacing.

Campaign-level (offer / ICP / signals): `GET <backend_url>/api/me/campaigns/<campaign_slug>/config` — hold as `campaign_config`.

- HTTP 200 with `campaign_config` having `offer`, `icp`, `signals.weighted` populated → continue.
- HTTP 200 with `{"config": null}` or missing required fields → "This campaign (`<campaign_slug>`) doesn't have an offer / ICP / signals configured yet. Run `/hhq:new-campaign` if it's brand new, or `offer-review` / `icp-discovery` to fill in the gaps." Stop.
- HTTP 404 `campaign_not_found` → "This project is pinned to campaign `<campaign_slug>` but the backend doesn't have it. Either fix `.hhq-campaign.json` or run `/hhq:new-campaign` to recreate." Stop.
- HTTP 401 → run the auth fallback in Phase 0 once, retry. If it fails again, tell the user to re-onboard.

Use `campaign_config.offer`, `.offer_hook`, `.offer_profile`, `.icp`, `.icp_profile`, `.signals` for ranking.

### Step B — Fetch contacts (campaign-scoped)

`GET <backend_url>/api/me/campaigns/<campaign_slug>/contacts?surfacable=1&per_page=500&page=1`

The `surfacable=1` filter is **fully backend-enforced** in v0.10+:

- **Pipeline stage** — excludes contacts whose `pipeline_stage_id` is set to anything other than the user's Lead stage. Null-stage and explicit-Lead pass.
- **Hard global lockout (7 days)** — excludes any contact whose `contacts.last_contacted_at` (across any campaign) is within the last 7 days. You just messaged them; don't surface again anywhere.
- **Recently-messaged on LinkedIn (30 days)** — excludes contacts whose `last_messaged_at` is within the last 30 days (LinkedIn historical, from the messages CSV digest).
- **Per-campaign cooldown (30 days)** — excludes contacts whose status in *this campaign* is `contacted` / `paused` / `disqualified`, OR whose `campaign_contacts.last_surfaced_at` for this campaign is within the last 30 days.

Each returned contact also includes a `cooldown_warnings` array — empty for most, but for any contact who was surfaced or contacted in **another** campaign within the last 30 days, the array contains:

```json
[
  {
    "campaign_slug": "p2-investors",
    "campaign_name": "Sunburnt Investors",
    "status": "drafted",
    "last_surfaced_at": "2026-04-19T..."
  }
]
```

You'll surface these to the user in Phase 4 as warnings ("Tim was surfaced in Sunburnt Investors 12 days ago — still add to this batch?") so they can swap if it'd look spammy.

If `total > 500`, page through additional pages until you have all of them.

If `total` is `0` → "No surfacable contacts right now in `<campaign_slug>` — either everything's already in your active pipeline, in cooldown, or you haven't imported any yet. Drop a LinkedIn export, business card scan, spreadsheet, or connect Gmail to bring more in." Stop.

The endpoint returns id, slug, first_name, last_name, headline, company, position, email, linkedin_url, pipeline_stage_id, **status (per-campaign)**, **last_surfaced_at (per-campaign)**, last_messaged_at, last_contacted_at, message_count, **cooldown_warnings**.

If checks pass, continue without ceremony — give them 5 prospects.

## Phase 1 — Eligibility (backend-enforced, plus warnings)

In v0.10+ the backend pre-filters out everyone who fails any eligibility rule (status / per-campaign cooldown / hard global 7-day lockout / 30-day LinkedIn recently-messaged / pipeline stage). The list returned by Step B is already eligible — you don't re-filter.

**You DO need to handle two cases the backend can't decide for you:**

1. **Cross-campaign warnings** — for any returned contact with a non-empty `cooldown_warnings` array, you'll flag them when surfacing in Phase 4. They were surfaced or contacted in another campaign within the last 30 days, so adding them here might feel spammy. The user decides whether to keep or swap.
2. **Empty result** — if `total` is 0, surface the empty-state message from Step B and stop.

If `total` is below 5, surface what you have and note it: "Only 4 eligible right now in `<campaign_slug>` — here they are."

**Hard rule:** never surface a contact whose `cooldown_warnings` includes another campaign with `status: "contacted"` and `last_surfaced_at` within 7 days — backend should have already filtered this via `last_contacted_at`, but treat any such row as a defence-in-depth signal: skip silently and log to yourself.

Cooldown values are hard-coded constants (7-day hard global, 30-day per-campaign, 30-day cross-campaign warn). Do not invent config knobs.

## Phase 2 — Pre-filter to ~50–100 candidates

Looking at all eligible contacts, narrow to a candidate pool of roughly 50–100 using fast signal evaluation on the fields available from the list endpoint:

- **Role match**: keyword overlap between `position` / `headline` and `icp.role` / `icp.other`
- **Industry / company match**: keyword overlap between `company` and `icp.industry`
- **Seniority band**: inferred from `position` (Founder / CEO / Director → senior; Manager / Lead → mid; Analyst / Intern / Graduate → junior). Compare against ICP role's implied seniority.

Signals you CANNOT evaluate at this phase (reserve for pass-2 in research-and-draft):

- Post recency or topical relevance (no posts in the contact record)
- Recent role change (only current position is stored)
- Geographic match (no location field)

Be honest about this constraint when reasoning. If the user's most-weighted signal is one we can't evaluate now (e.g. "post topical relevance"), tell them at the end: "Post-relevance is your top signal but I can't see posts at this stage — I'll factor that in during the research step in a moment."

The pre-filter should be permissive — keep anyone with even a moderate match on the highest-weighted evaluable signals. Trim down based on weighted score, not strict pass/fail.

## Phase 3 — Rank candidates (the remote seam)

**This is the remote boundary.** In V2, replace this phase with:
```
POST <backend_url>/api/mcp/rank_prospects
Authorization: Bearer <jwt>

{
  "contact_ids": [<the ~50-100 ids from phase 2>],
  "config": <the user's config>
}
→ { "ranked": [{contact_id, reasoning}, ...] }
```

For V1 dogfood, do this yourself: reason over the candidate pool given the offer, ICP, and weighted signals; pick the top 5; produce a one-line signal-referenced reasoning per pick.

Reasoning quality matters. The reasoning is what gives the user confidence in your pick — it must reference *which signals* drove the rank, in plain language. Examples of good reasoning:

- "Founder at a small satellite company — strong role + industry match for your microgravity offer."
- "Recently joined Gilmour Space as Senior Propulsion Engineer — fresh role, in your target industry."
- "Director at the Australian Space Agency — senior, on-target industry. Likely a longer-cycle conversation."

Do NOT write generic reasoning like "Good fit for your ICP." That's useless to the user.

## Phase 4 — Surface the 5

Present them as a numbered list. For any prospect with non-empty `cooldown_warnings`, append a short warning line beneath the reasoning so the user sees it before confirming:

```
Here are 5 to look at:

1. **<Full Name>** — <Position>, <Company>
   <one-line signal-referenced reasoning>
   <linkedin URL if present>
   ⚠ surfaced in <Campaign Name> <N> days ago — still include?

2. **<Full Name>** — ...
   <reasoning>
   <linkedin URL>

...

Look right? You can say "let's go", drop one ("not Greg"), or swap ("instead of Greg, give me someone in fintech"). To skip a warning, drop that one and I'll replace.
```

Keep formatting clean. No emoji elsewhere. No scores or signal-weight breakdowns. The reasoning IS the signal breakdown, in plain language. The warning is the only extra line.

If a prospect has multiple warnings (e.g. two other campaigns), pick the most recent one for the line. Format the days from `last_surfaced_at` to "today" rounded down.

## Phase 5 — Handle user edits

The user can do one of three things:

- **Confirm** ("let's go", "looks good", "yes", "perfect"): proceed to Phase 6 with the current 5.
- **Drop** ("not Greg", "skip the second one", "drop Marina and Tom"): remove those from the batch, replace each with the next-best candidate from the ranked pool. Re-show the updated 5 and re-confirm.
- **Swap with a constraint** ("instead of Greg, someone in fintech", "swap the analyst for a director"): apply the constraint as an extra filter on the candidate pool, pick the best replacement. Re-show.

Do this conversationally. One round of edits is normal; two is fine; if the user is grinding past three rounds something's off — gently suggest they re-run with a different ICP weighting later.

If after edits the user can't find a good 5, accept fewer (3 or 4 is OK). Don't pad with weak picks.

## Phase 6 — Persist the batch

Once the user confirms the final list:

### Step 6a — Check for an existing open batch

`GET <backend_url>/api/me/campaigns/<campaign_slug>/current-batch`

If `batch` is non-empty, tell the user:

> "You have an open batch in `<campaign_slug>` from <last surfaced_at, formatted readable> — overwrite with a fresh 5? (yes / no)"

If no → suggest they run research-and-draft on the existing batch instead. Stop.
If yes → continue and overwrite below.

### Step 6b — Write the new current-batch

`PUT <backend_url>/api/me/campaigns/<campaign_slug>/current-batch`

Body:
```json
{
  "batch": [
    {
      "contact_id": <id>,
      "surfaced_at": "<ISO timestamp now>",
      "drafted_at": null,
      "reasoning": "<the one-line signal-referenced reasoning from Phase 3>"
    }
    // ... up to 5 entries
  ]
}
```

Expect HTTP 200 with the saved batch echoed. On 422 (e.g. contact_id ownership check), something's wrong — tell the user and stop.

### Step 6c — Update each contact's per-campaign status

For each prospect in the batch, `PUT <backend_url>/api/me/campaigns/<campaign_slug>/contacts/{contactSlug}` with body:

```json
{
  "status": "surfaced",
  "last_surfaced_at": "<ISO timestamp now>"
}
```

This writes to `campaign_contacts` (the per-campaign join row), creating it if missing. The contact's user-level fields are not touched — only this campaign's view of the contact. Other campaigns retain their own status/cooldown for the same person.

Only update prospects whose previous (per-campaign) status was NOT already `contacted`, `paused`, or `disqualified` (defensive — the surfacable filter should have excluded those, but recheck per row). Don't touch any other fields.

## Phase 7 — Hand off to research-and-draft

Close with a short prompt that triggers the next skill:

> "Saved your batch. When you're ready, say 'let's go' or 'draft them' and I'll research each one and write you an opener."

Do NOT do the research yourself in this skill. Do NOT draft any messages. That's all `research-and-draft`.

## Things you must NOT do

- Do NOT do any LinkedIn enrichment, web fetch, or per-prospect research here. Pure list-based ranking.
- Do NOT call `/api/mcp/rank_prospects` in V1 — leave that seam for V2. V1 ranks in-context.
- Do NOT modify `~/.hhq/machine.json` except to update `jwt` and `jwt_expires_at` after a refresh / re-activate.
- Do NOT modify the user's config or the campaign's config.
- Do NOT surface more than 5. Hard cap, also enforced by the backend (`PUT /me/campaigns/{slug}/current-batch` rejects > 5).
- Do NOT pad weak picks to fill 5 — if only 3 are good, surface 3.
- Do NOT show numeric scores or signal-weight breakdowns to the user. The one-line reasoning is the explanation.
- Do NOT bypass the cooldown for any prospect, ever — even via the cross-campaign warning. The user can still choose to skip a warned prospect; you do not auto-skip.
- Do NOT log the JWT, licence key, or auth file contents.
- Do NOT write per-prospect files locally. Per-campaign research / messages live on the backend at `/api/me/campaigns/{slug}/contacts/{contactSlug}`.

## Edge cases to handle gracefully

- **Empty position field on a contact** → still rankable on company/seniority signals; reduce its rank weight slightly.
- **Empty company field** → company-based signals get zero, but role and seniority still rank.
- **Master has fewer than 5 eligible contacts total** → surface what's eligible, be honest about the count.
- **`/api/me/campaigns/{slug}/current-batch` already has an open batch** → handled in Step 6a.
- **User has weighted signals that are all pass-2-only** (e.g. all about post recency / topical) → surface using best-available signals, and explicitly tell the user that their top signals will be applied in the research step rather than now.
- **Signal weights array is empty or malformed** → fall back to equal weighting across the four list-evaluable signals; tell the user once.
- **Backend down mid-flow** → if the GET /me/contacts call succeeded but a later PUT fails, you've shown the user 5 prospects you can't persist. Tell them honestly: "Showed you 5 but couldn't save the batch — backend's having a moment. Try again in a few minutes." Don't half-update individual contacts.
