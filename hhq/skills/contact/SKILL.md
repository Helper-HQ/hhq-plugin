---
name: contact
description: View and manage a single contact's full record — the rolling user-level dossier (a synthesised free-form profile that captures who they are, personal context, decision dynamics, current situation), recent conversation bullets across all the user's campaigns (with dates and source — gmail / linkedin / manual_touch / manual_edit / research), pipeline stage in each campaign they're part of, and recent manual touches. Triggers when the user says "show me <name>", "tell me about <name>", "what do I know about <name>", "edit <name>'s profile", "/hhq:contact <name>", or any free-form sentence asking about a specific contact's record. Supports conversational edits to the dossier ("change her role to VP", "add: prefers WhatsApp", "remove the line about Outlook", "clear her dossier") and to bullets ("delete the bullet from April 15"). Saves dossier edits via PUT /api/me/contacts/{slug}/dossier (auto-versioned in contact_dossier_history). Saves bullet deletions via DELETE /api/me/campaigns/{slug}/contacts/{contactSlug}/conversation-notes/{noteId}. Fuzzy-matches the name input. Run any time the user wants to see or curate what HHQ knows about a person.
---

# Contact — Sales Helper Lite

You are showing the user the full record HHQ has on a single contact and letting them curate it. The dossier is the rolling, synthesised "who is this person right now" profile (user-level, shared across campaigns). Bullets are the dated conversation notes per campaign. Both can be edited; the user has final say.

Stay clean and scannable. The user will use this to prep for calls, audit what HHQ thinks, or correct drift.

## When this skill runs

Trigger on any variant of:

- "show me Sarah Chen"
- "what do I know about Marcus"
- "tell me about Priya"
- "show me Greg's profile"
- "edit Sarah's dossier"
- "what's in Sarah's record"
- "/hhq:contact <name>"

Don't trigger on bulk listing requests ("show me my contacts") — that's a different surface.

## Phase 0 — Auth and campaign context

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

### Step 0c — Resolve current campaign (informational only)

Read `<project-dir>/.hhq-campaign.json` for `campaign_slug` (default `default`). The dossier is user-level so it's the same across all campaigns; this is just used to highlight the active campaign in the per-campaign breakdown.

## Phase 1 — Resolve the contact

Match the user's `<name>` reference against contacts.

`GET <backend_url>/api/me/contacts?per_page=500&page=1` (page through if total > 500). Match strategy in priority order:

1. **Exact full-name match** (case-insensitive on `first_name + ' ' + last_name`) → use that
2. **Exact first-name match** with only one match → use that
3. **First-name + company hint match** ("Sarah at Acme" → match `first_name` AND `company` substring) → use that
4. **Multiple matches on first name** → ask:
   ```
   Multiple matches for "Sarah":
     1. Sarah Chen — VP Sales, Acme
     2. Sarah Patel — Founder, Patel Robotics
   Which one?
   ```
5. **No match** → "I don't have anyone matching `<name>`. Add via `/hhq:ingest-contacts` first."

Cache the resolved `contact_slug`.

## Phase 2 — Fetch the full record

Three calls in parallel:

1. `GET <backend_url>/api/me/contacts/<contact_slug>` → user-level base fields + research + notes
2. `GET <backend_url>/api/me/contacts/<contact_slug>/dossier` → contact_profile + last 20 dossier history rows
3. `GET <backend_url>/api/me/manual-touches?contact_slug=<contact_slug>&limit=20` → recent touches across all campaigns

Then iterate over the contact's campaigns:

4. For each campaign the user has, `GET <backend_url>/api/me/campaigns/<campaign_slug>/contacts/<contact_slug>` to get per-campaign status, conversation_notes (bullets), draft_message, next_followup_due_at.
   - Discover campaigns first via `GET <backend_url>/api/me/campaigns` if you don't already have them cached.
   - 404 means the contact isn't in that campaign — skip it silently.

## Phase 3 — Render the record

```
═══ <Full Name> ═══

<Position> at <Company>
<email if present>  ·  <linkedin_url if present>  ·  <phone if present>
last contact: <last_contacted_at relative>  ·  last reply: <last_messaged_at relative>  ·  <message_count> msgs total

┄ Dossier ┄
<contact_profile rendered as plain markdown — sections, bullets, paragraphs as written>
<if empty: "_No dossier yet — will populate as you have conversations or log touches._">

last updated: <contact_profile_updated_at relative>

┄ In your pipelines ┄
• <Campaign Name> — Stage: <stage name>  ·  status: <campaign status>  ·  reminder: <if next_followup_due_at>  ·  draft: <if has draft>
• <Campaign Name> — Stage: <stage name>  ·  status: <campaign status>
<if only in default campaign and unconfigured: skip this block>

┄ Recent conversation bullets ┄
[<date>] [<source>] [<campaign abbrev>] <bullet text>  (id: <short id>)
[<date>] [<source>] [<campaign abbrev>] <bullet text>  (id: <short id>)
...up to 15 most recent across all campaigns, newest first

┄ Recent manual touches ┄
[<occurred_at>] <type> in <Campaign Name> — "<note or '(no note)'>"
...up to 10 most recent

┄ What now? ┄
  edit dossier   — rewrite the dossier (or describe a specific change)
  add to dossier — append a line you want kept
  drop bullet    — say "drop bullet <id>" or "drop the bullet from <date>"
  clear dossier  — wipe the dossier and start fresh (confirm before)
  log touch      — capture a new interaction (routes to /hhq:log-touch)
  done           — close out
```

For dates: relative if within 90 days ("3d ago", "2w ago", "1mo ago"), absolute otherwise ("2026-01-15"). For bullet ids, show the first 8 chars only — full uuid only used internally.

If the contact has 0 campaign memberships beyond a default no-config campaign, skip the "In your pipelines" block.

If recent_bullets is empty, render: `_No conversation bullets yet — bullets get extracted when you process this contact in /hhq:followups or log a touch._`

## Phase 4 — Handle edits

Conversational. The user describes a change in plain language; you parse, confirm, and apply.

### Edit dossier

Two flavours:

**A) Specific change** ("change her role to VP", "remove the line about Outlook", "add: prefers WhatsApp"):
- Compute the new dossier text yourself by applying the change to the existing one
- Show a before/after diff (just the changed lines)
- Confirm: "Save these changes? (yes / no)"
- On yes: `PUT /api/me/contacts/<contact_slug>/dossier` with `{"contact_profile": "<new>", "trigger_source": "manual_edit"}`

**B) Full rewrite** ("rewrite her dossier", "redo from scratch"):
- Pull all bullets across campaigns + recent touches
- Generate a fresh ~500-word dossier
- Show full preview
- Confirm: "Save? (yes / no / edit further)"
- On yes: PUT with `{"contact_profile": "<new>", "trigger_source": "manual_edit"}`

### Add to dossier

User says "add: prefers WhatsApp for quick questions". Append as a line in the most-relevant existing section (or a new "Notes" section if no obvious fit). Confirm + save with `trigger_source: "manual_edit"`.

### Drop bullet

User says "drop bullet abc12345" or "drop the bullet from April 15".

- Match by id prefix (first 8 chars) OR by date+text fragment
- Identify the bullet's `campaign_slug` (it lives on `campaign_contacts.conversation_notes` for one specific campaign)
- Confirm: "Drop this bullet from <campaign>? `<bullet text>` (yes / no)"
- On yes: `DELETE /api/me/campaigns/<campaign_slug>/contacts/<contact_slug>/conversation-notes/<noteId>` (full uuid)

After the bullet is dropped, ask: "Want to regenerate the dossier without it? Some of what was in there might have come from this bullet. (yes / no)"

If yes, do a regenerate (Phase 4 dossier section, flavour B).

### Clear dossier

User says "clear her dossier" / "wipe it" / "start over".

- Confirm with extra friction: "This wipes Sarah's dossier completely. Bullets stay, but the synthesised profile resets to empty. The wipe is logged in history and can be reviewed but not auto-restored. Confirm? (yes / no)"
- On yes: `PUT .../dossier` with `{"contact_profile": null, "trigger_source": "wipe"}`

### Log touch

If the user says "log a touch" or similar, hand off to the `log-touch` skill — don't try to handle it inline.

### Done / exit

User says "done", "thanks", "close", "that's all" → "Cool. Sarah's record is up to date." Stop.

## Phase 5 — Re-render after edits

After any change, re-fetch and re-render the full record (Phase 2 + Phase 3) so the user sees the saved state. Then offer the same "What now?" menu.

If the user makes 3+ edits in a row, drop the menu and just ask "anything else?" — they're in flow.

## Things you must NOT do

- Do NOT auto-regenerate the dossier from new conversation reads in this skill. That's `surface-followups`'s job. This skill is for explicit user-initiated edits.
- Do NOT delete a contact from this skill. There's no delete endpoint in V1 by design — soft-delete via the contact's pipeline stage (`Not a fit`) is the supported path.
- Do NOT show the JWT, licence key, raw signal weights, or any auth file content.
- Do NOT silently truncate the dossier on save. If the user wants a long dossier, save the long dossier. The ~500-word soft cap is a guideline for AUTO regen, not a hard limit on manual edits.
- Do NOT modify a bullet's text in-place. V1 only supports drop. To "edit" a bullet, drop it and add a new one (in a fresh `surface-followups` pass or via `add: ...` to dossier).
- Do NOT bulk-delete bullets ("drop all bullets from before March"). V1 = one at a time. If the user asks, suggest they handle it via dossier rewrite (which folds away stale bullets implicitly).
- Do NOT modify `<project-dir>/.hhq-session.json` except to update jwt / jwt_expires_at after a refresh.

## Edge cases to handle gracefully

- **Contact has dossier but no bullets and no campaigns** (user manually added them via ingest, hand-typed a dossier) → render gracefully, just skip the empty sections.
- **Contact in 5+ campaigns** → render all but cap "Recent conversation bullets" at 15 across all campaigns. Note: "(showing 15 of <N> total bullets — for the full set, view by campaign)".
- **Dossier history has 20+ versions** → only show that the history exists, not the full content. Offer "show me previous versions" as a follow-up — render up to 5 most recent on demand. Don't dump all of it inline.
- **Bullet id prefix collides** (two bullets share first 8 chars — astronomically unlikely but) → show both, ask which.
- **User refers to a bullet by date but multiple bullets share that date** → show all matching bullets numbered, ask which.
- **Wipe confirmed but PUT fails** → tell the user honestly, don't pretend it worked. They can retry.
- **Contact is on a campaign whose config has been deleted** → the `/api/me/campaigns/{slug}/contacts/{contactSlug}` endpoint will 404 — skip that campaign in the "In your pipelines" render.
