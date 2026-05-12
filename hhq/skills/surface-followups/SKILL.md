---
name: surface-followups
description: On-demand follow-ups loop — campaign-scoped. Triggers when the user says "who do I need to follow up with", "give me my follow-ups", "follow-ups for today", "who's waiting on me", "let's do follow-ups", or invokes /hhq:followups. Resolves the active campaign from `<project-dir>/.hhq-campaign.json`, auto-runs `sync-gmail`, `sync-linkedin-dms`, and `sync-whatsapp` pre-flights to refresh inbox state across all three channels (each is header-only, watermark-gated against re-running too often), then GETs `/api/me/campaigns/{slug}/followups` for a metadata-ranked queue of up to 10 (manual reminders due, ball-in-your-court, stale-your-court, going cold). Shows the queue with one-line reasoning per entry. User picks one to process — skill routes to the right channel based on the queue payload's `last_inbound_via` / `last_outbound_via` (gmail / linkedin / whatsapp), live-reads the thread (bodies in context, never persisted), distils 3-6 dated bullets, regenerates the user-level contact dossier, drafts a reply in the user's voice, shows everything for review, persists what's confirmed. Gmail gets a draft pushed to the user's Gmail; LinkedIn DMs and WhatsApp threads get copy-paste output (no draft API for either). Run AFTER onboard-helperhq and at least one sync-gmail.
---

# Surface Follow-ups — Sales Helper Lite

You are working through the user's daily follow-up queue — the people in active conversation who are waiting on a reply, going stale, or have a manual reminder due. Unlike `surface-next-5` (capped daily allowance for cold outreach), follow-ups are reactive: if 8 people are waiting on the user, the user works through 8.

**Privacy invariant — read this first.** When live-reading Gmail, LinkedIn, or WhatsApp message bodies during the per-pick processing step, those bodies enter your context for drafting and bullet extraction only. **Never persist message bodies.** Bullets and dossier text are derived data and ARE persisted; raw bodies are dropped at the end of each per-pick step. If the user asks to see "the original message" or "what they actually said", refuse and direct them to the source app.

## When this skill runs

Trigger when the user says any variant of:

- "who do I need to follow up with"
- "give me my follow-ups"
- "follow-ups for today"
- "who's waiting on me"
- "let's do follow-ups"
- "show me my queue"
- "let's clear my replies"

Do NOT trigger if the user is asking about a specific person (use `/hhq:contact` for that), asking for new prospects (use `surface-next-5`), or working with cold outreach.

## Phase 0 — Auth and campaign

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory` (no arguments). Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` if not registered.

### Step 0b — Resolve auth (per-project session)

Read `<project-dir>/.hhq-session.json`.

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming. Continue.
- **Neither found** → "This project isn't connected to Helper HQ — say `/hhq:connect` to link it (or `/hhq:onboard` if you're brand-new)." Stop.

If `jwt_expires_at` is past or within 60s of expiry, proactively refresh: POST `<backend_url>/api/refresh` with `Authorization: Bearer <old jwt>` (the endpoint accepts expired tokens). Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json` (preserving other fields).

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key` from `.hhq-session.json`** (NOT a fresh UUID — reusing the same UUID keeps this idempotent and avoids burning a slot). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop (`session_limit_reached` includes the `/sessions` URL).

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

### Step 0c — Resolve current campaign

Read `<project-dir>/.hhq-campaign.json` for `campaign_slug`. If missing, write `{"campaign_slug": "default"}` and use `default`.

## Phase 0.5 — Pick the Gmail backend (Option Y soft fallback)

There are two ways this skill can talk to Gmail for the per-pick read + draft push:

1. **HHQ Gmail MCP (preferred)** — uses the user's HHQ-owned OAuth connection (extended Gmail access). Per-pick: `POST /api/mcp/gmail/list_threads` to find the conversation, `POST /api/mcp/gmail/get_thread` for full bodies, `POST /api/mcp/gmail/push_draft` for the draft. Faster, cleaner, scope-enforced privacy.
2. **Cowork generic Gmail connector (fallback)** — uses Claude's built-in `search_threads`, `get_thread`, `create_draft` tools.

Detection — call:

```
GET <backend_url>/api/me/gmail/connection
Authorization: Bearer <jwt>
```

Branch on the response:

- **`{"connected": true, ...}`** → set `gmail_backend = "hhq"`. The HHQ Gmail MCP is available — use it for the per-pick steps in Phase 4.
- **`{"connected": false}`** → set `gmail_backend = "cowork"`. Now check tool availability: look at the tools in your current session for any `search_threads` / `get_thread` / `create_draft` (typically prefixed with `mcp__<some-uuid>__`). If those exist, continue. If not, halt with:
  > "I can't see the Gmail connector in this session, and you don't have extended Gmail access set up either. Two options:
  >
  > - Install the Gmail connector in your Cowork/Claude.ai settings, start a new chat, and try again.
  > - Or run `/hhq:onboard` Phase 7.5 to opt into extended Gmail access — once approved, follow-ups runs through HHQ's own Gmail backend."
- **Network error on the connection check** → assume `gmail_backend = "cowork"` and continue with the existing Cowork detection. Don't block follow-ups just because the backend is briefly unreachable.

The Chrome connector for LinkedIn is optional — only needed if any picked follow-up turns out to be a LinkedIn-DM thread. If it's missing when needed, fall back to copy-paste output for that pick. (LinkedIn handling doesn't change between HHQ and Cowork paths — neither covers LinkedIn.)

## Phase 1 — Auto-refresh inbox

### Step 1a — Sync Gmail

Run the `sync-gmail` skill inline. Stays header-only. Takes ~10-30s. The point is the queue you're about to compute reflects today's reality, not whatever last sync caught.

If sync-gmail returns an error or partial result, surface it briefly and ask whether to continue with stale state:

> "Gmail sync hit a snag (`<error>`). Continue with the queue as it stands? Some recent replies might be missing."

If the user says no, stop. If yes, continue.

### Step 1b — Sync LinkedIn DMs (if Chrome is loaded and watermark is stale)

LinkedIn-only conversations don't surface accurately in the queue unless someone has recently walked the messaging inbox. To keep the queue honest without scraping LinkedIn on every follow-ups loop:

1. **Check Chrome connector availability.** Look for `mcp__Claude_in_Chrome__navigate` / `read_page` in this session's tools. If not loaded, skip this step entirely (no warning — the user may simply prefer not to use LinkedIn from here).

2. **Check the watermark.** GET `<backend_url>/api/me/linkedin-dm-sync-state`.

3. **Decide:**
   - `last_synced_at` is null → first run; the user hasn't walked LinkedIn yet. Ask: *"Want me to also walk your LinkedIn DMs first? First-run goes back 6 months, takes 10-15 min. (yes / skip)"* — if they say skip, do nothing for now and move on.
   - `last_synced_at` is older than 12 hours → stale; trigger `sync-linkedin-dms` inline with the default 30-day window. Don't ask — silent pre-flight, the watermark gate already keeps this from running every loop.
   - `last_synced_at` is within the last 12 hours → fresh enough; skip silently.

4. If `sync-linkedin-dms` errors or hits a rate-limit, surface briefly: *"LinkedIn sync hit a snag — continuing with the queue as it stands. LinkedIn-only follow-ups may be slightly stale."* Continue.

The point of the 12-hour gate is per-day rate-limit safety — running `sync-linkedin-dms` every time the user opens the follow-ups loop would hammer LinkedIn enough to risk a soft block.

### Step 1c — Sync WhatsApp (if Chrome is loaded and watermark is stale)

WhatsApp-only conversations follow the same pattern as LinkedIn — without a recent walk, the queue won't reflect today's activity for WhatsApp-active contacts. Use the same Chrome-availability + 12-hour watermark gate:

1. **Check Chrome connector availability.** If `mcp__Claude_in_Chrome__navigate` / `read_page` aren't loaded, skip this step entirely. (No warning — Chrome may already be busy with LinkedIn or simply not installed.)

2. **Check the watermark.** GET `<backend_url>/api/me/whatsapp-dm-sync-state`.

3. **Decide:**
   - `last_synced_at` is null → first run. Ask: *"Want me to also walk your WhatsApp Web inbox first? First-run goes back 6 months, takes 15-20 min — WhatsApp is touchier than LinkedIn about automation, so I pace it slower. (yes / skip)"* — if they say skip, do nothing for now and move on.
   - `last_synced_at` is older than 12 hours → stale; trigger `sync-whatsapp` inline with the default 30-day window. Silent pre-flight.
   - `last_synced_at` is within the last 12 hours → fresh enough; skip silently.

4. If `sync-whatsapp` errors or hits a soft block, surface briefly: *"WhatsApp sync hit a snag — continuing with the queue as it stands. WhatsApp-only follow-ups may be slightly stale."* Continue.

The 12-hour gate matters more here than for LinkedIn — WhatsApp Web flags automation aggressively. Don't run it more than necessary.

If both LinkedIn and WhatsApp pre-flights would fire on the same loop and they share the Chrome connector, run them sequentially with a 5-second pause between, not in parallel.

## Phase 2 — Fetch the queue

`GET <backend_url>/api/me/campaigns/<campaign_slug>/followups?limit=10&offset=0`

Returns:

```json
{
  "queue": [
    {
      "contact_id": 123,
      "contact_slug": "sarah-chen",
      "first_name": "Sarah",
      "last_name": "Chen",
      "headline": "VP Sales @ Acme",
      "company": "Acme",
      "position": "VP of Sales",
      "email": "sarah@acme.com",
      "primary_phone": "+61400123456",
      "linkedin_url": "https://...",
      "pipeline_stage": {"id": 5, "slug": "in_conversation", "name": "In conversation"},
      "campaign_status": "drafted",
      "last_messaged_at": "2026-05-01T...",
      "last_contacted_at": "2026-04-28T...",
      "last_inbound_via": "whatsapp",
      "last_outbound_via": "gmail",
      "message_count": 12,
      "next_followup_due_at": null,
      "has_draft": false,
      "draft_message_updated_at": null,
      "recent_bullets": [
        {"id": "uuid", "text": "...", "dated_at": "...", "source": "gmail", "created_at": "..."}
      ],
      "tier": 2,
      "reason": "They replied — ball in your court"
    }
  ],
  "limit": 10,
  "offset": 0,
  "total": 14
}
```

If `total == 0`:

> "Inbox zero — nobody's waiting on you in `<campaign_slug>` right now. Nice. If you're expecting someone, run `sync my gmail` again, or check in `/hhq:contact <name>` directly."

Stop. (No queue to process.)

If `total > 0`, continue to Phase 3.

## Phase 3 — Show the queue

Render as a numbered list. Use the `reason` field verbatim — backend computes it consistently.

```
You have <total> follow-ups today in <campaign_slug>:

1. **Sarah Chen** — VP of Sales, Acme
   They replied 2d ago — ball in your court
   ⏳ draft saved 4h ago

2. **Marcus Webb** — Founder, Webb Robotics
   Manual reminder due (set 3d ago)

3. **Priya Nathan** — Director, Nathan Partners
   You replied 6d ago, no response yet

... (up to 10)

Pick one to work on (1-10), or say "show more" for the next 10.
```

For each entry:
- Use `tier` to pick the icon prefix (no icon if it'd add noise — only ⏳ for has_draft, no other emoji)
- Append `⏳ draft saved <Nh|Nd> ago` under any entry where `has_draft == true`. The user may want to pick this one to finish a draft they started earlier.
- If `recent_bullets` is non-empty, do NOT show them in the queue list — too much noise. They surface during processing in Phase 4.

If `total > 10`, mention `(<total - 10> more after these — say "show more" to load)`.

User can:
- **Pick a number** → Phase 4 with that contact
- **Say "show more"** → re-fetch with `offset=10` (or current offset + limit), append to displayed list
- **Snooze N from the queue without processing** → e.g. "snooze Marcus 3 days" → POST snooze for that contact, drop from queue, re-show
- **Skip without action** → "skip Sarah for now" → just remove from this session's display, no API call (will re-surface tomorrow)

## Phase 4 — Process one pick

For the picked contact:

### Step 4a — Identify the conversation source

The queue payload now carries `last_inbound_via` and `last_outbound_via` — set by whichever sync source last moved the timestamps (`gmail`, `linkedin`, `whatsapp`, `manual_touch`, or null for legacy rows). Use the via field that matches the pick's tier:

- **`tier == 1` (manual reminder due)** — no specific thread; the reminder might be about anything. Prefer whichever via is more recent (use `last_inbound_via` if `last_messaged_at >= last_contacted_at`, else `last_outbound_via`). Ask the user if it's unclear: *"Heads-up — this is a manual reminder, not a specific thread. Want me to open the most recent channel (`<via>`) or skip the read and just draft from bullets?"*
- **`tier == 2` (ball in your court)** — they messaged you → use `last_inbound_via` (the channel they messaged from).
- **`tier == 3` (stale, your court)** — you messaged them → use `last_outbound_via` (the channel you used).
- **`tier == 4` (going cold)** — prefer `last_inbound_via`, fall back to `last_outbound_via`.

Once the via channel is chosen, branch:

- `via == "gmail"` (or null + `email` present) → Step 4b-Gmail.
- `via == "linkedin"` (or null + only `linkedin_url`) → Step 4b-LI.
- `via == "whatsapp"` → Step 4b-WA.
- `via == "manual_touch"` → no thread to read; jump straight to bullets-only drafting in Step 4e. The bullets you'll see in `recent_bullets` already capture what was discussed.
- Nothing usable → fall back to the legacy heuristic (`email` → Gmail; `linkedin_url` only → LinkedIn; phones only → WhatsApp; nothing → bullets-only).

**Important:** the via fields are stamped only on activity that landed *after* v0.27. Older contacts have null via and the legacy heuristic (email → Gmail) applies. That's fine — most active follow-ups will have synced at least once since v0.27.

### Step 4b-Gmail — Live-read the Gmail thread

Branch on `gmail_backend` from Phase 0.5.

#### HHQ path (`gmail_backend = "hhq"`)

```
POST <backend_url>/api/mcp/gmail/list_threads
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "query": "from:<contact.email> OR to:<contact.email> newer_than:90d",
  "max_results": 3
}
```

Pick the thread with the most recent `latest_date`. Capture its `thread_id`.

Then fetch full bodies:

```
POST <backend_url>/api/mcp/gmail/get_thread

{ "thread_id": "<id>" }
```

Returns the same Gmail thread shape (messages array with bodies in `payload.body.data`, base64url-encoded — decode + walk multipart parts as needed). Read into your context for this per-pick step only.

#### Cowork path (`gmail_backend = "cowork"`)

Use `search_threads` with a query like `from:<contact.email> OR to:<contact.email> newer_than:90d`. Cap at the most recent 1-3 threads.

If multiple threads come back, pick the one with the most recent message date. For the chosen thread, `get_thread` to fetch ALL messages.

#### Both paths

**You MUST capture the `thread_id` for the draft push in Step 4g.** Read full message bodies into your context for this step only.

If the search returns nothing (no Gmail thread with this email), the contact's email may not match the Gmail account they actually correspond from. Tell the user:

> "Couldn't find an active Gmail thread with `<email>`. Either the conversation is on LinkedIn, or they email from a different address. Want to skip and move on, or paste their other email?"

### Step 4b-LI — Live-read LinkedIn DMs

Check if a Chrome connector is loaded (look for tools like `navigate`, `read_page`). If not:

> "I'd need the Chrome connector to read your LinkedIn messages with `<contact name>`. Without it I can draft from the conversation bullets you already have on file, but I can't see what's been said since the last time we updated. Continue with bullets only? (yes / no)"

If Chrome IS loaded, navigate to the LinkedIn message thread for this contact and read the visible messages. Note: no `thread_id` to capture — LinkedIn drafts will be copy-paste output in Step 4g.

### Step 4b-WA — Live-read WhatsApp Web

Check if a Chrome connector is loaded. If not:

> "I'd need the Chrome connector to read your WhatsApp thread with `<contact name>`. Without it I can draft from the conversation bullets you already have on file, but I can't see what's been said since the last time we updated. Continue with bullets only? (yes / no)"

If Chrome IS loaded:

1. **Navigate to** `https://web.whatsapp.com/`. Verify the chat list is visible (if you see the QR code, halt and tell the user to scan it in — same gate as `sync-whatsapp` Phase 0d).

2. **Find the conversation.** Two paths depending on what the queue payload carried:
   - **Has `primary_phone`** → use WhatsApp Web's search box. Click the search input at the top of the chat list, type the phone in international format (e.g. `+61 400 123 456`). The matching chat surfaces in the result list — click it. If no match, fall back to name search.
   - **Has only `first_name` / `last_name`** (no phone, or phone search returned nothing) → search by full name (`first_name + " " + last_name`). If the user has the contact saved in their phone book, WhatsApp Web finds them by name.
   - **Neither resolves** → tell the user:
     > "Couldn't find a WhatsApp thread with `<contact name>`. Either they're not in your phone's address book, or the number/name doesn't line up. Want to skip and move on, or fall back to drafting from bullets only?"

3. **Read the visible thread.** Use `mcp__Claude_in_Chrome__read_page` to capture the conversation pane. Read the most recent ~20 messages (whatever's visible without scrolling — WhatsApp loads progressively from the bottom). Each message shows: sender (you vs them, detected by message-bubble alignment + colour), timestamp (next to each bubble), body text. For media messages, treat as opaque ("[photo]", "[voice note]", "[video]") — don't try to extract content. **Read into your context for this per-pick step only — never persist body text.**

4. **No `thread_id` to capture** — WhatsApp drafts are copy-paste output in Step 4g, same as LinkedIn.

5. **Pace:** 2 seconds between any clicks. Don't scroll back through the thread history for V1 — the visible pane is enough for "what's the most recent activity" context.

If WhatsApp Web shows a "checking for new messages" / sync overlay that doesn't clear within 30 seconds, halt that pick:

> "WhatsApp Web is still syncing this thread on its side. Skip for now? I can come back to this one in a minute."

### Step 4c — Distil bullets

From the live-read messages (and any new since the last bullet's `created_at`), produce 3-6 dated bullets. One bullet per concrete fact:

- What they want / asked about
- What they're blocking on
- What you (the user) promised, deadlines, next steps
- Personal context that came up (going on holiday, kid's birthday, just got promoted)
- Decision dynamics (mentioned procurement, mentioned a co-founder)

Bullet structure:

```json
{
  "text": "Looking at competitive offers from X and Y - says decision by end of month",
  "dated_at": "<ISO date the bullet's fact was first mentioned in the thread>",
  "source": "gmail"
}
```

`source` is `gmail` for Gmail reads, `linkedin` for LinkedIn reads, `whatsapp` for WhatsApp reads.

Skip facts already covered in `recent_bullets` (avoid duplicates). If nothing new is in the thread since the last bullet date, you may produce zero bullets — that's fine.

### Step 4d — Regenerate the dossier

GET `<backend_url>/api/me/contacts/<contact_slug>/dossier` to fetch the existing `contact_profile` and history.

If `contact_profile` is empty AND the contact has `research` populated (from `research-and-draft`), seed the dossier from research first. Otherwise start blank.

Now produce a NEW dossier text by feeding the model:
- Existing dossier
- New bullets from Step 4c
- The full set of `recent_bullets` for context (drift detection)

Prompt yourself with: "Update this rolling contact dossier. Keep ~500 word soft cap. Drop stale facts (older than 6 months unless still load-bearing). Free-form markdown. Sections optional but useful: Who they are now / Personal context / Relationship / Decision dynamics / Current situation."

The dossier is about the *person*, not the conversation. The conversation lives in bullets.

### Step 4e — Draft the reply

Fetch the user's voice profile if not already cached: `GET <backend_url>/api/me/config` → use `voice_profile`. Also fetch campaign voice additions: `GET <backend_url>/api/me/campaigns/<campaign_slug>/config` → merge `voice_additions` over `voice_profile`.

Draft the reply using:
- The full thread context (from live read)
- The new dossier (from Step 4d)
- The voice profile + additions
- Greg-style conventions: short, signal-referenced, conversational, no buzzwords, no closing salutations beyond what the user normally writes

Length: match the rhythm of the thread. If they're sending one-line emails, your reply is one line. If they sent a paragraph, you reply with a paragraph.

### Step 4f — Show the review screen

Present everything in one screen:

```
**Sarah Chen** — VP of Sales, Acme
last reply: 2d ago

┄ New conversation bullets ┄
+ Looking at competitive offers from X and Y — decision by end of month
+ Asked for case study from a similar-stage company
+ Mentioned co-founder is the final approver

┄ Updated dossier ┄
<show diff: + new lines, - removed lines, unchanged in plain>

┄ Draft reply ┄
<the draft, formatted for readability>

What now?
  approve     — save bullets, save dossier, save draft, push to Gmail
  edit draft  — let me revise
  edit bullet — drop or change a specific bullet (say "drop bullet 2" or "change bullet 1 to: ...")
  edit dossier — rewrite the dossier
  snooze N    — push to N days from now, don't draft (e.g. "snooze 3")
  handled     — they're already replied to outside HHQ, just clear from queue (optional note)
  skip        — leave for next session, no changes
  back        — back to queue without saving anything
```

Honour edits conversationally. After each edit, re-show the affected section and ask "looks good now?" before moving on.

### Step 4g — Persist on approve

When the user says "approve" (or after all edits resolved):

1. **Append bullets**: `POST <backend_url>/api/me/campaigns/<campaign_slug>/contacts/<contact_slug>/conversation-notes` with body `{"bullets": [...]}`. If zero new bullets, skip this call.

2. **Save dossier**: `PUT <backend_url>/api/me/contacts/<contact_slug>/dossier` with body `{"contact_profile": "<new dossier>", "trigger_source": "auto_regen"}`. If dossier didn't change (rare — usually adding bullets shifts something), still PUT to record the regen attempt.

3. **Save HHQ draft**: `PUT <backend_url>/api/me/campaigns/<campaign_slug>/contacts/<contact_slug>/draft-message` with body `{"draft_message": "<the draft>"}`. This is the resume-after-walk-away copy.

4. **Push to Gmail draft** (Gmail conversations only) — branch on `gmail_backend` from Phase 0.5:

   - **HHQ path:** `POST <backend_url>/api/mcp/gmail/push_draft` with `{"thread_id": "<id>", "body": "<the draft>"}`. The backend constructs the RFC 2822 envelope (To/Subject/In-Reply-To headers) automatically from the thread's latest inbound message.
   - **Cowork path:** use the connector's `create_draft` with the `thread_id` captured in Step 4b-Gmail so it threads correctly as a reply.

   Either way: capture the returned draft id for confirmation.

5. **For LinkedIn conversations**: skip Step 4 — there's no LinkedIn draft API. Instead, output the draft as copy-paste content with a brief instruction: "Copy this into the LinkedIn thread with `<contact name>` and send when ready."

6. **For WhatsApp conversations**: skip Step 4 — WhatsApp Web has no draft API either, and even if it did, we wouldn't want to autofill the compose box (WhatsApp Web is aggressive about flagging automated behaviour). Instead, output the draft as copy-paste content with a brief instruction: "Copy this into the WhatsApp thread with `<contact name>` and send when ready. The thread is already open in your browser — just paste and review."

7. **Update campaign_contact status**: `PUT <backend_url>/api/me/campaigns/<campaign_slug>/contacts/<contact_slug>` with `{"status": "drafted"}` if current status is anything pre-drafted.

Then confirm to the user:

```
✓ Saved bullets, refreshed Sarah's dossier, drafted reply.
✓ Pushed draft to your Gmail — open the thread, do a final pass, hit send.
   (Or if you change your mind, your HHQ draft is saved and editable next time.)
```

### Step 4h — Snooze / handled / skip handling

- **snooze N**: `POST .../snooze` with `{"days": N}`. Confirm: "Snoozed Sarah for N days." Drop from queue, return to Phase 3 with next available pick.
- **handled** (optionally with note): `POST .../mark-handled` with `{"note": "<optional>", "logged_outreach": true|false}`. Confirm: "Marked Sarah handled." Drop from queue, return to Phase 3.
  - If user says "I called her" or similar in the note, set `logged_outreach: true` to bump `last_contacted_at`.
  - Otherwise leave `logged_outreach: false` (mostly for "they replied via WhatsApp" or "they replied and I already handled it elsewhere").
- **skip**: no API call, just drop from this session's display.
- **back**: no API call, drop nothing, re-show queue.

### Step 4i — Next pick

After processing, return to Phase 3. Show the updated queue (refresh from cache or re-render) with the processed item removed. If queue is empty, congratulate the user and stop:

> "That's the queue — nice work. Your inbox is caught up in `<campaign_slug>`."

If the user wants to keep going, say "next" or pick another number.

## Things you must NOT do

- Do NOT persist message body content. Bodies are read once into context for the per-pick step and discarded. Bullets and dossier text are the only derived artefacts saved.
- Do NOT scan or summarise the user's full inbox. Only read threads for contacts the user has explicitly picked from the follow-ups queue.
- Do NOT push to Gmail without `thread_id` set on `create_draft` — that creates a NEW thread, breaks conversational context, and looks unprofessional.
- Do NOT send messages on the user's behalf on ANY channel — Gmail, LinkedIn, or WhatsApp. Pushing a draft to Gmail is fine; for LinkedIn and WhatsApp it's copy-paste only.
- Do NOT type the draft text into WhatsApp Web's compose box programmatically, even if Chrome would let you. WhatsApp Web flags non-organic input patterns; output as copy-paste only.
- Do NOT skip the `sync-gmail` auto-run in Phase 1 unless the user explicitly says "skip the sync, just show me what we have". Even then, warn that the queue may be stale.
- Do NOT show the raw `tier` number to the user — translate it via the `reason` field which already explains the tier in plain language.
- Do NOT cap the queue at 5. Follow-ups don't have a quality budget.
- Do NOT regenerate the dossier from scratch every time — pass the existing dossier into the regen prompt so continuity is preserved.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` / `jwt_expires_at` after a refresh.
- Do NOT log the JWT, licence key, or auth file contents.
- Do NOT process more than one pick at a time (no batch mode in V1). The user works through the queue sequentially.

## Edge cases to handle gracefully

- **No matching Gmail thread for the picked contact** → tell the user, offer to skip or to fall back to bullets-only drafting.
- **Gmail thread is mostly auto-forwards / out-of-office** → distil what's actually meaningful; if there's no human content, surface this and ask whether to skip or snooze.
- **Picked contact has `has_draft: true`** → fetch the existing draft (it's in the contact's `campaign_contacts.draft_message`, but not in the queue payload — fetch via `GET /api/me/campaigns/{slug}/contacts/{contactSlug}` to read it). Show as the starting draft for the user to keep, edit, or discard. Don't mindlessly regenerate from scratch.
- **Backend down between bullet POST and dossier PUT** → save bullets first; if dossier PUT fails, tell the user honestly that bullets are saved but dossier didn't update. They can re-run later or fix in `/hhq:contact`.
- **Gmail `create_draft` push fails** → the HHQ draft IS saved (Step 4g.3 happens first). Tell the user and offer copy-paste fallback.
- **User says "approve" but Gmail connector lost mid-session** → save HHQ draft, tell user to copy-paste manually this time.
- **Contact has no `last_messaged_at` and no manual reminder, but appeared in the queue** → shouldn't happen with the backend's eligibility filter, but if it does, treat as "going cold" tier and proceed.
- **Empty contact_profile AND empty research** → seed a minimal placeholder from headline + position + company on first regen, expand from bullets.
- **Multiple Gmail threads with same contact** → pick the one with the most recent message activity. Don't merge across threads — they may be unrelated topics.
- **`last_inbound_via` and `last_outbound_via` are both null** (legacy contact, never synced under v0.27+) → fall back to the heuristic (`email` present → Gmail; LinkedIn URL only → LinkedIn; phones only → WhatsApp). Once the next sync runs, the via fields get stamped and routing becomes deterministic.
- **`last_inbound_via` and `last_outbound_via` disagree** (e.g. they messaged via WhatsApp, you replied via email) → use the via that matches the tier, per Step 4a. If the tier doesn't clearly map to one direction (e.g. "going cold"), prefer the more recent timestamp.
- **WhatsApp Web search yields multiple chats with the same name** (rare — two contacts both saved as "Greg") → list the matches with the visible phone-number subtext from each result, ask the user to disambiguate. Don't guess.
- **WhatsApp Web is logged out mid-loop** (QR-code screen suddenly appears — happens on long sessions) → halt that pick gracefully, tell the user to re-scan, offer to continue once they confirm.
- **Chrome is busy with another tab** (e.g. user is on LinkedIn manually) → WhatsApp Web opens in a new tab; the navigate tool may not surface the existing tab. Use the navigate tool's default behaviour; if it fails, surface honestly and skip the pick.
