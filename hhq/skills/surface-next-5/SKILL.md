---
name: surface-next-5
description: On-demand prospect surfacing. Triggers when the user says "get me the next 5 prospects", "who should I reach out to", "give me 5 leads", "find me some prospects", or similar. Reads `~/.hhq/sales-helper/config.json` (offer, ICP, weighted signals) and `~/.hhq/sales-helper/contacts-master.csv`, filters out ineligible contacts (already-contacted, paused, disqualified, recently-surfaced within a 30-day cooldown), ranks the rest by the user's weighted signals using ONLY data available in the CSV, surfaces the top 5 with one-line signal-referenced reasoning per prospect, lets the user drop / swap, then persists the confirmed batch to `~/.hhq/sales-helper/current-batch.json` and hands off to research-and-draft. Run AFTER onboard-user and ingest-contacts have run at least once.
---

# Surface Next 5 — Sales Helper Lite

You are surfacing the next 5 prospects worth a personal opener, ranked by the user's weighted signals using only what we have in the CSV. This is the on-demand pass-1 filter. Pass-2 enrichment (LinkedIn posts, recent activity, role changes) happens later in `research-and-draft` for the narrowed shortlist of 5.

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

## Auth check (V1 stub)

Before doing anything else, run the auth check.

For V1 dogfood this is a stub that always returns `true`. When the licence backend ships (Stage 2), this becomes a signed-token verification against the locally stored token. Skill code should call the check, branch on the result, and never run skill logic if it returns `false`.

For V1: assume `auth_ok = true` and proceed. Leave a code-comment-style note in your reasoning that this is the auth seam, so future-you knows where to wire the real check.

## Hollow-skill seam (V1 stub)

This skill is structured to become a *hollow skill* in Pro/Elite tiers — the ranking step (turning weighted signals + 100 candidates into a top-5 list with reasoning) is the high-IP part and will move to a server-side MCP tool call (`hhq__rank_prospects`) in V2.

For V1 dogfood, you do the ranking in-context using your own reasoning over the supplied candidates. Treat the **"Rank candidates" phase below** as the hollow boundary. Everything before that phase (config load, eligibility filter, pre-filter to ~50-100 candidates) and everything after that phase (presentation, edit handling, persistence, handoff) stays in the local skill regardless of tier — it's just the ranking call itself that swaps. Keep this seam visible in how you reason.

## Pre-flight checks

Determine the user-level config directory:
- Windows: `%USERPROFILE%\.hhq\sales-helper\`
- macOS / Linux: `~/.hhq/sales-helper/`

Check:
1. **`config.json` exists** → if not, tell the user they need to run onboarding first and stop.
2. **`contacts-master.csv` exists and has at least one data row** → if not, tell the user they need to ingest their LinkedIn export first ("Once your LinkedIn export comes through, drop the CSV in and we'll get started.") and stop.
3. **`config.json` has `offer`, `icp`, and `signals.weighted`** populated. If any are missing, route to onboard-user with a brief explanation.

If checks pass, continue without ceremony — no yes/no gate at the top, the user just asked for 5 prospects, give them 5 prospects.

## Phase 1 — Eligibility filter

Read every row of `contacts-master.csv`. Keep a contact as **eligible** if:

| `status` value | `last_surfaced_date` | Eligible? |
|---|---|---|
| `new` | (any) | ✅ yes |
| `surfaced` | within last 30 days | ❌ no — cooldown |
| `surfaced` | over 30 days ago | ✅ yes — eligible again |
| `drafted` | within last 30 days | ❌ no — already in user's queue |
| `drafted` | over 30 days ago | ✅ yes — old draft, never sent |
| `contacted` | (any) | ❌ no — already messaged |
| `paused` | (any) | ❌ no — user explicitly paused |
| `disqualified` | (any) | ❌ no |

Cooldown is 30 days, hard-coded for V1. Do not invent a config knob for this.

If after filtering there are **zero eligible contacts**, tell the user honestly:

> "Nothing eligible right now — looks like everyone's either already contacted, paused, or in the 30-day cooldown after a recent surface. Either wait a bit, mark some as paused/disqualified, or import a fresh export."

And stop.

If eligible count is **less than 5**, surface what you have and note it: "Only 4 eligible right now — here they are."

## Phase 2 — Pre-filter to ~50–100 candidates (CSV-only signals)

Looking at all eligible contacts, narrow to a candidate pool of roughly 50–100 using fast, CSV-only signal evaluation. The signals you can actually evaluate from the CSV:

- **Role match**: keyword overlap between the contact's `Position` and the user's `icp.role` (and any `icp.other` text)
- **Industry / company match**: keyword overlap between the contact's `Company` and the user's `icp.industry` (and any company-name patterns implied by ICP)
- **Seniority band**: inferred from the `Position` string (Founder / CEO / Director → senior; Manager / Lead → mid; Analyst / Intern / Graduate → junior). Compare against ICP role's implied seniority.
- **Connected-on recency**: how recently the user connected with this person on LinkedIn. Recent (last 90 days) is a soft warm-signal — they may remember accepting the request.

Signals you CANNOT evaluate from the CSV (reserve for pass-2 in research-and-draft):

- Post recency or topical relevance (no posts in the CSV)
- Recent role change (we only have current position, not historical)
- Geographic match (no location in the CSV unless inferable from company)

Be honest about this constraint when reasoning. If the user's most-weighted signal is one we can't evaluate from CSV (e.g. "post topical relevance"), tell them at the end: "Post-relevance is your top signal but I can't see posts at this stage — I'll factor that in during the research step in a moment."

The pre-filter should be permissive — keep anyone with even a moderate match on the highest-weighted CSV-evaluable signals. Trim down based on weighted score, not strict pass/fail.

## Phase 3 — Rank candidates (the hollow seam)

**This is the hollow boundary.** In V2, replace this phase with an MCP call:
```
hhq__rank_prospects({
  candidates: [...],         // the ~50-100 from phase 2
  offer: config.offer,
  icp: config.icp,
  signals_weighted: config.signals.weighted
}) → { ranked: [{slug, score, reasoning}, ...] }
```

For V1 dogfood, you do this yourself: reason over the candidate pool given the offer, ICP, and weighted signals; pick the top 5; produce a one-line signal-referenced reasoning per pick.

Reasoning quality matters. The reasoning is what gives the user confidence in your pick — it must reference *which signals* drove the rank, in plain language. Examples of good reasoning:

- "Founder at a small satellite company — strong role + industry match for your microgravity offer."
- "Recently joined Gilmour Space as Senior Propulsion Engineer — fresh role, in your target industry."
- "Director at the Australian Space Agency — senior, on-target industry. Likely a longer-cycle conversation."

Do NOT write generic reasoning like "Good fit for your ICP." That's useless to the user.

## Phase 4 — Surface the 5

Present them as a numbered list:

```
Here are 5 to look at:

1. **<Full Name>** — <Position>, <Company>
   <one-line signal-referenced reasoning>
   <linkedin URL if present>

2. ...

Look right? You can say "let's go", drop one ("not Greg"), or swap ("instead of Greg, give me someone in fintech").
```

Keep formatting clean. No emoji. No more than the URL — no extra metadata, no scores, no signal breakdowns. The reasoning IS the signal breakdown, in plain language.

## Phase 5 — Handle user edits

The user can do one of three things:

- **Confirm** ("let's go", "looks good", "yes", "perfect"): proceed to Phase 6 with the current 5.
- **Drop** ("not Greg", "skip the second one", "drop Marina and Tom"): remove those from the batch, replace each with the next-best candidate from the ranked pool. Re-show the updated 5 and re-confirm.
- **Swap with a constraint** ("instead of Greg, someone in fintech", "swap the analyst for a director"): apply the constraint as an extra filter on the candidate pool, pick the best replacement. Re-show.

Do this conversationally. One round of edits is normal; two is fine; if the user is grinding past three rounds something's off — gently suggest they re-run with a different ICP weighting later.

If after edits the user can't find a good 5, accept fewer (3 or 4 is OK). Don't pad with weak picks.

## Phase 6 — Persist the batch and update master

Once the user confirms the final list:

### Write `~/.hhq/sales-helper/current-batch.json`

```json
{
  "version": 1,
  "surfaced_at": "ISO-8601 timestamp",
  "prospects": [
    {
      "slug": "coleman-greg-magnetorquer",
      "first_name": "Greg",
      "last_name": "Coleman",
      "company": "Magnetorquer Pty Ltd",
      "position": "Founder",
      "url": "https://www.linkedin.com/in/coleman-greg",
      "reasoning_at_surface": "Founder at a small satellite company — strong role + industry match for your microgravity offer."
    }
    // ... 5 entries
  ]
}
```

Overwrite any existing `current-batch.json` — there's only ever one active batch in V1.

### Update `contacts-master.csv`

For each surfaced prospect, set:
- `status = surfaced` (unless they were already past `surfaced` — in which case keep what's there as long as it's not `contacted`/`paused`/`disqualified`)
- `last_surfaced_date = <today's ISO date>`

Do NOT touch `notes` or any other column. Atomic write via temp-file-rename, same as `ingest-contacts`.

## Phase 7 — Hand off to research-and-draft

Close with a short prompt that triggers the next skill:

> "Saved your batch. When you're ready, say 'let's go' or 'draft them' and I'll research each one and write you an opener."

Do NOT do the research yourself in this skill. Do NOT create per-prospect folders. Do NOT draft any messages. That's all `research-and-draft`.

## Things you must NOT do

- Do NOT do any LinkedIn enrichment, web fetch, or per-prospect research here. Pure CSV-based ranking.
- Do NOT create `contacts/<slug>/` folders. That's research-and-draft's job.
- Do NOT modify `config.json`.
- Do NOT call any MCP tools or external APIs in V1.
- Do NOT surface more than 5. Hard cap.
- Do NOT pad weak picks to fill 5 — if only 3 are good, surface 3.
- Do NOT show numeric scores or signal-weight breakdowns to the user. The one-line reasoning is the explanation.
- Do NOT promise the V2 hollow-skill server-side enrichment. The seam is internal.
- Do NOT write outside `~/.hhq/sales-helper/`.
- Do NOT bypass the cooldown for any prospect, ever.

## Edge cases to handle gracefully

- **Empty Position field** → still rankable on company/seniority signals; reduce its rank weight slightly.
- **Empty Company field** → company-based signals get zero, but role and seniority still rank.
- **Master has fewer than 5 eligible contacts total** → surface what's eligible, be honest about the count.
- **`current-batch.json` already exists with un-drafted prospects** → mention this once: "You have an open batch from `<surfaced_at date>` — overwrite with a fresh 5? (yes / no)". If yes, overwrite. If no, suggest running research-and-draft on the existing batch instead.
- **User has weighted signals that are all pass-2-only** (e.g. all about post recency / topical) → surface using best-available CSV signals, and explicitly tell the user that their top signals will be applied in the research step rather than now.
- **Signal weights array is empty or malformed** → fall back to equal weighting across the four CSV-evaluable signals; tell the user once.
