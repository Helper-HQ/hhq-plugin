---
name: remap-pipeline
description: Walks the user through editing their pipeline stages — rename, reorder, add custom stages, or delete custom stages. Used to map the seven Helper HQ defaults onto the user's real sales process. Triggers when the user says "remap pipeline", "edit my pipeline", "edit my stages", "change my pipeline", "add a pipeline stage", "rename a stage", "my pipeline doesn't match", or invokes /hhq:remap-pipeline. Unlocks the pipeline server-side, walks the user through each desired change, then re-locks. Default stages can be renamed and reordered but never deleted (sync-gmail's stage-advance keys off them by slug). Custom stages get auto-generated `custom_*` slugs and are inert to sync-gmail's auto-advance, so insert them anywhere in the funnel without breaking automation.
---

# Remap pipeline — Sales Helper Lite

You are walking the user through editing their pipeline stages — the lanes their contacts live in. The default seven (Lead, Outreach sent, In conversation, Meeting booked, Proposal sent, Customer, Not a fit) cover most B2B sales motions, but real users have real flows that often look different. This skill lets them map onto the defaults via rename / reorder / add / delete, then locks the shape so it stays stable.

This is a conversational skill. One change per turn. Confirm before destructive actions (delete, large reorders). Re-lock at the end so the pipeline doesn't drift.

## When this skill runs

Trigger when the user says any variant of:

- "remap my pipeline"
- "edit my pipeline stages"
- "change my pipeline"
- "add a pipeline stage"
- "rename a stage"
- "my pipeline doesn't match how I sell"
- "let's redo the stages"
- "my stages are wrong"

Also trigger inline if `ingest-contacts` detects the user's CRM stage labels don't match the defaults AND the user replies with intent to remap ("yeah let me fix the pipeline first").

## Phase 0 — Auth

Use `mcp__ccd_directory__request_directory` (no arguments) to get the project folder. Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` if the tool isn't registered (rare CLI case).

Read `<project-dir>/.hhq-session.json` (per-project session auth, v0.11+).

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming to `.hhq-session.json`. Continue.
- **Neither found** → "No auth — say `/hhq:connect` to link this project (or `/hhq:onboard` if you're brand-new)." Stop.

If `jwt_expires_at` is past or within 60s of expiry, proactively refresh: POST `<backend_url>/api/refresh` with the existing JWT (accepts expired tokens). Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json`.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key` from `.hhq-session.json`** (NOT a fresh UUID — reusing the same UUID keeps this idempotent and avoids burning a slot). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop (`session_limit_reached` includes the `/sessions` URL).

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

## Phase 1 — Show current state and unlock

```
GET <backend_url>/api/me/pipeline-stages
```

Response shape:

```json
{
  "stages": [
    {"id": 1, "slug": "lead", "name": "Lead", "order": 1, "is_default": true},
    {"id": 2, "slug": "outreach_sent", "name": "Outreach sent", "order": 2, "is_default": true},
    ...
  ],
  "pipeline_locked": true,
  "pipeline_locked_at": "2026-04-29T10:00:00+00:00"
}
```

Show the current pipeline:

```
Your pipeline right now:

  1. Lead              (default)
  2. Outreach sent     (default)
  3. In conversation   (default)
  4. Meeting booked    (default)
  5. Proposal sent     (default)
  6. Customer          (default)
  7. Not a fit         (default)

What would you like to change? You can:
  • rename a stage         ("rename Lead to Prospecting")
  • reorder                ("move Customer above Proposal sent")
  • add a custom stage     ("add Negotiating after Proposal sent")
  • delete a custom stage  (only stages you've added — defaults stay forever)
  • show again
  • done

(Defaults can be renamed and reordered, but not deleted — they're load-bearing for Gmail sync's auto-advance and contact import routing.)
```

If `pipeline_locked` is `true`, unlock first:

```
POST <backend_url>/api/me/pipeline-stages/unlock
```

If unlock returns non-2xx, tell the user something went wrong server-side and stop. Don't silently proceed with a locked pipeline.

## Phase 2 — Edit loop

Each turn, parse the user's intent into one of: rename, reorder, add, delete, show, done, cancel.

### Rename

User: "rename Lead to Prospecting" / "call Lead 'Prospecting' instead" / "Outreach sent → Discovery Complete"

Confirm:

> "Rename **Lead** → **Prospecting**? (yes / no)"

On yes:

```
PATCH <backend_url>/api/me/pipeline-stages/<id>
Authorization: Bearer <jwt>
Content-Type: application/json

{"name": "Prospecting"}
```

Note that the *slug* (`lead`) stays as-is — it's the internal anchor that sync-gmail and import routing use. The user-facing label is what changes. Don't volunteer this; only mention if the user asks why their renamed stage is "still treated as Lead by Gmail sync" (it is, deliberately).

### Reorder

User: "move Customer above Proposal sent" / "swap In conversation and Meeting booked" / "put Not a fit at the top"

For single-position moves, recompute the full ordered list locally, then PUT:

```
PUT <backend_url>/api/me/pipeline-stages/reorder
Authorization: Bearer <jwt>
Content-Type: application/json

{"ordered_ids": [1, 2, 3, 5, 4, 6, 7]}
```

`ordered_ids` must contain *every* stage id for this user, exactly once. The endpoint returns the full updated list.

For larger reorders ("rearrange them all"), show the user the current order numbered, ask them to type the desired order ("3, 1, 2, 4, 5, 6, 7" or by name), recompute, confirm, PUT.

After PUT, show the new order.

### Add a custom stage

User: "add Negotiating after Proposal sent" / "I need a Demo stage" / "add Qualified between Outreach sent and In conversation"

Resolve `after_stage_id` from the user's positional cue. If they don't say where, default to "after the last default stage you'd put it next to" and confirm before posting.

Confirm:

> "Add **Negotiating** after Proposal sent? It'll be inserted between Proposal sent and Customer. (yes / no)"

On yes:

```
POST <backend_url>/api/me/pipeline-stages
Authorization: Bearer <jwt>
Content-Type: application/json

{"name": "Negotiating", "after_stage_id": <proposal_sent_id>}
```

Response: `{stage: {id, slug, name, order, is_default}, pipeline_locked}`. The slug will be `custom_negotiating`. Show the new full list.

If the user names a stage that collides with a default name ("add a Lead stage"), explain that one already exists and offer to rename it instead.

### Delete a custom stage

User: "delete Negotiating" / "remove the Qualified stage"

Look up the stage by name (case-insensitive). If `is_default` is true, refuse:

> "Default stages can't be deleted — they're load-bearing for Gmail sync's auto-advance and contact import routing. I can rename it if a different label would help."

Otherwise, count contacts at that stage from the GET cache (or do a quick `GET /api/me/contacts?per_page=500` filter — but only if the user wants to know; don't pre-fetch unprompted). Warn:

> "Delete **Negotiating**? Any contacts at that stage will move to **Lead**. (yes / no)"

On yes:

```
DELETE <backend_url>/api/me/pipeline-stages/<id>
Authorization: Bearer <jwt>
```

Response: `{ok: true}`. Show the new list.

### Show

User: "show again" / "where are we" / "what's the current pipeline"

Re-fetch and display the current ordered list with `(default)` / `(custom)` markers.

### Done / cancel

User: "done" / "that's it" / "save and lock" / "finish"

Re-lock:

```
POST <backend_url>/api/me/pipeline-stages/lock
```

Show a final summary:

> "Locked in. Your pipeline:
>
>   1. Prospecting       (default — was Lead)
>   2. Discovery Complete (default — was Outreach sent)
>   3. Qualified         (custom)
>   4. In conversation   (default)
>   5. Meeting booked    (default)
>   6. Negotiating       (custom)
>   7. Proposal sent     (default)
>   8. Customer          (default)
>   9. Paused            (custom)
>  10. Not a fit         (default)
>
> Stages stay locked until you re-run /hhq:remap-pipeline. Renames are still allowed any time; structural changes (add / delete / reorder) need a remap."

User: "cancel" / "actually nevermind" / "abort"

Re-lock without summarising changes; tell them the changes they made before cancelling are still in (rename / add / delete / reorder all hit the API as you went). The lock just freezes the current state.

## Phase 3 — Errors and edges

- **HTTP 423 `pipeline_locked`** — shouldn't happen if Phase 1's unlock worked; if it does, retry the unlock once. If still locked, surface honestly and stop.
- **HTTP 422 `pipeline_stage_default`** on delete — the user tried to delete a default. Recover with the "defaults can't be deleted" copy and continue.
- **HTTP 422 `after_stage_not_found`** on add — usually a stale stage id from a prior fetch. Re-GET the list, ask the user to pick again.
- **HTTP 422 `reorder_set_mismatch`** on reorder — your `ordered_ids` is missing or extra. Re-GET, recompute, retry once. If it fails again, tell the user something went wrong and stop.
- **User exits unexpectedly** (closes the chat, says "stop" mid-edit) — the lock state stays however it currently is. On the next run, Phase 1 will detect unlocked state and offer to re-lock as the first action.

## Things you must NOT do

- Do NOT delete a default stage. Even if the user insists. Explain the constraint and offer rename instead.
- Do NOT batch many changes into one confirmation. One change, one confirmation, one API call. The point of this skill is care.
- Do NOT modify the slug field directly — slugs are derived server-side from the name on create, and stable forever after. Don't try to PATCH a slug.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` / `jwt_expires_at` after a refresh.
- Do NOT log the JWT, licence key, or auth file contents.
- Do NOT auto-lock mid-loop. Lock only on `done` / `cancel`. The user can do many edits before locking.
- Do NOT volunteer the slug-vs-name distinction unless the user explicitly asks. It's an internal implementation detail — surfacing it adds complexity without value.

## Edge cases to handle gracefully

- **User wants more than the current 7 + custom shape** — fine, keep adding. There's no hard cap server-side; UX gets noisy past ~12 stages but that's their call.
- **User wants to delete every custom stage and start over** — walk through them one by one with confirmation. Don't add a "delete all" shortcut.
- **User asks "what's a slug?"** — explain briefly: "Slug is the internal name our automation uses to find a stage. Defaults like `lead` and `in_conversation` are anchors for Gmail sync — that's why we keep them around even after rename. Custom stages get a `custom_<name>` slug auto-generated; you don't pick it."
- **User confused about why their renamed Lead is still treated as Lead by Gmail sync** — confirm yes, the slug `lead` is what the automation reads. The label is cosmetic. If they want different *automation behaviour*, that's not something this skill or rename does.
- **User says "I want sync-gmail to advance to my custom stage instead of In conversation"** — out of scope for this skill, and out of scope for V1 in general. Tell them: "V1 always advances bidirectional Gmail correspondents to In conversation. If you want different stage advance rules, let's flag it as feedback and revisit." Add it to the user's mental backlog, don't promise it.
- **User has 0 custom stages and just wants to rename + reorder** — that's the common case. Default stages are renameable + reorderable. They'll never need to add anything if their sales motion fits the seven-stage shape.
- **User adds 5+ stages in one session** — fine. Each one is its own POST with confirmation.
