---
name: sync-google-contacts
description: On-demand trigger for the backend's Google Contacts sync. The sync pulls the user's Google Contacts via the People API (using the same Google OAuth that powers extended Gmail access), matches each person against the user's existing HHQ contacts by email → name+company, and backfills phone numbers + any missing name fields. Strictly backfill-only — never creates new HHQ contacts (Google holds hundreds of auto-saved entries; flooding HHQ with them would be noise). Runs daily on a backend schedule; this skill is the manual trigger for "I just added contacts on my phone and want them now" + status inspection. Triggers when the user says "sync my google contacts", "pull google contacts", "refresh phones from google", "backfill phones", "/hhq:sync-google-contacts". Requires extended Gmail access (HHQ OAuth) WITH the contacts.readonly scope — users connected before v0.25 must reconnect to add the scope. Run AFTER /hhq:connect-gmail.
---

# Sync Google Contacts — phone backfill

You are running an on-demand Google Contacts sync. The actual sync happens server-side — your job is to trigger the backend job, poll until it's done, and report what changed.

The backend already has the user's Google OAuth (granted via `/hhq:connect-gmail`), reuses the same refresh token to call the People API, matches each Google person against existing HHQ contacts by email → name+company, and merges phones + fills blank name fields. This skill is the thin wrapper.

## Why this skill exists

The backend runs a daily scheduled sync, so 99% of the time the user's HHQ contacts already have fresh phone data. But occasionally:

- The user just added someone to their phone's address book and wants the phone in HHQ now (not tomorrow).
- The user reconnected Gmail and wants to verify the contacts scope was granted.
- A new HHQ contact was just created and the user wants their Google Contacts phone number attached immediately.

For all of those, this skill is the right answer. Run it, get the result, move on.

## Phase 0 — Auth

Use `mcp__ccd_directory__request_directory` (no arguments) to get the project folder. Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` if the tool isn't registered.

Read `<project-dir>/.hhq-session.json`.

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming. Continue.
- **Neither** → "No auth — say `/hhq:connect` to link this project (or `/hhq:onboard` if you're brand-new)." Stop.

If `jwt_expires_at` is past or within 60s of expiry, POST `<backend_url>/api/refresh` with `Authorization: Bearer <old jwt>`, save the new JWT.

All API calls use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any call below**, recover ONCE per the standard pattern:

- `token_expired` → POST `/api/refresh`. Retry.
- `session_revoked` or `invalid_token` → POST `/api/activate` with the existing `session_id` + `license_key` (reuse, don't mint). Retry. Tell user: *"Your session had been released — I've re-established it."*
- `license_inactive` → "Contact help@helperhq.co." Stop.

**On a second 401**, surface honestly and stop.

## Phase 1 — Trigger the sync

POST the trigger:

```
POST <backend_url>/api/me/google-contacts/sync
Authorization: Bearer <jwt>
```

Branch on the response:

- **202** with `{ job_id, status: "queued" }` → continue to Phase 2.
- **409 `google_not_connected`** → tell the user:
  > "You haven't connected Google to Helper HQ yet. Run `/hhq:connect-gmail` first, then come back to this."
  Stop.
- **409 `contacts_scope_missing`** → tell the user:
  > "Your Google connection is active but doesn't have permission to read Google Contacts (this scope was added in v0.25). Run `/hhq:connect-gmail` again — Google will prompt you to approve the new permission. Then come back and rerun this."
  Stop.
- **5xx / network error** → "Couldn't reach the backend. Try again in a minute or check the dashboard."

## Phase 2 — Poll until complete

Every 2 seconds, GET the status:

```
GET <backend_url>/api/me/google-contacts/sync/{job_id}
Authorization: Bearer <jwt>
```

Branch on `status`:

- `queued` or `running` → wait 2s, poll again. Cap total wait at 60s. If still not done after 60s, tell the user it's still running in the background and they can check back later (the sync is async — the backend will finish whether the user waits or not).
- `completed` → continue to Phase 3.
- `failed` → see Phase 4.

## Phase 3 — Report the result

The `result` object on a completed job has this shape:

```json
{
  "total_processed": 247,
  "matched_count": 38,
  "phones_added": 22,
  "fields_filled": 4,
  "skipped_no_match": 209,
  "skipped_no_phone": 0,
  "used_full_sync": false
}
```

Render a brief plain-English summary. Lead with what changed; mention skipped only if it's an interesting number.

> "Synced your Google Contacts.
>
> - **`<matched_count>` contacts** matched against your HHQ list.
> - **`<phones_added>` new phone numbers** added (existing primary numbers preserved).
> - **`<fields_filled>` missing name fields** filled in.
> - **`<skipped_no_match>` Google entries** had no matching HHQ contact — these stay in Google only (backfill-only, by design).
>
> Re-run daily by default — say 'sync my google contacts' if you want a fresh pull now."

If `phones_added == 0 && fields_filled == 0`:

> "Synced — nothing changed. Either your Google Contacts has no phones for the people in your HHQ list, or those numbers were already attached from a previous sync. **`<skipped_no_match>` Google entries** had no matching HHQ contact, which is normal (Google holds a lot of auto-saved people; we only fill in numbers for contacts you've already added to HHQ)."

If `total_processed == 0`:

> "Synced — but your Google Contacts came back empty. If that's a surprise, check on contacts.google.com that the account you connected to Helper HQ actually has contacts in it."

## Phase 4 — Handle a failed job

A failed job has `status: "failed"` and an `error_code`. Map to a user message:

- `contacts_scope_missing` → "Your Google connection doesn't have permission to read Google Contacts. Run `/hhq:connect-gmail` again — Google will prompt you to approve the new permission."
- `google_revoked` → "Your Google connection was revoked (likely from Google's account settings). Run `/hhq:connect-gmail` to reconnect."
- `sync_token_expired` → "Google's incremental sync window expired (usually means it's been >7 days since the last sync). I cleared the marker — next run will be a full pull. Try once more."
- `people_api_error` → "Google's People API returned an error. Try again in a few minutes; if it keeps happening, check the dashboard."
- `unhandled` / anything else → "Something went wrong during the sync. Try again, or check the dashboard if it keeps happening."

In every failure case, do NOT auto-retry. The user asked for this sync; they decide whether to retry.

## Things you must NOT do

- Do NOT call the People API directly from the skill — the backend is the only caller.
- Do NOT attempt to create new HHQ contacts from Google data. The backend will not do this, and you should not either. The skill is purely a status reporter.
- Do NOT modify any HHQ contact directly from this skill — the backend applies all changes inside the job.
- Do NOT poll faster than every 2 seconds. The backend job is doing real People API work; rapid polling is just load.
- Do NOT log the JWT, licence key, or auth file contents.

## Edge cases to handle gracefully

- **First-ever sync** → `used_full_sync` will be `true`, and `total_processed` may be in the hundreds. Just report normally; the next run will be much smaller (incremental syncToken kicks in).
- **Sync still running after 60s** → tell the user it's running in the background and they can come back: *"Still going — Google has a lot of contacts. The sync will finish on its own; run me again later for the result, or check `<backend_url>/api/me/google-contacts/sync/<job_id>` directly."*
- **User has no HHQ contacts yet** → `matched_count` will be 0 across the board. That's fine — say so plainly and suggest they run `/hhq:ingest-contacts` first if they haven't.
- **User reconnected Gmail mid-flight** → the daily backend job picks up the new scope automatically. If the user is impatient, this skill triggers an immediate run.
