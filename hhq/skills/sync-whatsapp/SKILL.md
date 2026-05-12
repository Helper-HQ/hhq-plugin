---
name: sync-whatsapp
description: Walks the user's WhatsApp Web messaging inbox via the Chrome connector to refresh "last messaged" data on their contacts — direct WhatsApp analogue of sync-linkedin-dms. First-run default 6 months back, ongoing 30 days (override with an explicit window like `/hhq:sync-whatsapp 90d`). Captures direction from the inbox preview prefix ("You:" prefix vs not) so ball-in-your-court detection in `/hhq:followups` works for WhatsApp-only conversations. Posts a digest of `{source:"whatsapp", phone?, display_name?, last_inbound_at?, last_outbound_at?, message_count}` per correspondent to `/api/me/messages/import`, then stamps the user's watermark via `/api/me/whatsapp-dm-sync-state`. Header-only: contact label and inbox card metadata only — never reads message bodies. WhatsApp has no public profile URL, so matching is by normalised phone (against contacts.phones from Google Contacts sync) or by display name (against first_name + last_name). Never auto-creates contacts. Triggers on "sync my whatsapp", "sync whatsapp dms", "refresh whatsapp messages", "pull whatsapp", "/hhq:sync-whatsapp". Run AFTER onboard-helperhq and (ideally) at least one /hhq:sync-google-contacts so phone matching has something to match against.
---

# Sync WhatsApp — Sales Helper

You are walking the user's WhatsApp Web messaging inbox via the Chrome connector and folding recent DM activity into Helper HQ. For each conversation in the window, you stamp the contact's `last_messaged_at` (when *they* last messaged the user) and/or `last_contacted_at` (when *the user* last messaged *them*) so the follow-ups queue ranks WhatsApp-quiet contacts the same way it ranks Gmail- and LinkedIn-quiet ones.

This is a one-shot ingestion skill: trigger, run, report, done. Conversational at the edges (window confirmation, partial-progress on rate-limit, final summary).

## Privacy invariant — read this first

The Chrome connector can see anything WhatsApp Web renders. **Helper HQ only reads three things per chat:** the contact label (a name or a phone number — whatever WhatsApp displays in the chat list), the last activity timestamp, and the preview snippet's *direction prefix* ("You: ..." vs anything else). Never read, store, summarise, or display message bodies. Not for "context", not for "research".

If a thread must be opened to resolve anything (rare — almost everything you need is in the chat list itself), read only the contact header at the very top — name and the number underneath if displayed — and immediately discard everything else.

If the user explicitly asks "what did Greg say last week" mid-skill, refuse politely:

> "I only walk the inbox to find out *who* you've been messaging and roughly *when* — I don't read the messages. If you want a deep read, open the conversation in WhatsApp directly."

## When this skill runs

Trigger when the user says any variant of:

- "sync my whatsapp"
- "sync whatsapp dms"
- "refresh whatsapp messages"
- "pull whatsapp"
- "ingest whatsapp messages"
- "update whatsapp"
- "who have I been messaging on whatsapp"

Do NOT trigger from generic phrases like "check my whatsapp" — only on explicit ingest intent. Do NOT trigger if the user is mid-onboarding.

## Phase 0 — Auth and Chrome connector

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory` (no arguments). Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` if not registered.

### Step 0b — Resolve auth (per-project session)

Read `<project-dir>/.hhq-session.json`.

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming. Continue.
- **Neither found** → "No auth — say `/hhq:connect` to link this project (or `/hhq:onboard` if you're brand-new)." Stop.

If `jwt_expires_at` is past or within 60s of expiry, proactively refresh: POST `<backend_url>/api/refresh` with `Authorization: Bearer <old jwt>`. Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json` (preserving other fields).

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key`** from `.hhq-session.json` (NOT a fresh UUID). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID.

### Step 0c — Verify Chrome connector is loaded

Look at the tools available in your current session for any `mcp__Claude_in_Chrome__navigate` / `mcp__Claude_in_Chrome__read_page` / `mcp__Claude_in_Chrome__find`. If those aren't loaded, halt:

> "I need the Chrome connector to walk your WhatsApp Web inbox. Install Claude in Chrome from your Cowork/Claude.ai connector settings, start a new chat, and try again."

### Step 0d — Verify the user is logged into WhatsApp Web

Navigate to `https://web.whatsapp.com/`.

WhatsApp Web has three logged-out states to detect:

- **QR code screen** ("Scan to log in" / "Use WhatsApp on your phone") → halt:
  > "You're not signed into WhatsApp Web in your Chrome session. Open `https://web.whatsapp.com/` in your browser, scan the QR code with your phone (WhatsApp → Settings → Linked Devices → Link a device), then re-run."
- **"Sync in progress" / loading bar over 30s** → wait up to 60s, then halt:
  > "WhatsApp Web is still syncing your conversations. Give it another minute, then re-run."
- **Logged in** → the chat list is visible on the left. Continue.

Don't read the chat content while detecting — the URL state + visible-element presence is enough.

## Phase 1 — Decide the sync window

### Step 1a — Parse args

The user may have invoked with an explicit window:

- `/hhq:sync-whatsapp` → no arg, defaults from watermark
- `/hhq:sync-whatsapp 30d` → 30 days
- `/hhq:sync-whatsapp 90d` → 90 days
- `/hhq:sync-whatsapp 6mo` or `/hhq:sync-whatsapp 180d` → 6 months
- `/hhq:sync-whatsapp 1y` → 365 days

Map: `Nd` = N days, `Nmo` = N × 30 days, `Ny` = N × 365 days. Anything else → ignore and fall through to watermark default.

### Step 1b — Read the watermark

```
GET <backend_url>/api/me/whatsapp-dm-sync-state
Authorization: Bearer <jwt>
```

Returns `{"last_synced_at": "2026-04-15T..." | null}`.

- **`last_synced_at` is null** → first run. Propose 6 months:
  > "First WhatsApp sync — I'll go back **6 months** to catch conversations that have gone quiet so they show up in your follow-ups queue. After today, future runs default to the last 30 days. Sound good? (yes / change window)"
  >
  > If yes → `window_days = 180`. If "change window", parse the user's reply and use that.

- **`last_synced_at` is set** → ongoing. Default to 30 days:
  > "Refreshing the last 30 days of WhatsApp DMs — last synced `<relative time>`."

### Step 1c — Tell the user what's about to happen

> "Walking your WhatsApp Web inbox for the last `<window>` — header-only (contact label + timestamp + direction, never bodies). I'll pace this politely; WhatsApp Web is more aggressive about flagging automation than LinkedIn, so I'll go a touch slower. Estimate: ~3-5 minutes for 30 days, 15-20 minutes for 6 months. I'll show progress as I go."

## Phase 2 — Walk the inbox

### Step 2a — Read the chat list

Navigate to `https://web.whatsapp.com/` if you're not already there.

Use `mcp__Claude_in_Chrome__read_page` to read the visible page. The chat list is in the left pane. Each chat row shows:

- **Contact label** — either a name (as saved in the user's phone book) or a phone number string (when the sender isn't in their address book). WhatsApp Web shows whichever it has.
- **Last activity timestamp** — typically `12:34` (today), `Yesterday`, a day name like `Monday` (within the last week), or `DD/MM/YYYY` (older).
- **Last message preview** — short snippet, prefixed with **"You: "** when the most recent message was outbound. No prefix (just the snippet) when inbound. The preview itself you ignore beyond the prefix.
- **Unread badge** — green circle with a count, sometimes present. You can ignore this for V1.

If the list is completely empty ("Start a new chat" placeholder), halt cleanly:

> "Your WhatsApp Web inbox is empty (or hasn't synced yet). Nothing to do."

### Step 2b — Walk visible chats, then scroll

Build an in-memory digest as a list of rows. For each visible chat row:

1. **Extract the contact label.** Detect whether it parses as a phone (E.164 with `+` prefix, or AU national `04...` / similar). If yes → set `phone` to the label and leave `display_name` null. If no → set `display_name` to the label and leave `phone` null.

2. **Extract last activity timestamp.** Convert to an absolute ISO date:
   - `HH:MM` (numeric time only) → today at that time
   - `Yesterday` → yesterday at 23:59 (we don't have the actual time)
   - A day name (`Monday`, `Tuesday`, …) within last 7 days → the most recent occurrence of that day, at 23:59
   - `DD/MM/YYYY` → that date at noon
   - Anything else / unclear → today's date as a soft fallback (rare)

3. **Compare to window.** If the timestamp is older than `window_days`, you've reached the window edge — stop scrolling, you're done.

4. **Extract the snippet prefix.** If the preview snippet starts with **"You: "** or **"You:"** → outbound (`last_outbound_at` set). Otherwise → inbound (`last_inbound_at` set). If the preview is missing entirely (rare — happens for media-only messages or system events), leave direction null and only stamp neither timestamp; skip this row.

5. **Set `message_count` to 1** for V1. WhatsApp doesn't expose per-conversation message counts in the chat list, and opening the conversation to count would (a) break the header-only discipline and (b) take orders of magnitude more time. The count is a soft signal in HHQ; one is good enough.

6. **Skip group conversations.** Groups appear in the chat list too — usually with a group avatar (multiple silhouettes) and a non-personal name like "Family" or "Team Sales". If you can detect group-ness from the chat row (group avatar, multiple-name preview like "Alice: hey"), skip entirely. Don't add to the digest.

After processing all visible chats, scroll the chat list pane and wait ~2.5 seconds for lazy-load. Re-read. Repeat until either:

- The newest chat after a scroll is older than `window_days` (you've reached the edge), OR
- A scroll yields no new chats (you've hit the end of the inbox), OR
- You've processed 500+ chats (sanity cap — surface honestly and stop)

**Pacing rules:**

- **2.5 seconds wait after each scroll-load** (slightly slower than LinkedIn — WhatsApp Web is touchier).
- If a scroll fails or the page stops responding, wait 5 seconds and retry once. Then bail.
- Show progress every 25 chats: `"Scanned 50 chats, oldest activity 2026-02-14, still going…"`. The user can interrupt at any time.

### Step 2c — Detect rate-limits / soft-blocks

If at any point WhatsApp Web shows a "your account is being challenged" / "verify it's you" prompt, or the page goes blank / loops on a loading state for >30s, halt cleanly:

1. Stop scrolling.
2. POST whatever digest you've already built to `/api/me/messages/import` (Phase 3) — partial progress is better than zero progress.
3. POST a watermark update with `last_synced_at` set to the oldest activity you successfully reached, NOT today. Next run picks up from there.
4. Tell the user honestly:
   > "WhatsApp Web paused me at `<N chats scanned>`. I've saved progress up to `<oldest activity reached>`. Wait an hour or two and run again — I'll resume from there. If this keeps happening, WhatsApp might be flagging the session — open WhatsApp Web normally for a minute and the sense of automation usually goes away."

Stop. Don't retry automatically.

## Phase 3 — POST the digest

For each correspondent in the digest, build a row:

```json
{
  "source": "whatsapp",
  "phone": "+61400123456",
  "display_name": null,
  "last_inbound_at": "2026-05-10T14:00:00Z",
  "last_outbound_at": null,
  "message_count": 1
}
```

`phone` and `display_name` are populated based on what the chat label looked like (Step 2b.1). At least one must be present per row. `last_inbound_at` and `last_outbound_at` are populated based on the direction detected from the preview prefix; at least one must be present.

POST in one batch:

```
POST <backend_url>/api/me/messages/import
Authorization: Bearer <jwt>
Content-Type: application/json

{ "messages": [ <row 1>, <row 2>, ... ] }
```

Expected response shape:

```json
{ "updated": 18, "unchanged": 4, "unmatched": 12, "total": 34 }
```

`unmatched` is correspondents whose phone (or name) doesn't resolve to any HHQ contact. This is *expected* — WhatsApp typically holds many more people than HHQ does (family, friends, suppliers, randoms). Not an error. We **don't** auto-create contacts from WhatsApp activity — flooding HHQ with random correspondents would defeat the prospect-ranking signal.

If the digest is empty (the user's WhatsApp inbox had nothing in the window), skip the POST and report a friendly "nothing in window" message.

## Phase 4 — Stamp the watermark

```
PUT <backend_url>/api/me/whatsapp-dm-sync-state
Authorization: Bearer <jwt>
Content-Type: application/json

{ "last_synced_at": "<now ISO8601>" }
```

This is what tells future runs that this sync happened.

## Phase 5 — Report honestly

After a successful sync, give a brief summary:

> "Synced your last `<window>` of WhatsApp.
>
> - **`<correspondents>` conversations** in the window.
> - **`<updated>` contacts updated** with fresh `last messaged` / `last contacted` dates — `<N>` of those will surface in your next follow-ups loop because they've gone quiet.
> - **`<unmatched>` not in your HHQ contacts** — that's expected (WhatsApp has a lot of people you don't track in your sales pipeline). If any of them should be tracked, add them via `/hhq:ingest-contacts` or `/hhq:log-touch`.
> - *(if any)* **`<unchanged>` already up to date.**
>
> Run this any time — say 'sync my whatsapp' or `/hhq:sync-whatsapp 90d` for a longer window. Tip: run `/hhq:sync-google-contacts` first if phones look unmatched — that backfills phones onto your HHQ contacts so WhatsApp can find them."

## Things you must NOT do

- Do NOT read message bodies. Chat list preview snippet (the prefix only) — that's it. Never open a conversation to read.
- Do NOT send messages on the user's behalf. This is read-only.
- Do NOT exceed the user's specified window. If they said 30 days, stop at 30 days even if scrolling further would surface more conversations.
- Do NOT scrape more than 500 chats in a single run. WhatsApp Web is more aggressive than LinkedIn about flagging automation; the cap is conservative.
- Do NOT retry past a rate-limit / verification signal. Save partial progress and exit.
- Do NOT auto-create HHQ contacts from WhatsApp activity. Unmatched is the right outcome.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` / `jwt_expires_at` after a refresh.
- Do NOT log the JWT, licence key, or auth file contents. Don't echo any message body content even if accidentally read.

## Edge cases to handle gracefully

- **Group conversations** → skip entirely (Step 2b.6). Don't try to model "the user messaged a group" as multiple per-member updates.
- **A contact label is a phone number** → treat as `phone`, run through backend phone match. If they're saved in the user's Google Contacts (and `/hhq:sync-google-contacts` has run), the match will resolve cleanly.
- **A contact label is a name** → treat as `display_name`, fall back to name match. Single-token names (just "Sarah") only match when there's exactly one HHQ contact with that first_name (matcher logic is conservative — multiple matches → unmatched).
- **A chat with someone outside your HHQ contacts** → `unmatched` count. That's the right outcome; do NOT auto-create.
- **A timestamp like "Yesterday" without a time** → resolve to yesterday at 23:59. Slight upper-bound bias is fine.
- **Chat list re-renders on scroll without adding new items** → dedupe by (phone || display_name). Two consecutive no-progress scrolls → treat as end-of-history.
- **Backend POST fails mid-batch** → single batch POST, all-or-nothing. On 5xx, retry once after 2s. On second failure, save the digest to `<project-dir>/.hhq-whatsapp-pending.json` and tell the user to re-run later. Don't stamp the watermark if the POST didn't land.
- **Watermark PUT fails after a successful import** → not fatal. Tell the user "Synced, but couldn't update the sync timestamp — the next run may re-do some work." Don't roll anything back.
- **Multi-device WhatsApp** → the user's primary phone is the source of truth. Whatever WhatsApp Web shows is what we ingest; if it's stale, that's a WhatsApp problem, not ours.
- **WhatsApp Business account vs personal** → both render the same in Web. Skill doesn't differentiate; both get walked.
