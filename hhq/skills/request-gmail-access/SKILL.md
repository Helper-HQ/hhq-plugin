---
name: request-gmail-access
description: Submit (or check) a request for extended Gmail access — the prerequisite for /hhq:connect-gmail. Self-contained alternative to running /hhq:onboard Phase 7.5 just for this one step. Triggers on "request gmail access", "submit gmail access request", "ask for extended gmail", "I want gmail access", "/hhq:request-gmail-access". Idempotent — checks current state first and only submits if no active request exists.
---

# Request Gmail access — Helper HQ extended Gmail access

You are submitting (or surfacing the state of) the user's request for extended Gmail access. This is the **first half** of the two-step Gmail flow:

1. `/hhq:request-gmail-access` (this skill) — submit the request, admin reviews.
2. `/hhq:connect-gmail` — after approval, drive the OAuth handshake.

This skill exists as a standalone path so users with an already-onboarded account don't need to re-run `/hhq:onboard` (which would overwrite their offer / ICP / voice / signals) just to submit this one step.

## When this skill runs

Trigger on:

- "request gmail access"
- "submit gmail access request"
- "ask for extended gmail"
- "I want gmail access"
- "I need to request gmail"
- The user explicitly invokes `/hhq:request-gmail-access`

If the user just says "connect gmail" and no request exists yet, route to this skill first; once it's `approved` (later), they run `/hhq:connect-gmail`.

## Phase 0 — Auth

Use `mcp__ccd_directory__request_directory` (no arguments) to get the project folder. Save as `<project-dir>`.

Read `<project-dir>/.hhq-session.json`.

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. If `jwt_expires_at` is past or within 60s, proactively refresh: POST `<backend_url>/api/refresh` with the existing JWT (accepts expired tokens). Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json`.
- **Not found** → "No auth — say `/hhq:connect` to link this project (or `/hhq:onboard` if you're brand-new)." Stop.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key` from `.hhq-session.json`** (NOT a fresh UUID — reusing the same UUID keeps this idempotent and avoids burning a slot). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop (`session_limit_reached` includes the `/sessions` URL).

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

## Phase 1 — Check current state

```
GET <backend_url>/api/me/gmail/access-request
```

Branch on `status`:

### `none` — no request on file

Continue to Phase 2 to submit.

### `pending` — already in queue

> "Your extended Gmail access request for `<gmail_address>` is already in the queue (submitted `<requested_at>`). We'll email you when it's reviewed — usually within a working day. Once approved, run `/hhq:connect-gmail` to finish the connection."

Stop.

### `approved` — already approved

> "Your extended Gmail access for `<gmail_address>` is already approved. Run `/hhq:connect-gmail` to complete the OAuth handshake."

Stop.

### `rejected` — earlier request was rejected

> "Your earlier request for `<gmail_address>` wasn't approved. Reason: `<rejected_reason>`.
>
> Want to submit a new request? Reply with the Gmail address you'd like to use (it can be the same one or a different one)."

If the user provides an address, continue to Phase 2 with that address. If they decline, stop.

## Phase 2 — Ask for the Gmail address (only for `none` and `rejected` cases)

For the `none` case (no prior request), ask:

> "Which Gmail address would you like extended Helper HQ access for? Paste the full address — for example `you@gmail.com` or `you@your-domain.com` if it's a Workspace account."

Validate the input:

- Must contain `@` and a domain.
- Must look like a real email (basic regex check).
- Reject obviously bogus inputs ("test", "n/a", etc.) and ask again.

## Phase 3 — Submit

```
POST <backend_url>/api/me/gmail/access-request
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "gmail_address": "<the address>"
}
```

Use `curl -sk -X POST ...` via Bash.

Handle the response:

- **HTTP 201** → success. Continue to Phase 4.
- **HTTP 409 `already_approved`** → race condition (state changed between Phase 1 and Phase 3). Show the "approved" message from Phase 1 and stop.
- **HTTP 409 `already_pending`** → race condition. Show the "pending" message from Phase 1 and stop.
- **HTTP 422** → email validation failed server-side. Show the server's message verbatim and loop back to Phase 2.
- **HTTP 401 / network error** → tell the user the backend isn't reachable and suggest a retry. Don't claim success. Stop.

## Phase 4 — Close

> "Submitted. Your request for `<gmail_address>` is now in the queue. We'll email you when it's reviewed — usually within a working day. Once approved, run `/hhq:connect-gmail` to complete the OAuth handshake."

Stop.

## Things you must NOT do

- Do NOT prompt for offer / ICP / voice / signals. This skill is about the access request only — those questions belong in `/hhq:onboard`.
- Do NOT drive the OAuth flow. That's `/hhq:connect-gmail`'s job, and only works once the request is approved.
- Do NOT skip the Phase 1 state check. Submitting without checking risks confusing 409s and a worse error message than we can produce by branching cleanly.
- Do NOT submit on behalf of the user without an explicit address. The skill must always have the user paste or confirm the Gmail address before POSTing.
- Do NOT log the JWT or full session-file contents in chat output.
