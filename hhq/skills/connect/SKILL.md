---
name: connect
description: Connect a fresh Cowork project to an already-onboarded Helper HQ account. Paste your licence key, pick which campaign to pin this project to (or create a new one). No offer / ICP / voice / signals questions — that's `/hhq:onboard`. Use this when you've already onboarded once on another machine or in another project, and just want this project hooked up. Triggers on "connect", "connect this project", "connect to my account", "I have a licence key", "link this project", "/hhq:connect".
---

# Connect — Helper HQ

You are connecting an already-onboarded Helper HQ user to a fresh Cowork project. The user has a licence key, has completed the deep onboarding (offer / ICP / voice / signals) elsewhere, and just needs THIS project activated and pinned to one of their existing campaigns.

**This skill does NOT ask onboarding questions.** It does activation (or just refreshes auth if the machine is already known), pins this project to a campaign, and stops. If the user hasn't onboarded at all yet (no campaigns on the backend), route them to `/hhq:onboard` instead.

**Per-machine semantics — read this carefully.** A "machine" in Helper HQ is the `machine_id` stored in `~/.hhq/machine.json`. If that file already exists on this laptop (from a prior Cowork project, prior CLI session, or earlier `/hhq:onboard`), this skill **reuses the existing `machine_id`** — it does NOT consume a fresh slot. The whole point of v0.10 auth is one machine = one slot, regardless of how many Cowork projects are open on it. A fresh `machine_id` is only generated when there is genuinely no prior machine identity (first-ever connect on this laptop, or the user explicitly chose project-local auth by declining `~/.hhq/` access).

## When this skill runs

Trigger when the user says:
- "connect"
- "connect this project"
- "connect to my account"
- "connect to helper hq"
- "I have a licence key, just connect"
- "link this project to my account"
- "hook this project up"
- Or invokes `/hhq:connect` directly.

Do NOT trigger if the user is genuinely new (never onboarded). Onboard-user is the right path for first-time setup. The signal: if they say "set me up", "I'm new", "start over", route to `onboard-user`.

## Phase 0 — Auth folder access + project folder

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory` with no `path` argument. The user accepts the project-folder permission prompt (familiar from v0.9). Save the returned path as `<project-dir>`.

If the tool isn't available (CLI), fall back to `~/.hhq/sales-helper/`.

### Step 0b — Request access to `~/.hhq/` — REQUIRED CALL

**You MUST call `mcp__ccd_directory__request_directory({"path": "~/.hhq"})` at this step.** Do not skip it. Do not assume Cowork has already granted or denied access — the call IS the mechanism by which Cowork shows the user a permission prompt. Without this call there is no prompt, no grant, and no way to read or write the per-machine auth file. Skipping it is the bug — every Cowork project would consume its own machine slot, breaking the v0.10 architecture's per-machine semantics.

Three possible outcomes from the call:

- **Tool returns success (user approves the prompt OR Cowork has already granted access from a prior project)** → `~/.hhq/` is now readable and writable in this session. Continue with the canonical auth path at `~/.hhq/machine.json`.
- **Tool returns an error indicating the user declined** → set `home_hhq_unavailable = true`. Auth falls back to `<project-dir>/.hhq-auth.json` for this project. Tell the user once, briefly: "OK, this project will use project-local auth — each Cowork project will need its own activation, consuming one of your machine slots."
- **Tool itself is not registered in this session** (rare; only in some local Claude Code CLI environments where the CCD MCP isn't loaded) → treat as approved. CLI doesn't sandbox `~/.hhq/`. Continue.

Do NOT short-circuit this with phrases like *"the shared ~/.hhq folder isn't accessible from Cowork"* — that conclusion can only be reached by *making the call and getting back a denial*. If you haven't made the call, you don't know.

## Phase 1 — Check existing auth (project-local AND per-machine)

Try both locations and pick the one that yields valid auth. Order matters:

1. Read `<project-dir>/.hhq-auth.json` — if exists, hold as `project_auth`.
2. Read `~/.hhq/machine.json` (skip if `home_hhq_unavailable` was set in Phase 0) — if exists, hold as `machine_auth`.

Then branch on what you found:

### Branch A — Project-local auth exists (`project_auth` non-null)

> "This project is already connected. Licence `<project_auth.license_key first 4 + '...' + last 4>`, machine `<project_auth.machine_id first 8>...`, pinned to campaign `<existing campaign_slug or 'default'>`.
>
> Want to (a) re-pin this project to a different campaign, (b) reconnect with a different licence (overwrites the existing connection), or (c) cancel?"

- **Re-pin** → reuse `project_auth` as your auth, skip to Phase 5.
- **Reconnect** → drop `project_auth`, continue to Phase 2 (fresh licence prompt).
- **Cancel** → stop politely.

### Branch B — Per-machine auth exists (`machine_auth` non-null, no project auth)

This is the **common case** when the user opens a fresh Cowork project on a laptop where they've already run Helper HQ before. Reuse the machine identity:

Tell the user briefly:

> "Found your machine licence at `~/.hhq/machine.json` — reusing the existing machine_id (no new slot consumed). Refreshing your token and pinning this project to a campaign now."

Refresh the JWT (don't generate a new machine_id, don't ask for licence key — we already have both):

```
POST https://hhq.ngrok.dev/api/refresh
Authorization: Bearer <machine_auth.jwt>
```

- **HTTP 200** → parse new `token` and `expires_at`. Update `~/.hhq/machine.json` with the refreshed values (preserve all other fields). Hold the auth and skip to Phase 5.
- **HTTP 401** (token invalid / revoked) → fall through to a re-activate using the same `license_key` + `machine_id`:

  ```
  POST https://hhq.ngrok.dev/api/activate
  { "license_key": "<machine_auth.license_key>", "machine_id": "<machine_auth.machine_id>" }
  ```

  The backend's idempotent existing-machine branch recognises the `machine_id` and does NOT consume a new slot — it just refreshes the JWT. On 200, save and skip to Phase 5. On 403 `machine_limit_reached` → impossible-but-handle-anyway: tell the user something's odd with their slots and to contact support. On 403 `license_inactive` → tell user, stop.
- **Other errors** → tell the user the backend's not responding; don't touch the auth file; stop.

### Branch C — Neither file (`project_auth` and `machine_auth` both null)

Genuinely first-time on this machine (or first time in this project after declining `~/.hhq/` access in Phase 0). Fresh activation needed → continue to Phase 2.

## Phase 2 — Ask for the licence key

Only reached for Branch A "Reconnect" or Branch C above.

Ask:

> "Paste your Helper HQ licence key. It looks like `hhq_...` (or `HHQ-...`) and you got it in your purchase email. This connects this project to your account."

Validate: starts with `hhq_` or `HHQ-`, length ≥ 16 chars. If invalid, ask again with a brief reason.

## Phase 3 — Activate (fresh slot)

This phase is reached only for genuinely new machine identity (Branch C, or Branch A "Reconnect" with a different licence). It DOES consume a machine slot.

Generate a UUIDv4 as the `machine_id`. Use `powershell -NoProfile -c '[guid]::NewGuid().ToString()'` on Windows or `uuidgen` on Mac/Linux. Strip whitespace.

POST to the backend:

```
POST https://hhq.ngrok.dev/api/activate
Content-Type: application/json

{
  "license_key": "<the licence key>",
  "machine_id": "<the UUID>"
}
```

Use `curl -sk -X POST ...` via the Bash tool.

Handle the response:

- **HTTP 200** → parse `token`, `expires_at`, `tier`, `helpers`. Continue to Phase 4.
- **HTTP 404 `license_not_found`** → "That licence key isn't recognised. Re-paste from your purchase email." Loop back to Phase 2.
- **HTTP 403 `license_inactive`** → "Your licence is suspended or expired. Contact `help@helperhq.co`." Stop.
- **HTTP 403 `machine_limit_reached`** → "Your licence is at its machine-slot limit. Contact `help@helperhq.co` to bump the limit, or revoke an old slot via the dashboard." Stop.
- **HTTP 422 / network error** → tell the user the backend isn't reachable, suggest a retry. Don't write a partial auth file. Stop.

## Phase 4 — Save the auth file

Only reached after Phase 3 (fresh activation). For Branch B (token refresh), the auth file was already updated in Phase 1.

**If `home_hhq_unavailable` is NOT set:**

Create `~/.hhq/` if missing (`mkdir -p ~/.hhq`). Write `~/.hhq/machine.json`:

```json
{
  "backend_url": "https://hhq.ngrok.dev",
  "license_key": "<the licence key>",
  "machine_id": "<the UUID>",
  "jwt": "<token>",
  "jwt_expires_at": "<expires_at>",
  "tier": "<lite|pro|elite>",
  "helpers": ["<...>"]
}
```

**If `home_hhq_unavailable` IS set:**

Write the same payload to `<project-dir>/.hhq-auth.json` instead.

Update `tier` and `helpers` on every refresh.

## Phase 5 — List campaigns and pick

`GET https://hhq.ngrok.dev/api/me/campaigns` with `Authorization: Bearer <jwt>`.

Hold the response. Three sub-cases:

### Sub-case A — Empty (no campaigns)

The user is licensed but never completed full onboarding (their account has no campaigns yet). Route them:

> "You're connected, but you don't have any campaigns set up yet. Run `/hhq:onboard` to do the full first-time setup (offer, ICP, voice, signals). That'll create your default campaign and pin this project to it."

Stop. Don't write `.hhq-campaign.json` — let onboard-user handle that.

### Sub-case B — Exactly one campaign (most common after seeded reset)

> "You have one campaign on file: `default — Default`. Pinning this project to it.
>
> If you'd rather create a new campaign in this project, say 'new'. Otherwise this is done."

Default action: write `<project-dir>/.hhq-campaign.json` with `{"campaign_slug": "default"}`. Skip to Phase 6.

If the user says "new" → invoke `/hhq:new-campaign` inline.

### Sub-case C — Multiple campaigns

> "You have these campaigns:
>
> · `default` — Default
> · `sunburnt-investors` — Sunburnt Investors
> · `helper-hq-customers` — Helper HQ Customers
>
> Which one should this project be for? (paste a slug, or say 'new' to create a new one)"

- **They paste an existing slug** → write `<project-dir>/.hhq-campaign.json` with `{"campaign_slug": "<picked>"}`. Skip to Phase 6.
- **They say `new`** → invoke `/hhq:new-campaign` inline. The new-campaign skill will overwrite the campaign pin.
- **Invalid slug** → tell them honestly, show the list again, ask once more.

## Phase 6 — Close

Tell the user briefly:

> "Connected. This project is now using campaign `<slug>`. Say 'get me the next 5 prospects' to start surfacing, or 'I've got my LinkedIn export' if you have one to import."

Stop.

## Things you must NOT do

- Do NOT ask offer / ICP / voice / signals questions. That's `/hhq:onboard` for first-timers and `/hhq:new-campaign` for additional campaigns.
- Do NOT consume a machine slot unnecessarily. If `~/.hhq/machine.json` exists OR if `<project-dir>/.hhq-auth.json` exists with a valid `machine_id`, reuse the existing machine identity — refresh the JWT (and re-activate against the same `machine_id` if refresh fails) rather than generating a new UUID. The backend's `/api/activate` is idempotent for an existing `machine_id`; a fresh UUID is the trigger that consumes a slot.
- Do NOT touch the user's existing campaigns on the backend. This skill only reads campaigns; never PUTs config.
- Do NOT modify `<project-dir>/.hhq-campaign.json` until activation succeeded — partial state is worse than no state.
- Do NOT log the licence key, JWT, or contents of the auth file in chat output. The summary at start of Phase 1 (`<licence_key first 4 + '...' + last 4>`) is the only acceptable exposure, for confirmation purposes.
- Do NOT route to `/hhq:onboard` unless the user genuinely has no campaigns yet (Sub-case A in Phase 5). The whole point of this skill is to skip onboarding for people who've already done it.
