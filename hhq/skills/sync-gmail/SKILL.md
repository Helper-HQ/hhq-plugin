---
name: sync-gmail
description: Pulls the user's recent Gmail correspondents (last 30 days) via Claude's Gmail MCP connector, builds a per-correspondent digest from message headers (From / To / Date — never bodies), and applies the v0.9 brief's two transitions: (1) existing contacts get last_messaged_at + message_count refreshed and a stage advance to "In conversation" if they're at any pre-conversation stage (Lead, Outreach sent, custom stages with order < in_conversation, or no stage) and the correspondence is bidirectional; (2) brand-new bidirectional correspondents become Lead-stage contacts with source=gmail_messages so surface-next-5 picks them up. Triggers when the user says "sync my gmail", "pull my gmail contacts", "ingest gmail", "refresh from gmail", "update from gmail", or similar. Run AFTER onboard-helperhq (which gates on the Gmail connector being installed). Never reads message bodies — only the From / To / Date headers, by code-discipline. The Gmail MCP connector exposes full thread access, so this is a soft privacy guarantee enforced in this skill, not a hard one enforced by OAuth scope. If the connector isn't loaded in the current session, halts and routes the user back to onboard-helperhq.
---

# Sync Gmail — Sales Helper Lite

You are pulling the user's last 30 days of Gmail correspondents through Claude's Gmail MCP connector and folding them into Helper HQ — updating existing contacts' "last messaged" data and pipeline stage where appropriate, and creating new Lead-stage contacts for bidirectional correspondents who aren't yet in the user's contact list.

This is a one-shot ingestion skill: trigger, run, report, done. Conversational only at the edges (start, finish, any per-row decisions the user wants to make).

## Privacy invariant — read this first

The Gmail MCP connector gives full thread access (including bodies). **Helper HQ only reads the From / To / Date headers.** Never read, store, summarise, or display message bodies. Not for "context", not for "research". Headers only.

This is a soft guarantee — the scope is broader than what we use, and the discipline lives in this skill's instructions rather than in OAuth scope enforcement. Future patches with a CASA-assessed scope-narrow path may tighten this. For now: don't read bodies.

If the user explicitly asks for body content ("what did Greg say last week?"), refuse politely:

> "I only read email headers — sender, recipient, and timestamp — to figure out who you've been in conversation with recently. I never look at message contents. If you want to follow up on something specific, open the thread in Gmail directly."

## When this skill runs

Trigger when the user says any variant of:

- "sync my gmail"
- "pull my gmail contacts"
- "ingest gmail"
- "refresh from gmail"
- "update gmail"
- "import recent emails"
- "get my recent emailers"
- "who have I been emailing lately"

Do NOT trigger if the user is mid-onboarding (the Gmail connector check happens there in Phase 0.5). Do NOT trigger from generic phrases like "check my email" — only on explicit ingest intent.

## Phase 0 — Auth

Use `mcp__ccd_directory__request_directory` (no arguments) to get the project folder. Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` if the tool isn't registered (rare CLI case).

Read `<project-dir>/.hhq-session.json` (per-project session auth, v0.11+).

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming to `.hhq-session.json`. Continue.
- **Neither found** → "No auth — say `/hhq:connect` to link this project (or `/hhq:onboard` if you're brand-new)." Stop.

If `jwt_expires_at` is past or within 60s of expiry, proactively refresh: POST `<backend_url>/api/refresh` with `Authorization: Bearer <old jwt>` (the endpoint accepts expired tokens). Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json` (preserving other fields).

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key` from `.hhq-session.json`** (NOT a fresh UUID — reusing the same UUID keeps this idempotent and avoids burning a slot). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop (`session_limit_reached` includes the `/sessions` URL).

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

## Phase 0.5 — Pick the Gmail backend (Option Y soft fallback)

There are two ways this skill can talk to Gmail:

1. **HHQ Gmail MCP (preferred)** — uses the user's HHQ-owned OAuth connection (extended Gmail access). Single backend call (`POST /api/mcp/gmail/sync_correspondents`) returns the full correspondent digest server-side. Faster, cleaner, scope-enforced privacy.
2. **Cowork generic Gmail connector (fallback)** — uses Claude's built-in Gmail tools (`search_threads`, `get_thread`, etc.). Requires the user to have installed the connector in their Cowork/Claude.ai settings. Phase 1-3 build the digest client-side.

Detection — call:

```
GET <backend_url>/api/me/gmail/connection
Authorization: Bearer <jwt>
```

Branch on the response:

- **`{"connected": true, ...}`** → set `gmail_backend = "hhq"`. The HHQ Gmail MCP is available — skip Phase 1-3 entirely and use the HHQ short-circuit in Phase 3.5 below. **Don't bother checking for Cowork tools** — even if both are present, HHQ wins (Option Y rule).
- **`{"connected": false}`** → set `gmail_backend = "cowork"`. Fall back to the Cowork connector. Now check tool availability: look at the tools in your current session for any `search_threads` / `get_thread` / etc. (typically prefixed with `mcp__<some-uuid>__`). If those exist, continue with Phase 1-3 as written. If not, halt with:
  > "I can't see the Gmail connector in this session, and you don't have extended Gmail access set up either. Two options:
  >
  > - Install the Gmail connector in your Cowork/Claude.ai settings (Settings → Connectors → Gmail), start a new chat, and run 'sync my gmail' again.
  > - Or run `/hhq:onboard` Phase 7.5 to opt into extended Gmail access — Helper HQ's own OAuth path. Once approved (typically a working day), you won't need Cowork's connector for sync."
- **Network error on the connection check** → assume `gmail_backend = "cowork"` and continue with the existing Cowork detection. Don't block sync just because the backend is briefly unreachable.

## Phase 1 — Resolve the user's Gmail address (Cowork path only)

**Skip this phase if `gmail_backend = "hhq"` from Phase 0.5.** Jump to Phase 3.5.

We need the user's own email address to know which side of each message is "us" vs "them." The MCP connector doesn't expose a direct profile lookup, so derive it from any sent message:

1. Use `search_threads` with a query like `in:sent` and a small page size (1-3 threads).
2. For one returned thread, use `get_thread` to read its messages.
3. Find any message in the thread with `From` that looks like a personal address (not a `noreply@` or list-style address). That's the user's Gmail.
4. Cache the address for the rest of this skill invocation.

If the user has zero sent messages (very new Gmail account), ask them directly:

> "Quick — what's the Gmail address you connected? I need it to figure out direction on each thread."

Validate the answer is a syntactically reasonable email; cache it.

## Phase 2 — Search for recent threads (Cowork path only)

**Skip this phase if `gmail_backend = "hhq"`.** Jump to Phase 3.5.

Use `search_threads` with a query like `newer_than:30d` to pull threads from the last 30 days. Cap at ~200 threads — the brief's "active correspondent" definition is about *recent* activity, and 200 is plenty of signal for a 30-day window.

If the search returns more than 200 threads, take the first 200 (the connector typically returns most-recent first).

## Phase 3 — Build the per-correspondent digest (Cowork path only)

**Skip this phase if `gmail_backend = "hhq"`.** Jump to Phase 3.5.

For each thread:

1. `get_thread` to fetch the thread's messages. Read **only** the headers: `From`, `To`, `Cc`, `Date`. Ignore `body`, `snippet`, `subject` content.
2. For each message in the thread, classify direction:
   - If `From` matches the user's cached Gmail address → outgoing (user-initiated).
   - Otherwise → incoming.
3. Pick the "other party" per message:
   - For outgoing: each address in `To` (and optionally `Cc` — keep simple and use only `To` for v0.9). Skip the user's own address if it's in `To` (rare self-loop).
   - For incoming: the address in `From`.
4. Parse the address: `"Display Name" <email@domain>` or bare `email@domain`. Extract email + display name.
5. Update the digest map at key = `lowercase(email)`:
   - `email` — original case from the first occurrence
   - `display_name` — first non-null display name we see
   - `last_emailed_at` — max of (existing, this message's Date)
   - `message_count` — increment
   - `initiated_by_user_count` — increment if this message was outgoing
   - `received_count` — increment if this message was incoming

After walking all threads, the digest is a flat object keyed by lowercase email.

**DO NOT** include any thread subjects, snippets, or bodies in the digest. Privacy invariant.

## Phase 3.5 — HHQ short-circuit (HHQ path only)

**Skip this phase if `gmail_backend = "cowork"`.**

If you got here from Phase 0.5 with `gmail_backend = "hhq"`, the entire Phase 1-3 work happens server-side in a single call. Hit the HHQ Gmail MCP:

```
POST <backend_url>/api/mcp/gmail/sync_correspondents
Authorization: Bearer <jwt>
Content-Type: application/json

{ "window_days": 30 }
```

Response shape (matches what Phase 3 would have built locally):

```json
{
  "user_email": "brad@example.com",
  "window_days": 30,
  "thread_count": 87,
  "correspondents": {
    "greg@magnetorquer.com": {
      "email": "Greg@Magnetorquer.com",
      "display_name": "Greg Cole",
      "last_emailed_at": "2026-04-29T...",
      "message_count": 4,
      "initiated_by_user_count": 2,
      "received_count": 2
    },
    ...
  }
}
```

Use `correspondents` as your digest. Cache `user_email` for any later use. Continue to Phase 4 — the rest of the skill is identical regardless of which backend produced the digest.

**Why this is faster:** the HHQ MCP server is doing 1 list call + N metadata calls in PHP, hitting Gmail with a fresh access token, and returning a complete digest. Saves N+1 round-trips through the Cowork connector. Also enforces the header-only privacy contract by API contract (the endpoint NEVER returns body content), not by skill discipline.

## Phase 4 — Fetch context from the backend

You need three things from the backend before applying transitions:

1. **Pipeline stage IDs + orders** — `GET <backend_url>/api/me/pipeline-stages` → returns `{stages: [{id, slug, name, order, is_default}], pipeline_locked, pipeline_locked_at}`. Cache the `in_conversation` stage's `id` AND `order` by slug. Also build an `{id → order}` map across all stages — you'll use this to decide stage advance by funnel *position*, not slug, so custom stages inserted between Lead and In conversation also auto-advance correctly.
2. **Existing contacts (matchable by email)** — `GET <backend_url>/api/me/contacts?per_page=500&page=1` (page through if `total > 500`). For each contact with a non-null `email`, build a lookup `{lowercase_email: {id, slug, pipeline_stage_id, last_messaged_at, message_count}}` so you can match Gmail correspondents to existing contacts in O(1).

You don't need the heavy fields (research, notes, messages) for sync — the summary endpoint is enough.

## Phase 5 — Apply transitions per the brief

For each correspondent in the digest, decide one of three outcomes:

### Outcome A — Match existing contact

If the correspondent's email matches an existing contact (by lowercased email):

- **Update `last_messaged_at`** if the correspondent's `last_emailed_at` is newer than the existing contact's `last_messaged_at`. If the existing is null, set to the new value.
- **Update `message_count`** to `existing.message_count + correspondent.message_count` (cumulative across syncs).
- **Stage advance check** (order-based, so custom stages between Lead and In conversation also advance correctly): if the correspondence is bidirectional (`initiated_by_user_count >= 1` AND `received_count >= 1`) AND **either** the existing `pipeline_stage_id` is null **or** the existing stage's `order` is *strictly less than* `in_conversation`'s `order` AND the existing stage's slug is not `not_a_fit`, advance to `in_conversation_stage_id`. Otherwise, leave the stage alone. (Contacts already at or past In conversation, at any post-conversation stage like Meeting booked / Customer, or at the terminal `not_a_fit` stage are never auto-moved.)

Build a payload row:

```json
{
  "first_name": "<existing first_name>",
  "email": "<correspondent.email>",
  "last_messaged_at": "<new last_messaged_at>",
  "message_count": <new message_count>,
  "pipeline_stage_id": <new stage id or unchanged>,
  "source": "<existing source — DON'T overwrite to gmail_messages>"
}
```

Note we include `first_name` and `email` so the import endpoint's email-match dedup path catches the existing record (step 1 of the four-step dedup). The endpoint then runs the update path on the matched contact.

### Outcome B — Bidirectional new correspondent → create

If no existing match AND `initiated_by_user_count >= 1` AND `received_count >= 1`:

Build a creation row at Lead stage:

```json
{
  "first_name": "<from display_name, first whitespace-split>",
  "last_name": "<from display_name, second whitespace-split, or null>",
  "email": "<correspondent.email>",
  "source": "gmail_messages",
  "pipeline_stage_id": <lead_stage_id>,
  "last_messaged_at": "<correspondent.last_emailed_at>",
  "message_count": <correspondent.message_count>
}
```

If there's no display name (only a bare email like `greg@magnetorquer.com`), use the local-part as `first_name`: `first_name = "greg"`. Lower-quality but consistent with the brief's "every contact has a first_name" backend requirement.

### Outcome C — Skip

If no existing match AND NOT bidirectional (e.g. one-way newsletter, transactional email, the user only received but never replied) → skip entirely. Don't surface, don't add to the import payload.

The brief specifies bidirectional-only contact creation to avoid drowning the user in noise from newsletters and notification emails.

## Phase 6 — Submit one batch to the import endpoint

POST all the rows from Phase 5 in one request:

```
POST <backend_url>/api/me/contacts/import
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "contacts": [ <row 1>, <row 2>, ... ]
}
```

Expected response shape:

```json
{
  "created": 4,
  "updated": 18,
  "unchanged": 0,
  "review_count": 0,
  "total": 22
}
```

For Gmail sync, `created` = bidirectional new correspondents added at Lead. `updated` = existing contacts whose `last_messaged_at`, `message_count`, or stage moved. `review_count` should typically be 0 for Gmail sync because every row has an `email` (matches step 1 of dedup).

## Phase 7 — Report honestly

After a successful sync, give a brief summary:

> "Synced your last 30 days of Gmail.
>
> - **`<correspondents>` correspondents** in the window.
> - **`<created>` brand-new** added as Leads (you were emailing back and forth, but they weren't in your contacts yet).
> - **`<updated>` updated** with fresh `last messaged` timestamps; *(if any stages advanced)* `<N>` of those moved to **In conversation** because the back-and-forth says you're actively talking.
> - **`<skipped one-way>` newsletter / transactional senders skipped** — one-way correspondents only.
>
> Pipeline view: `https://helperhq.co/pipeline`
>
> Run this any time you want — say 'sync my gmail' or 'refresh from gmail'."

Compute `<skipped one-way>` as `<total digest size> - <total batch posted>`. If zero, omit that line.

## Things you must NOT do

- Do NOT read message bodies, subjects, snippets, or attachments. Headers only — From, To, Date.
- Do NOT store any thread content locally or send any to the backend beyond what's listed in the row schemas.
- Do NOT use the Gmail connector's `create_draft`, `create_label`, or any write tools. Read-only operation.
- Do NOT advance a contact whose current stage is at or past `in_conversation` (by `order`) — never regress In conversation, Meeting booked, Proposal sent, Customer, or any custom post-conversation stage. Never auto-move contacts at `not_a_fit` either; that's a terminal-by-intent stage.
- Do NOT overwrite an existing contact's `source` to `gmail_messages` — preserve whatever it was (LinkedIn, business card, spreadsheet). Source tells us where the contact ORIGINATED, not the most recent touch.
- Do NOT create non-bidirectional new contacts. One-way correspondents (newsletters, notifications) are noise — surfacing them dilutes the prospect ranking.
- Do NOT ingest messages older than 30 days for contact creation. The brief's "active correspondent" window is 30 days hard.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` / `jwt_expires_at` after a refresh.
- Do NOT log the JWT, licence key, or auth file contents. Don't echo any email body content even if accidentally read.

## Edge cases to handle gracefully

- **MCP rate-limited mid-sync** → halt, surface a partial result honestly, suggest the user retry in a few minutes. Don't half-update individual contacts.
- **A thread has only one message and it's outgoing** (user sent, no reply yet) → `received_count = 0`, so this person doesn't qualify as bidirectional. Won't be created. If the email matches an existing contact, last_messaged_at still updates for them (Outcome A applies regardless of bidirectionality).
- **A correspondent appears in multiple threads** → digest naturally aggregates by email, summing counts and taking max date.
- **Display name is in non-Latin script** (e.g. CJK names) → preserve verbatim. The backend handles unicode in slugs.
- **Forwarded threads** where the From in early messages is someone else (the original sender, not the user's correspondent) — for V1, keep simple and treat each message independently. May surface false bidirectionality. If this becomes a quality problem, revisit.
- **User's own messages in `To:` (self-loops)** — skip self.
- **Mailing-list addresses** like `team@company.com` — these may correspond to a generic mailbox, not an individual. They'll go through dedup as normal — could fuzzy-match an existing person at the same company, in which case the queue catches them. Acceptable.
- **`message_count` overflows int range** — never going to happen in 30 days. No guard.
- **Very large digest (>500 correspondents in 30 days)** — the import endpoint handles batches of thousands. POST it all in one request. If you hit a 413, tell the user and chunk by 500.
- **First-time sync vs incremental** — V1 always re-syncs the full 30-day window. There's no incremental "since last sync" mode. Simpler, slightly less efficient, but correct.
