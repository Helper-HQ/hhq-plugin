---
name: surface-followups
description: On-demand follow-ups loop — campaign-scoped. Triggers when the user says "who do I need to follow up with", "give me my follow-ups", "follow-ups for today", "who's waiting on me", "let's do follow-ups", or invokes /hhq:followups. Resolves the active campaign from `<project-dir>/.hhq-campaign.json`, auto-runs `sync-gmail` first to refresh inbox state (header-only, ~10-30s), then GETs `/api/me/campaigns/{slug}/followups` for a metadata-ranked queue of up to 10 (manual reminders due, ball-in-your-court, stale-your-court, going cold). Shows the queue with one-line reasoning per entry. User picks one to process — skill live-reads the Gmail thread (bodies in context, never persisted), distils 3-6 dated bullets, regenerates the user-level contact dossier from existing-dossier + new-bullets, drafts a reply in the user's voice referencing the conversation, shows everything for review (keep/edit dossier, keep/edit bullets, keep/edit/discard draft, snooze, mark handled, skip), persists what's confirmed (bullets to `campaign_contacts.conversation_notes`, dossier to `contacts.contact_profile`, draft to `campaign_contacts.draft_message`), then pushes the draft to Gmail as a reply-in-thread via the connector's `create_draft`. User opens Gmail to do a final pass and hit send. Loops back to queue. LinkedIn DMs handled via Chrome connector live-read with copy-paste output (no draft API). Run AFTER onboard-helperhq and at least one sync-gmail.
---

# Surface Follow-ups — Sales Helper Lite

You are working through the user's daily follow-up queue — the people in active conversation who are waiting on a reply, going stale, or have a manual reminder due. Unlike `surface-next-5` (capped daily allowance for cold outreach), follow-ups are reactive: if 8 people are waiting on the user, the user works through 8.

**Privacy invariant — read this first.** When live-reading Gmail or LinkedIn message bodies during the per-pick processing step, those bodies enter your context for drafting and bullet extraction only. **Never persist message bodies.** Bullets and dossier text are derived data and ARE persisted; raw bodies are dropped at the end of each per-pick step. If the user asks to see "the original email" or "what they actually said", refuse and direct them to Gmail / LinkedIn.

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

If `jwt_expires_at` is past or within 60s of expiry:
1. POST `<backend_url>/api/refresh` with `Authorization: Bearer <old jwt>`. On 200, save the new token + expires_at.
2. On 401, tell the user: "Your session was released — say `/hhq:connect` to re-link this project." Stop.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

### Step 0c — Resolve current campaign

Read `<project-dir>/.hhq-campaign.json` for `campaign_slug`. If missing, write `{"campaign_slug": "default"}` and use `default`.

## Phase 0.5 — Connector check

Look at the tools available in your current session. If you can see ANY tools whose names map to Gmail operations — `search_threads`, `get_thread`, `create_draft` (typically prefixed with `mcp__<some-uuid>__`) — proceed. The Gmail connector is required for follow-ups (without it, you can't read threads or push drafts).

If no Gmail tools are loaded, halt:

> "I can't see the Gmail connector in this session. Install it in your Cowork/Claude.ai settings (Settings → Connectors → Gmail), start a new chat, and try again."

The Chrome connector for LinkedIn is optional — only needed if any picked follow-up turns out to be a LinkedIn-DM thread. If it's missing when needed, fall back to copy-paste output for that pick.

## Phase 1 — Auto-refresh inbox

Run the `sync-gmail` skill inline. Stays header-only. Takes ~10-30s. The point is the queue you're about to compute reflects today's reality, not whatever last sync caught.

If sync-gmail returns an error or partial result, surface it briefly and ask whether to continue with stale state:

> "Gmail sync hit a snag (`<error>`). Continue with the queue as it stands? Some recent replies might be missing."

If the user says no, stop. If yes, continue.

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
      "linkedin_url": "https://...",
      "pipeline_stage": {"id": 5, "slug": "in_conversation", "name": "In conversation"},
      "campaign_status": "drafted",
      "last_messaged_at": "2026-05-01T...",
      "last_contacted_at": "2026-04-28T...",
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

Use the contact's `email` and `linkedin_url`:

- If `email` is set, the conversation is most likely Gmail — proceed with Gmail thread search.
- If `email` is empty but `linkedin_url` is set, this is a LinkedIn-DM-only conversation — skip to LinkedIn branch (Step 4b-LI).
- If both are set, default to Gmail (the more common case for active conversations).

### Step 4b-Gmail — Live-read the Gmail thread

Use `search_threads` with a query like `from:<contact.email> OR to:<contact.email> newer_than:90d`. Cap at the most recent 1-3 threads — usually one thread is the active conversation.

If multiple threads come back, pick the one with the most recent message date.

For the chosen thread, `get_thread` to fetch ALL messages. **You MUST capture the `thread_id` for the draft push in Step 4g.** Read full message bodies into your context for this step only.

If `search_threads` returns nothing, the contact's email may not match the Gmail account they actually correspond from. Tell the user:

> "Couldn't find an active Gmail thread with `<email>`. Either the conversation is on LinkedIn, or they email from a different address. Want to skip and move on, or paste their other email?"

### Step 4b-LI — Live-read LinkedIn DMs

Check if a Chrome connector is loaded (look for tools like `navigate`, `read_page`). If not:

> "I'd need the Chrome connector to read your LinkedIn messages with `<contact name>`. Without it I can draft from the conversation bullets you already have on file, but I can't see what's been said since the last time we updated. Continue with bullets only? (yes / no)"

If Chrome IS loaded, navigate to the LinkedIn message thread for this contact and read the visible messages. Note: no `thread_id` to capture — LinkedIn drafts will be copy-paste output in Step 4g.

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

`source` is `gmail` for Gmail reads, `linkedin` for LinkedIn reads.

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

4. **Push to Gmail draft** (Gmail conversations only): use the connector's `create_draft` with the `thread_id` captured in Step 4b-Gmail so it threads correctly as a reply. Capture the returned draft id for confirmation.

5. **For LinkedIn conversations**: skip Step 4 — there's no LinkedIn draft API. Instead, output the draft as copy-paste content with a brief instruction: "Copy this into the LinkedIn thread with `<contact name>` and send when ready."

6. **Update campaign_contact status**: `PUT <backend_url>/api/me/campaigns/<campaign_slug>/contacts/<contact_slug>` with `{"status": "drafted"}` if current status is anything pre-drafted.

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
- Do NOT send messages on the user's behalf. The whole point of the Gmail-draft handoff is that Gmail's UI is the last-mile review surface. Pushing a draft is fine; sending is not.
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
