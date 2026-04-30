---
name: connect
description: Connect a fresh Cowork project to an already-onboarded Helper HQ account. Paste your licence key, optionally name the project (for the user's session list), pick which campaign to pin this project to (or create a new one). No offer / ICP / voice / signals questions — that's `/hhq:onboard` (first-time setup) or `/hhq:new-campaign` (additional campaigns). Triggers on "connect", "connect this project", "I have a licence key", "link this project", "/hhq:connect".
---

# Connect — Helper HQ

You are connecting an already-onboarded Helper HQ user to a fresh Cowork project. The user has a licence key, has completed the deep onboarding elsewhere (offer / ICP / voice / signals already exist on the backend), and just needs THIS project hooked up to one of their existing campaigns.

This skill creates a new **session** on the user's licence — internally a new row tied to this Cowork project. Each Cowork project gets its own session; the default cap is 5 simultaneous sessions per licence. If the user has run out, the backend returns a helpful 403 pointing them at `/sessions` where they can release one.

**Architecture note (v0.11+).** Auth is per-project, not per-machine. The session file lives at `<project-dir>/.hhq-session.json` and only that one file. There is no `~/.hhq/`, no shared machine identity, no cross-project token. Cowork's sandbox model is the foundation, not the obstacle.

## When this skill runs

Trigger on:
- "connect", "connect this project", "connect to my account"
- "I have a licence key, just connect"
- "link this project to my account"
- "hook this project up"
- `/hhq:connect` invoked directly.

Do NOT trigger if the user has no licence yet — route to `/hhq:onboard` for first-time setup.

## Phase 0 — Get the project folder

Use `mcp__ccd_directory__request_directory` (no arguments) to get the persistent Cowork project folder. The user accepts a permission prompt the first time. Save the path as `<project-dir>`.

If the tool isn't registered (rare CLI case), fall back to `~/.hhq/sales-helper/` and create the directory if missing.

## Phase 1 — Check existing project session

Read `<project-dir>/.hhq-session.json`.

- **Found** with a valid-looking JWT (`jwt_expires_at` in the future) → this project is already connected. Show the user:

  > "This project is already connected. Licence `<licence_key first 4>...<last 4>`, pinned to campaign `<.hhq-campaign.json campaign_slug or 'default'>`.
  >
  > Want to (a) re-pin this project to a different campaign, (b) reconnect with a different licence (releases the existing session, takes a new slot), or (c) cancel?"

  - Re-pin → reuse existing auth, skip to Phase 5.
  - Reconnect → continue to Phase 2.
  - Cancel → stop politely.

- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate inline by renaming the file to `.hhq-session.json`. Then proceed as "Found" above.

- **Neither file found** → continue to Phase 2.

## Phase 2 — Ask for licence key (and optional project label)

> "Paste your Helper HQ licence key — it looks like `hhq_...` or `HHQ-...` and was emailed when you bought.
>
> Optional: what should I call this project? It shows up in your sessions list at helperhq.co/sessions, so you can tell which one is which when you have a few. Skip if not sure."

Validate licence: starts with `hhq_` or `HHQ-`, length ≥ 16. Project label is optional, max 80 chars.

## Phase 3 — Activate

Generate a UUIDv4 as the session UUID (sent as `session_id` to the backend). On Windows: `powershell -NoProfile -c '[guid]::NewGuid().ToString()'`. On Mac/Linux: `uuidgen`. Strip whitespace.

POST to backend:

```
POST https://helperhq.co/api/activate
Content-Type: application/json

{
  "license_key": "<the licence key>",
  "session_id": "<the UUID>",
  "project_label": "<the label or omit>"
}
```

Use `curl -sk -X POST ...` via Bash.

Handle the response:

- **HTTP 200** → parse `token`, `expires_at`, `tier`, `helpers`. Continue to Phase 4.
- **HTTP 404 `license_not_found`** → "That licence key isn't recognised. Re-paste from your purchase email." Loop back to Phase 2.
- **HTTP 403 `license_inactive`** → "Your licence is suspended or expired. Contact `help@helperhq.co`." Stop.
- **HTTP 403 `session_limit_reached`** → relay the backend's message verbatim to the user (it includes the URL to `/sessions`). Stop. The user releases a session in the dashboard, then re-runs `/hhq:connect` here.
- **HTTP 422 / network error** → tell the user the backend isn't reachable, suggest a retry. Don't write a partial session file. Stop.

## Phase 4 — Save the session file

Write `<project-dir>/.hhq-session.json`:

```json
{
  "backend_url": "https://helperhq.co",
  "license_key": "<the licence key>",
  "session_id": "<the UUID>",
  "jwt": "<token>",
  "jwt_expires_at": "<expires_at>",
  "tier": "<lite|pro|elite>",
  "helpers": ["<...>"],
  "project_label": "<the label or null>"
}
```

Update `tier`, `helpers`, `jwt`, `jwt_expires_at` on every successful refresh / re-activate.

## Phase 5 — List campaigns and pin

`GET https://helperhq.co/api/me/campaigns` with `Authorization: Bearer <jwt>`.

Three sub-cases:

### Sub-case A — Empty (no campaigns)

The user is licensed but never completed full onboarding (account has no campaigns yet). Route them:

> "You're connected, but you don't have any campaigns set up yet. Run `/hhq:onboard` to do the first-time setup (offer, ICP, voice, signals). That'll create your default campaign and pin this project to it."

Stop. Don't write `.hhq-campaign.json` — let `onboard-helperhq` handle the pin.

### Sub-case B — Exactly one campaign

> "You have one campaign on file: `default — Default`. Pinning this project to it.
>
> Say 'new' if you'd rather create a new campaign in this project."

Default action: write `<project-dir>/.hhq-campaign.json` with `{"campaign_slug": "default"}`. Skip to Phase 6.

If the user says "new" → invoke `/hhq:new-campaign` inline.

### Sub-case C — Multiple campaigns

> "You have these campaigns:
>
> · `default` — Default
> · `freds-wares` — Fred's Wares
> · `freds-app` — Fred's App
>
> Which should this project be for? (paste a slug, or say 'new' to create a new one)"

- Existing slug → write `<project-dir>/.hhq-campaign.json` with `{"campaign_slug": "<picked>"}`. Skip to Phase 6.
- `new` → invoke `/hhq:new-campaign` inline; it'll overwrite the campaign pin.
- Invalid slug → show the list again, ask once more.

## Phase 6 — Close

> "Connected. This project is now using campaign `<slug>`. Say 'get me the next 5 prospects' to start surfacing, or 'I've got my LinkedIn export' if you have a CSV to import."

Stop.

## Things you must NOT do

- Do NOT touch `~/.hhq/`. Sessions live per-project at `<project-dir>/.hhq-session.json`. There is no shared per-machine auth file in v0.11+.
- Do NOT auto-re-activate when a token refresh fails. The session may have been released by the user from `/sessions` — silently re-activating would consume a new slot. Tell the user to `/hhq:connect` explicitly.
- Do NOT prompt for offer / ICP / voice / signals. Those questions belong in `/hhq:onboard` (first-timer) or `/hhq:new-campaign` (additional campaigns).
- Do NOT log the licence key, JWT, or full session-file contents in chat output. The truncated `<first 4>...<last 4>` form in Phase 1 is the only exception, used to confirm identity.
- Do NOT modify `<project-dir>/.hhq-campaign.json` until activation has succeeded — partial state is worse than no state.
