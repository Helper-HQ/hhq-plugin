---
name: sync-linkedin-dms
description: Walks the user's LinkedIn messaging inbox via the Chrome connector to refresh "last messaged" data on their contacts — same shape as sync-gmail but for LinkedIn DMs. First-run default is 6 months back, ongoing default is 30 days (override with an explicit window like `/hhq:sync-linkedin-dms 90d`). Captures direction from the inbox preview prefix ("You: …" vs "Name: …") so ball-in-your-court detection in `/hhq:followups` works for LinkedIn-only conversations. Posts a digest of `{linkedin_url, last_inbound_at?, last_outbound_at?, message_count}` per correspondent to the existing `/api/me/messages/import` endpoint, then stamps the user's sync watermark via `/api/me/linkedin-dm-sync-state`. Header-only: profile URLs and inbox card metadata only — never reads message bodies. Triggers on "sync my linkedin", "sync my linkedin dms", "refresh linkedin messages", "pull linkedin dms", "/hhq:sync-linkedin-dms", or as the surface-followups pre-flight when the watermark is null or older than 12h. Run AFTER onboard-helperhq.
---

# Sync LinkedIn DMs — Sales Helper

You are walking the user's LinkedIn messaging inbox via the Chrome connector and folding recent DM activity into Helper HQ. For each correspondent in the window, you stamp the contact's `last_messaged_at` (when *they* last messaged the user) and/or `last_contacted_at` (when *the user* last messaged *them*) so the follow-ups queue ranks LinkedIn-quiet contacts the same way it ranks Gmail-quiet ones.

This is a one-shot ingestion skill: trigger, run, report, done. Conversational at the edges (window confirmation, partial-progress on rate-limit, final summary).

## Privacy invariant — read this first

The Chrome connector can see anything LinkedIn renders. **Helper HQ only reads three things:** correspondent name, correspondent profile URL (`linkedin.com/in/{slug}`), and the inbox card preview's *direction prefix* and *timestamp*. Never read, store, summarise, or display message bodies. Not for "context", not for "research".

If a thread must be opened to resolve a profile URL (when the inbox card alone doesn't expose it), read the thread header ONLY — the profile-link anchor in the conversation header — and immediately discard everything else. Don't summarise, quote, or surface body content even by accident.

If the user explicitly asks "what did Sarah say last week" mid-skill, refuse politely:

> "I only walk the inbox to find out *who* you've been messaging and roughly *when* — I don't read the messages. If you want a deep read, run `/hhq:followups` and pick Sarah; that loop has a controlled per-pick body read."

## When this skill runs

Trigger when the user says any variant of:

- "sync my linkedin"
- "sync my linkedin dms"
- "refresh linkedin messages"
- "pull linkedin dms"
- "ingest linkedin messages"
- "update linkedin"
- "who have I been messaging on linkedin"

Also auto-triggered as the `surface-followups` pre-flight when both (a) Chrome is loaded and (b) `linkedin_dm_last_synced_at` is null OR older than 12h. In that case the user is already mid-flow — do the sync silently with the default window (6mo if null, 30d otherwise) and only narrate if something goes wrong.

Do NOT trigger from generic phrases like "check my linkedin" — only on explicit ingest intent.

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
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key`** from `.hhq-session.json` (NOT a fresh UUID). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop.

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

### Step 0c — Verify Chrome connector is loaded

Look at the tools available in your current session for any `mcp__Claude_in_Chrome__navigate` / `mcp__Claude_in_Chrome__read_page` / `mcp__Claude_in_Chrome__find`. If those aren't loaded, halt:

> "I need the Chrome connector to walk your LinkedIn messaging inbox. Install Claude in Chrome from your Cowork/Claude.ai connector settings, start a new chat, and try again."

### Step 0d — Verify the user is logged into LinkedIn

Navigate to `https://www.linkedin.com/feed/`. If the page redirects to login or shows a logged-out marketing page, halt:

> "You're not logged into LinkedIn in your Chrome session. Open `https://www.linkedin.com/login`, sign in, then re-run."

If the feed loads, continue. (Don't read the feed content — the URL not bouncing is enough.)

## Phase 1 — Decide the sync window

### Step 1a — Parse args

The user may have invoked with an explicit window:

- `/hhq:sync-linkedin-dms` → no arg, defaults from watermark
- `/hhq:sync-linkedin-dms 30d` → 30 days
- `/hhq:sync-linkedin-dms 90d` → 90 days
- `/hhq:sync-linkedin-dms 6mo` or `/hhq:sync-linkedin-dms 180d` → 6 months
- `/hhq:sync-linkedin-dms 1y` → 365 days

Map: `Nd` = N days, `Nmo` = N × 30 days, `Ny` = N × 365 days. Anything else → ignore and fall through to watermark default.

If the user gave an explicit window, skip Step 1b and use that. Otherwise:

### Step 1b — Read the watermark

```
GET <backend_url>/api/me/linkedin-dm-sync-state
Authorization: Bearer <jwt>
```

Returns `{"last_synced_at": "2026-04-15T..." | null}`.

- **`last_synced_at` is null** → first run. Propose 6 months:
  > "First sync — I'll go back **6 months** to capture conversations that have gone quiet so they show up in your follow-ups queue. After today, future runs default to the last 30 days. Sound good? (yes / change window)"
  > 
  > If yes → `window_days = 180`. If "change window", parse the user's reply (`30d`, `90d`, etc.) and use that.

- **`last_synced_at` is set** → ongoing. Default to 30 days, no confirmation needed (silent default for the auto-run case from `surface-followups`):
  > "Refreshing the last 30 days of LinkedIn DMs — last synced `<relative time, e.g. 2 days ago>`."
  > 
  > Then start scraping. (If the user invoked manually they can interrupt and re-run with an explicit window.)

When invoked as the `surface-followups` auto-run, suppress the "First sync — going back 6 months" prompt only if the user has already confirmed via that flow this session. Otherwise still ask — a 6-month first run is too long to silently kick off.

### Step 1c — Tell the user what's about to happen (manual trigger only)

> "Walking your LinkedIn messaging inbox for the last `<window>` — header-only (correspondent + timestamp + direction, never bodies). I'll pace this politely; LinkedIn rate-limits aggressive scraping. Estimate: ~2-3 minutes for 30 days, 10-15 minutes for 6 months. I'll show progress as I go."

For the auto-run from `surface-followups`, skip this — just start.

## Phase 2 — Walk the inbox

### Step 2a — Open the messaging inbox

Navigate to `https://www.linkedin.com/messaging/`.

Use `mcp__Claude_in_Chrome__read_page` to read the visible page. The inbox is a left-hand list of conversation cards. Each card shows:

- Correspondent name (anchor link, usually `/in/{slug}/` OR the conversation route)
- Last activity timestamp (relative — "2d", "1w", "3mo")
- A snippet of the most recent message, prefixed with **"You:"** if outbound or **"<Name>:"** if inbound (sometimes truncated; sometimes absent for new conversations)

If the page didn't load or shows an empty state ("No messages yet"), halt with a friendly summary and stop.

### Step 2b — Walk visible cards, then scroll

Build an in-memory digest as a map `linkedin_url → {first_name, last_name, last_inbound_at, last_outbound_at, message_count}`.

For each visible conversation card:

1. **Extract correspondent name** from the card.
2. **Extract last activity timestamp.** LinkedIn shows it as relative ("2d", "1w", "Just now", "Apr 12"). Convert to an absolute ISO date by combining with today's date. Use heuristics:
   - "Just now" / "now" / "Xm" → today
   - "Xh" → today
   - "Xd" → today minus X days
   - "Xw" → today minus X × 7 days
   - "Xmo" → today minus X × 30 days
   - "Xy" → today minus X × 365 days
   - "Mon DD" (no year) → assume current year, or previous year if that date is in the future
   - "Mon DD, YYYY" → exact
3. **Compare to window.** If the timestamp is older than `window_days`, you've reached the window edge — stop scrolling, you're done.
4. **Extract the snippet prefix.** If it starts with "You:" or "you:" → outbound. If it starts with "<Name>:" or matches the correspondent's name → inbound. If absent or ambiguous → leave direction null (will only stamp `last_messaged_at` not `last_contacted_at` — conservative).
5. **Resolve the profile URL.** Strategy depends on what the card exposes:
   - **If the card's clickable name links to `/in/{slug}/` directly** → use that URL, normalised to `https://www.linkedin.com/in/{slug}` (strip query strings + trailing slash).
   - **If the card links to `/messaging/thread/{id}/` only** → defer URL resolution. Capture the thread URL plus the visible name + any visible headline; resolve in Step 2c.

6. **Increment a `message_count` for this correspondent.** Each visible card represents one conversation, but LinkedIn doesn't expose the per-conversation message count without opening the thread. Set `message_count` to 1 per conversation for V1 — the existing `messages/import` endpoint requires it but the actual integer is only used as a soft "talkativeness" signal in `surface-next-5`. Don't open threads just to count messages.

After processing all visible cards, check if there are more conversations below. Scroll the inbox pane (use `mcp__Claude_in_Chrome__eval` with `document.querySelector('[class*="msg-conversations-container"]').scrollTop = …` or the equivalent — the scrollable element is the conversation list, not the page) and wait ~2s for lazy-load. Re-read. Repeat until either:

- The newest card after a scroll is older than `window_days` (you've reached the edge), OR
- A scroll yields no new cards (you've hit the end of the user's history), OR
- You've processed 1000+ conversations (sanity cap — surface honestly and stop)

**Pacing rules:**
- 2 second wait after each scroll-load.
- If a scroll fails or the page stops responding, wait 5 seconds and retry once. Then bail.
- Show progress every 50 conversations: `"Scanned 80 conversations, oldest activity 2026-02-14, still going…"`. The user can interrupt at any time.

### Step 2c — Resolve deferred profile URLs (thread-link cards only)

For any card where Step 2b only captured a thread URL (no `/in/{slug}/` anchor on the card itself), you need to open the thread to read the profile link from the conversation header.

For each deferred entry:

1. Navigate to the thread URL (`https://www.linkedin.com/messaging/thread/{id}/`).
2. Use `mcp__Claude_in_Chrome__read_page` and **immediately** find the conversation header's profile link — typically the correspondent's name/avatar at the top of the thread, anchoring to `/in/{slug}/`.
3. Extract that URL only. **Do not read or quote any message body content from the thread.** Even if it scrolls past in the page-read output, ignore it — your task is the header anchor.
4. Pace: 2 seconds between thread opens.

If the thread has no resolvable profile link (rare — group chat, deleted user), skip that conversation entirely (don't add it to the digest).

After all deferred URLs are resolved, navigate back to `https://www.linkedin.com/messaging/` to leave the page in a clean state.

### Step 2d — Detect rate-limits / soft-blocks

If at any point the page redirects to `linkedin.com/checkpoint/`, shows a captcha, shows a "you're being rate limited" banner, or returns a blank/error page, halt cleanly:

1. Stop scrolling / opening threads.
2. POST whatever digest you've already built to `/api/me/messages/import` (Phase 3) — partial progress is better than zero progress.
3. POST a sync-state update with `last_synced_at` set to *now minus the unprocessed window* — i.e. record the watermark at the oldest activity you successfully reached, NOT today. This way the next run picks up where this one stopped.
4. Tell the user honestly:
   > "LinkedIn paused me at `<N conversations scanned>`. I've saved progress up to `<oldest activity reached>`. Wait an hour or two and run again — I'll resume from there."

Stop. Don't retry automatically.

## Phase 3 — POST the digest

For each correspondent in the digest with a resolved `linkedin_url`, build a row:

```json
{
  "linkedin_url": "https://www.linkedin.com/in/coleman-greg",
  "last_inbound_at": "2026-04-20T10:00:00Z",
  "last_outbound_at": "2026-04-25T09:00:00Z",
  "message_count": 1
}
```

Include `last_inbound_at` and/or `last_outbound_at` only when you actually captured direction from the snippet prefix. If direction was ambiguous, just send `last_messaged_at` (the legacy field) — the backend will use it for `last_messaged_at` only and leave `last_contacted_at` alone. Conservative.

POST in one batch:

```
POST <backend_url>/api/me/messages/import
Authorization: Bearer <jwt>
Content-Type: application/json

{ "messages": [ <row 1>, <row 2>, ... ] }
```

Expected response shape:

```json
{
  "updated": 18,
  "unchanged": 4,
  "unmatched": 6,
  "total": 28
}
```

`unmatched` is correspondents whose `linkedin_url` doesn't match any contact in the user's network. Not an error — happens when someone's been chatting with people they're not connected to (LinkedIn lets you DM connection-requesters, group members, etc.).

If the digest is empty (the user's inbox had nothing in the window), skip the POST and report a friendly "nothing in window" message.

## Phase 4 — Stamp the watermark

```
PUT <backend_url>/api/me/linkedin-dm-sync-state
Authorization: Bearer <jwt>
Content-Type: application/json

{ "last_synced_at": "<now ISO8601>" }
```

This is what tells future runs (and the `surface-followups` pre-flight) that this sync happened.

## Phase 5 — Report honestly

After a successful sync, give a brief summary:

> "Synced your last `<window>` of LinkedIn DMs.
>
> - **`<correspondents>` conversations** in the window.
> - **`<updated>` contacts updated** with fresh `last messaged` / `last contacted` dates — `<N>` of those will surface in your next follow-ups loop because they've gone quiet.
> - **`<unmatched>` correspondents not in your contacts** — they're not in your LinkedIn connection list (yet). Run `/hhq:ingest-contacts` with a fresh `Connections.csv` if you want to fold them in.
> - *(if any)* **`<unchanged>` already up to date.**
>
> Run this any time — say 'sync my linkedin' or '/hhq:sync-linkedin-dms 90d' for a longer window."

If invoked as the `surface-followups` pre-flight, skip the verbose summary — just say:

> "✓ LinkedIn DMs refreshed (`<updated>` updates)."

…and let `surface-followups` continue.

## Things you must NOT do

- Do NOT read message bodies. Inbox card preview snippet (the prefix + first ~80 chars) only — and even that is used only to detect direction, not stored or surfaced.
- Do NOT open a thread except when needed to resolve a missing profile URL — and even then, read the conversation header anchor only.
- Do NOT send messages on the user's behalf. This is read-only — the inbox is opened, not replied to.
- Do NOT exceed the user's specified window. If they said 30 days, stop at 30 days even if scrolling further would surface more conversations.
- Do NOT scrape more than 1000 conversations in a single run, even on a 6-month first sync. Surface honestly and stop with a partial result.
- Do NOT retry past a rate-limit signal. Save partial progress and exit.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` / `jwt_expires_at` after a refresh.
- Do NOT log the JWT, licence key, or auth file contents.
- Do NOT include `last_messaged_at` in the legacy field AND `last_inbound_at` for the same row — pick one. The backend prefers `last_inbound_at` when both are present, but sending both is wasteful and signals confusion.

## Edge cases to handle gracefully

- **Group conversations** (3+ participants) → skip entirely. The "correspondent" concept doesn't apply cleanly. Don't add to digest.
- **Conversations with someone outside the user's connections** (`unmatched` in the response) → already handled by the backend; just surface the count in the summary.
- **InMail / Sponsored messages** → LinkedIn shows these in the inbox with different styling. If detectable from the card (look for "InMail" / "Sponsored" badges), skip — they're not normal correspondents.
- **A correspondent appears but their profile redirects to "user no longer available"** when resolving deferred URLs → skip that conversation. Don't try to fuzzy-match by name.
- **The user has 10,000+ conversations and a 6-month window** → the 1000-conversation cap kicks in. Bail with partial progress; the user can re-run and keep going.
- **Inbox is filtered to a sub-tab** (Focused / Other / Archived) → the skill always runs against the default inbox view. If the user has a specific filter active, the visible cards may not reflect their full activity. Trust LinkedIn's default; document this behaviour in the report only if the user asks.
- **Direction prefix is in another language** ("Tu :", "Tú:", "Du:", etc.) → match heuristically. If the prefix is the correspondent's display name, it's inbound; otherwise treat as outbound. If unclear, leave direction null.
- **Card timestamp says "Yesterday" or "Today" instead of relative** → resolve to today / today minus 1 day.
- **A scroll yields the same cards twice** (LinkedIn occasionally re-renders without adding new) → dedupe by `linkedin_url`. After two consecutive no-progress scrolls, treat as end-of-history.
- **Backend POST fails mid-batch** → since it's a single batch POST, either it all goes or none. On 5xx, retry once after 2s. On second failure, save the digest to a temp file at `<project-dir>/.hhq-linkedin-dm-pending.json` and tell the user to re-run later. Don't update the watermark if the POST didn't land.
- **Watermark PUT fails after a successful import** → not fatal. Tell the user "Synced, but couldn't update the sync timestamp — the next run may re-do some work." Don't roll anything back.
