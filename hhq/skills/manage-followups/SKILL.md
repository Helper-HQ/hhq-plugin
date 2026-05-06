---
name: manage-followups
description: Admin Helper — surfaces inbox threads where the user sent the last message more than N days ago (default 5) and no one has replied. Presents a one-pick-at-a-time queue, then drafts a follow-up in the user's voice and pushes it to Gmail as a reply-in-thread draft. Triggers on "what am I waiting on", "manage my followups", "who haven't I heard back from", "follow up on emails", "show pending replies", "/hhq:manage-followups", "/hhq:manage-followups <days>" (custom threshold). Requires extended Gmail access (HHQ OAuth) — routes the user to /hhq:connect-gmail if not connected. Broader than Sales Helper's surface-followups (which is sales-pipeline-only); this one covers ANY inbox thread regardless of relationship_type — suppliers, partners, miscellaneous correspondence, the lot.
---

# Manage Follow-ups — Admin Helper

You are surfacing threads where the user sent the last message and the other party has gone quiet. Triage-then-process pattern: build the queue cheaply, render it, user picks one at a time, you read that thread + draft a follow-up + push it as a Gmail draft.

This is the **broader counterpart** to Sales Helper's `surface-followups`. That one is for sales conversations (in-conversation pipeline stage, prospects only, four-tier ranking). This one is general-purpose Gmail follow-ups — any thread, regardless of who the other party is.

## When this skill runs

Trigger on:

- "what am I waiting on"
- "manage my followups"
- "who haven't I heard back from"
- "follow up on emails"
- "show me pending replies"
- "anyone I'm chasing"
- `/hhq:manage-followups`
- `/hhq:manage-followups 10` (custom threshold in days)

If the user mentions sales / prospects / pipeline explicitly ("follow up on prospects" or "sales follow-ups"), route them to `/hhq:followups` instead — that's the Sales Helper version with pipeline + dossier integration.

If the trigger phrase is ambiguous and the user has both helpers active, ask once: "Sales follow-ups (pipeline contacts only) or general inbox follow-ups?"

## Phase 0 — Auth + Gmail connection check

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory`. Save as `<project-dir>`. Fallback `~/.hhq/sales-helper/`.

### Step 0b — Resolve auth

Read `<project-dir>/.hhq-session.json`. If missing → "No auth — say `/hhq:connect` to link this project (or `/hhq:onboard` if you're brand-new)." Stop.

If `jwt_expires_at` is past or within 60s of expiry, proactively refresh: POST `<backend_url>/api/refresh` with the existing JWT (accepts expired tokens). Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json`.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key` from `.hhq-session.json`** (NOT a fresh UUID — reusing the same UUID keeps this idempotent and avoids burning a slot). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop (`session_limit_reached` includes the `/sessions` URL).

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

### Step 0c — Confirm extended Gmail access

`GET <backend_url>/api/me/gmail/connection`.

- `connected: true` → continue.
- `connected: false` → halt:
  > "Manage-followups uses Helper HQ's direct Gmail integration (extended Gmail access — beta). You're not connected yet. Run `/hhq:onboard` to opt in, or `/hhq:connect-gmail` if you already have approval."

This skill **does not work** with Cowork's generic Gmail connector — it doesn't expose `list_inbox` or the Gmail search syntax we need.

## Phase 1 — Build the queue

### Step 1a — Resolve the threshold

Default: 5 days. The user can override via:
- The slash command argument: `/hhq:manage-followups 10` → threshold = 10 days
- Natural language: "follow-ups older than a week" → 7 days; "last month" → 30 days

Cap at 90 days (Gmail search limit + relevance). Floor at 1 day.

### Step 1b — Find candidate threads

Use the MCP's `list_threads` (the unconstrained variant — it does NOT auto-prepend `in:inbox`, unlike `list_inbox`).

`POST <backend_url>/api/mcp/gmail/list_threads` with:

```json
{
  "max_results": 100,
  "query": "in:sent newer_than:60d older_than:<threshold>d"
}
```

The response includes per-thread `latest_from`, `latest_to`, `latest_date`, `first_from`, etc. — exactly what we need to check who sent last without an extra `get_thread` round-trip.

### Step 1c — Filter to "user sent last"

For each thread in the response:

1. Look at `latest_from`. Parse the email out (e.g. `Brad <brad@example.com>` → `brad@example.com`).
2. Get the user's own Gmail address from `/api/me/gmail/connection` (`email` field — already loaded in Phase 0c).
3. If `latest_from` matches the user's address (case-insensitive) → candidate.
4. Otherwise → skip (they already replied — ball is in user's court — that's `surface-followups` territory if it's a sales contact, but Admin Helper doesn't track those here).
5. Compute `days_since_user_last_sent` from `latest_date` (parse the RFC 2822 date) to today.

**Cap at 30 candidates** to keep render manageable. Sort by `days_since_user_last_sent` descending (oldest / most overdue first).

### Step 1d — Enrich with sender context

For each candidate, look up the recipient (the other party — not the user) against `/api/me/contacts?per_page=500`. Build the same lookup map as `triage-inbox`. Surface relationship_type + dossier (`contact_profile`) where present — gives the user a quick "what was I talking to them about" memory jog without re-reading the thread.

If the recipient isn't a contact, just use the display name from the To header.

If the contacts lookup fails (backend down), continue without enrichment.

## Phase 2 — Render the queue

```
**Manage follow-ups — 14 threads (last message you sent ≥ 5 days ago)**

  1. Greg Cole (prospect) — Re: Magnetorquer testing — 12 days ago
  2. Sarah Khan (prospect) — Pricing for 50 seats — 9 days ago
  3. Marcus Jones — Tuesday call confirmation — 8 days ago
  4. Anna Patel (customer) — Re: Q4 review — 7 days ago
  5. Stripe support — Account verification — 6 days ago
  6. (plus 9 more — say "show all" to see them)

**Pick one to follow up on:** number (1-5), name, "show all", or "stop".
```

Notes on rendering:
- **Cap at 5 visible by default.** Tighter than triage-inbox because the user is acting on each item one at a time, not bulk-approving.
- Show "(plus N more)" if the queue is bigger.
- Order: oldest-first (most overdue at the top).
- One line per thread: `[N]. <recipient name> (<badge if known contact>) — <subject> — <days> days ago`.
- Badges: `prospect`, `customer`, `partner`, `supplier`, `personal`, `email_contact`. No badge for unknowns.
- **No thread IDs visible.** Track them internally.

## Phase 3 — User picks one

Accept:
- A number (1-N) → that thread
- A name ("Greg" or "Sarah Khan") → match by recipient name
- "Show all" → re-render with all candidates visible (no cap)
- "Stop" / "done" / "cancel" → exit cleanly with brief summary

If the name reference is ambiguous (multiple Sarahs in the queue), ask: "Which Sarah — Khan or Patel?"

If the number is out of range, surface honestly and re-prompt.

## Phase 4 — Per-pick processing

For the picked thread, run the same drafting flow as the `draft-reply` skill — the rules are identical (Gmail thread is Gmail thread; we just got here a different way).

### Step 4a — Read the thread (full bodies)

Phase 1 only pulled metadata via `list_threads` (cheap header-only fetches per thread). For the picked thread, you now need the full bodies to draft a meaningful follow-up.

`POST <backend_url>/api/mcp/gmail/get_thread` with `{"thread_id": "<id>"}`. Response has all messages with bodies (base64url-encoded in `messages[].payload.body.data` — decode + walk multipart parts as needed).

Bodies live in your context for this single pick only; **never persist them**. Same v0.13 privacy contract as `surface-followups` and `draft-reply`.

### Step 4b — Read the user's voice + recipient context

Same as `draft-reply` Phase 3:
- `GET /api/me/config` for `voice_profile`. If null, draft generic and warn at the end.
- The recipient's contact context is already enriched from Phase 1d. Use the dossier (`contact_profile`) if available — it's the most valuable signal for "what's our actual relationship + history".

### Step 4c — Draft the follow-up

Same hard rules as `draft-reply` Phase 4:
- Reference the conversation specifically. **In a follow-up especially: name what you previously asked or proposed and acknowledge the gap.** Examples:
  - "Hi Sarah — bumping this in case it got buried. The 50-seat pricing question from a couple weeks back."
  - "Hey Greg, following up on the testing schedule. No rush — let me know if there's a better time to chase this."
- Match the user's voice. Use `do` / `dont` / `phrases` from the voice profile.
- **Default tone for follow-ups: light, low-pressure, acknowledges the gap without guilt-tripping.** People don't reply for a million reasons; the follow-up should make replying easy, not awkward.
- Default short — 2-3 sentences for a follow-up. Less is more.
- Single-name signoff matching the user's style.
- NO subject prefix in the body — the backend handles `Re:` automatically.

### Step 4d — Show + approve

Same render shape as `draft-reply` Phase 5:

```
**Follow-up to Sarah Khan (Re: Pricing for 50 seats — 9 days since you wrote):**

Hi Sarah,

Bumping this in case it got buried — the 50-seat pricing question
from a couple weeks back. No rush; happy to wait if you're heads-down
on something.

Brad

---
**Approve?** (yes — push to Gmail / edit / skip / stop)
```

### Step 4e — Apply the user's choice

- **Yes / approve** → `POST /api/mcp/gmail/push_draft` with `{"thread_id": "...", "body": "..."}`. On success, brief confirm: "Draft pushed. Open Gmail to send when you're ready." Then ask: "Next one? (number / name / done)".
- **Edit / "change X"** → conversational re-draft loop, same as `draft-reply`. Re-render and ask again.
- **Skip** → don't push anything. Move on: "Skipped. Next one? (number / name / done)".
- **Stop / done / cancel** → exit. Render the session summary (Phase 5).

## Phase 5 — Session summary

When the user is done:

> "Manage-followups done.
>
> - **3 drafts** pushed to Gmail (Sarah Khan, Marcus Jones, Anna Patel) — review in your Drafts folder and click send when ready.
> - **2 skipped** (Greg Cole, Stripe support).
> - **9 left** in the queue — run again any time.
>
> Reminder: Helper HQ never sends on your behalf. Drafts are sitting in Gmail waiting for your review."

If zero drafts pushed (user stopped immediately or skipped everything), drop the "drafts pushed" line. If the queue had nothing in it from Phase 1, skip the whole skill flow and just say:
> "Nothing in your follow-up queue — every recent thread you sent has been replied to. Nice."

## Things you must NOT do

- Do NOT auto-send. Helper HQ never sends. Always pushes to drafts. Same v0.13 product principle.
- Do NOT persist message bodies. Bodies live in your context only for the duration of the pick. Same privacy contract as `draft-reply` and `surface-followups`.
- Do NOT process every thread automatically. Triage-then-process means user picks each one. No "draft follow-ups for all 14" bulk path — every reply needs the user's eyes before it goes anywhere near send.
- Do NOT auto-skip threads from specific senders (e.g. mailing-list addresses). Some "we sent the last message" threads are intentional non-replies (e.g. you sent Stripe a question and they're a slow support team). Surface them all and let the user decide.
- Do NOT confuse the user with Sales Helper's `surface-followups`. If the trigger is ambiguous and the user has Sales Helper active, ask once which they want.
- Do NOT modify pipeline stages, dossiers, or campaign_contacts in this skill. Admin Helper's manage-followups is purely Gmail-side. The Sales Helper followups skill writes to those tables; this one doesn't.
- Do NOT push more than one draft per thread per session. If the user picks the same thread twice somehow, the second pick should re-render the existing draft (or surface "you already pushed a draft for this — open Gmail to send / edit there").
- Do NOT modify `<project-dir>/.hhq-session.json` except for JWT refresh.
- Do NOT run on the Cowork generic Gmail connector. Phase 0c gates on extended Gmail access.

## Edge cases to handle gracefully

- **Empty queue (no threads where user is last sender + >N days)** → "Nothing to follow up on — you're all caught up." Stop.
- **Threshold not parseable from the user's phrase** → fall back to default 5, mention briefly: "Using default 5-day threshold."
- **Thread fetched in Phase 1 was modified between fetch and pick** (e.g. they replied while user was deciding) → re-fetch on pick, if the latest message is now from the other party, tell the user: "They actually replied to this since I last looked — opening it in Gmail makes more sense than drafting a follow-up." Skip and move on.
- **Push_draft fails for a thread** → surface the error code (502 `gmail_api_error` etc.) honestly, don't break the session. User can retry the skill or move on to the next pick.
- **Thread is a no-reply automated chain** (e.g. you emailed `noreply@`, they obviously won't reply) → this surfaces in the queue. You can leave it to the user to skip — or in render, mark `(noreply address — won't get a response)` if the recipient looks like a noreply pattern. Helper, not blocker.
- **Recipient enrichment fails (contacts backend down)** → continue without badges. Don't block the queue on it.
- **Mailing-list threads where the user "replied to all"** — these can pollute the queue. If the recipient is clearly a list address (`@googlegroups.com`, etc.), still surface but without expecting a follow-up to be useful. User decides.
- **The user's Gmail address resolution fails** (connection's `email` field is null somehow) → re-fetch via `GmailService::fetchUserEmail` is implicit in the MCP. If you genuinely can't resolve it, fall back to asking: "Quick — confirm the Gmail address you connected? I need it to figure out which messages were yours." Cache for the session.
