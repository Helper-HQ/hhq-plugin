---
name: connect-gmail
description: Complete the OAuth flow to connect the user's Gmail to Helper HQ's own OAuth app (extended Gmail access — beta). Triggers when the user says "connect gmail", "connect my gmail to helper hq", "finish gmail connection", "set up extended gmail", or after they receive the "your access is approved" email. Requires the user to have already submitted a request via /hhq:onboard and been approved by an admin (added to Google Cloud Console test users + approved in Filament admin queue, which sends them an email). Opens the Google consent screen in their browser, polls the backend for the connection to land, confirms.
---

# Connect Gmail — Helper HQ extended Gmail access

You are completing the OAuth handshake that connects the user's Gmail account to Helper HQ's own OAuth app. This is the **second half** of extended Gmail access — onboarding (Phase 7.5) submitted the request, an admin approved them and emailed the go-ahead, and now this skill drives the actual consent + token storage.

This skill assumes the user has been **approved**. If they haven't, route them to `/hhq:onboard` to submit a request, or tell them to wait for the approval email.

## When this skill runs

Trigger on:

- "connect gmail"
- "connect my gmail to helper hq"
- "finish gmail connection"
- "set up extended gmail"
- "my gmail access was approved"
- The user explicitly invokes `/hhq:connect-gmail`

Do NOT trigger on generic "set up gmail" — that's an onboarding concern (`/hhq:onboard`).

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

## Phase 1 — Determine current state

Hit two endpoints to figure out where the user is:

```
GET <backend_url>/api/me/gmail/access-request
GET <backend_url>/api/me/gmail/connection
```

Branch on the combined state:

### Already connected

If `/connection` returns `{"connected": true, "email": "..."}`, the user is done.

> "Gmail is already connected as `<email>`. Nothing to do. If you want to disconnect or reconnect, run this skill again with `--reconnect` (coming soon) or message support."

Stop.

### Approved but not yet connected — drive OAuth

If `/access-request` returns `status: "approved"` AND `/connection` returns `connected: false`, drive the OAuth flow. Continue to Phase 2.

### Pending — waiting on admin

If `/access-request` returns `status: "pending"`:

> "Your extended Gmail access request for `<gmail_address>` is still being reviewed (submitted `<requested_at>`). We'll email you when it's approved — usually within a working day. Once you get the email, run `/hhq:connect-gmail` again."

Stop.

### Rejected

If `/access-request` returns `status: "rejected"`:

> "Your earlier request for `<gmail_address>` wasn't approved. Reason: `<rejected_reason>`. To try again with a different address, run `/hhq:onboard` and re-submit during the extended Gmail step."

Stop.

### No request on file

If `/access-request` returns `status: "none"`:

> "You haven't submitted an extended Gmail access request yet. Run `/hhq:request-gmail-access` to submit one — or `/hhq:onboard` if you'd rather do full first-time setup."

Stop.

## Phase 2 — Begin OAuth

Call:

```
POST <backend_url>/api/me/gmail/oauth/begin
Authorization: Bearer <jwt>
```

Response: `{"authorize_url": "...", "state": "...", "expires_in_seconds": 300}`.

Tell the user:

> "Open this URL in your browser to consent. The link expires in 5 minutes:
>
> `<authorize_url>`
>
> Sign in as the Gmail account you registered (`<gmail_address from access-request>`) and approve the consent screen. You'll see a Helper HQ confirmation page when it lands. I'll detect it on this side automatically."

If the user is on Windows and you can run a shell command, you may optionally offer to open the URL for them:

```
powershell -NoProfile -c "Start-Process '<url>'"
```

(Mac: `open '<url>'`. Linux: `xdg-open '<url>'`.)

Don't auto-open without offering — some users want to paste into a different browser profile.

## Phase 3 — Poll for the connection to land

Poll `GET <backend_url>/api/me/gmail/connection` every 3–5 seconds, up to a max of ~5 minutes (the OAuth state TTL). Look for `connected: true`.

While polling, you can stay quiet — don't spam the user with "still waiting" messages. One acknowledgement at the start ("Waiting for you to approve in the browser…") is enough. If 90 seconds pass with no result, check in once: "Still waiting — let me know if you hit any errors on the Google side."

### Connection landed

When the response shows `connected: true`:

> "Done — Gmail connected as `<email>`. Helper HQ now has direct, scoped access (read + label + draft + archive + trash; never send). You're all set for extended Gmail features."

**Then auto-chain into label setup if not already done.** Check `GET <backend_url>/api/me/gmail/labels/config` — if `is_setup: false`, immediately run the `setup-gmail-labels` skill inline:

> "Next, let's set up your Helper HQ Gmail labels — five default labels (To Do / Awaiting Reply / FYI / Notifications / Newsletters) that the auto-rules + triage-inbox use. About 30 seconds."

Then trigger the `setup-gmail-labels` skill. If `is_setup: true` already, skip the setup chain and just close out.

After labels are set up (or if they were already), close with:

> "**Reminder while we're in beta:** Google requires you to re-auth every 7 days until our app verification clears. When that happens, just run `/hhq:connect-gmail` again — takes 30 seconds."

Stop.

### Timeout

If 5 minutes pass and the connection hasn't landed, the OAuth state has expired:

> "The connection link expired before you finished. No problem — run `/hhq:connect-gmail` again to get a fresh link."

Stop.

### Error during poll

If `/connection` returns an error (rare — auth or network), surface it honestly:

> "Couldn't check the connection status: `<error message>`. Try running `/hhq:connect-gmail` again in a minute, or ping support if it persists."

## Things you must NOT do

- Do NOT skip the access-request check. The OAuth begin endpoint will return a URL even if the user isn't approved — but the consent page will fail with `Access blocked` because their address isn't on Google's test users list. Always verify approval first.
- Do NOT ask the user to paste tokens, codes, or anything from the URL bar. The flow is fully server-side; the only thing the user does in the browser is click "Allow" on Google's screen.
- Do NOT poll forever. Cap at 5 minutes — the state token expires anyway.
- Do NOT prompt for a Gmail address here — that's onboarding's job. This skill assumes the address is already on file from the access-request submission.
- Do NOT save anything to `<project-dir>` — the connection lives backend-side, scoped to the user (not the project / session). It works across all the user's Cowork projects automatically.
