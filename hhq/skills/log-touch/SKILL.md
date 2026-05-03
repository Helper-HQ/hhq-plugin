---
name: log-touch
description: Quick capture for offline interactions sync-gmail can't see — phone calls, meetings, in-person catch-ups, voicemails, SMS. Triggers when the user says "log a call with X", "had a coffee with Sarah", "log a meeting", "talked to X", "/hhq:log-touch", or any free-form sentence describing a recent interaction with a contact. Parses the contact reference (fuzzy-matches name to contacts.slug), the touch type (call/meeting/event/voicemail/sms/other), the note, and an optional follow-up reminder. POSTs to `/api/me/manual-touches` which (1) creates the touch row, (2) appends a bullet to the campaign_contacts.conversation_notes (source=manual_touch) so the dossier sees it, (3) optionally sets `next_followup_due_at` so surface-followups picks it up, and (4) bumps `last_contacted_at` for outreach-counting touches (calls / meetings / voicemails / sms by default). Always operates inside the project's pinned campaign — to log a touch in another campaign, the user opens that project. Run AFTER onboard-helperhq + at least one ingest-contacts.
---

# Log Touch — Sales Helper Lite

You are capturing a quick log of an offline interaction the user just had — a phone call, a meeting, an event chat, a voicemail. These are the touches Gmail sync can't see, but they need to feed the contact's dossier and the follow-ups queue exactly the same way an email reply would.

Stay terse. The user is jotting something down between meetings — don't ceremoniously walk them through forms.

## When this skill runs

Trigger on any variant of:

- "log a call with `<name>`"
- "log a meeting with `<name>`"
- "had a coffee with `<name>`"
- "talked to `<name>` about `<topic>`"
- "left a voicemail for `<name>`"
- "spoke to `<name>` — they want X"
- "ran into `<name>` at `<event>`"
- "/hhq:log-touch <free-form>"

Don't trigger from generic "how did the call go" questions or anything not anchored to a specific contact + interaction.

## Phase 0 — Auth and campaign

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory`. Save as `<project-dir>`. Fallback `~/.hhq/sales-helper/`.

### Step 0b — Resolve auth

Read `<project-dir>/.hhq-session.json`. Same logic as other skills — refresh on near-expiry, halt with `/hhq:connect` prompt if missing.

### Step 0c — Resolve current campaign

Read `<project-dir>/.hhq-campaign.json` for `campaign_slug`. If missing, write `{"campaign_slug": "default"}` and use `default`. The touch lands in this campaign — to log against a different campaign, the user opens that project.

## Phase 1 — Parse the touch

From the user's sentence, extract:

1. **Contact name** — the person who was on the other end. The user usually says a first name ("Sarah"), sometimes first + last ("Sarah Chen"), sometimes first + company ("Sarah at Acme"). Capture as `name_query`.

2. **Type** — pick one based on the verbs / nouns used:
   - "call", "phone", "rang", "spoke on the phone" → `call`
   - "meeting", "met", "coffee", "lunch", "Zoom", "video call" → `meeting`
   - "ran into", "saw at", "bumped into", "event", "conference" → `event`
   - "voicemail", "left a message" → `voicemail`
   - "text", "SMS", "WhatsApp", "messaged" → `sms`
   - default / can't tell → `other`

3. **Note** — the substantive content the user described. Strip out the contact reference and meta phrases ("log a call with Sarah — ") to get just the gist. Keep it short, one sentence usually. If the user said nothing substantive ("just logging it"), `note` is null.

4. **Reminder** — look for follow-up dates: "follow up Tuesday", "remind me in 3 days", "follow up next week", "follow up end of month". Convert to an absolute ISO date based on today (`<currentDate>` from environment). If no reminder phrase, leave null.

5. **Occurred at** — default to now. Override if the user says "yesterday", "this morning", "earlier today", "an hour ago" etc — convert to ISO timestamp.

6. **Counts as outreach** — defaults inferred from type (`call` / `meeting` / `voicemail` / `sms` → true; `event` / `other` → false). Override if the user explicitly says "doesn't count for outreach" or similar.

## Phase 2 — Resolve the contact

Match `name_query` against contacts:

`GET <backend_url>/api/me/contacts?per_page=500&page=1` (page through if total > 500) and search client-side. Match strategy:

1. **Exact full-name match** (case-insensitive on `first_name + ' ' + last_name`) → use that contact
2. **Exact first-name match** with only one match → use that contact
3. **First-name + company hint match** ("Sarah at Acme") — match `first_name` AND `company` substring → use that contact
4. **Multiple matches on first name** → ask the user to disambiguate:
   ```
   Multiple matches for "Sarah":
     1. Sarah Chen — VP Sales, Acme
     2. Sarah Patel — Founder, Patel Robotics
   Which one?
   ```
5. **No match** → "I don't have anyone matching `<name_query>`. Add them via `/hhq:ingest-contacts` first, or paste their full name + company so I can create them quickly."

Cache the resolved `contact_slug`.

## Phase 3 — Confirm before logging

Echo the parsed touch back to the user as a one-liner with a yes/no gate. Don't be ceremonial.

```
Logging: **call** with **Sarah Chen** — "Wants pricing for 50 seats, decision by end of month"
Reminder: follow up Tuesday May 6
Counts as outreach: yes (bumps her 7-day cold-outreach lockout)

Confirm? (yes / edit / cancel)
```

If the user says edit, prompt for what to change (note / type / reminder / contact / counts-as-outreach).

If cancel, drop everything, no API call.

If yes, proceed.

## Phase 4 — POST the touch

`POST <backend_url>/api/me/manual-touches`

Body:

```json
{
  "contact_slug": "<resolved>",
  "campaign_slug": "<campaign_slug>",
  "type": "<type>",
  "note": "<note or null>",
  "occurred_at": "<ISO timestamp>",
  "set_reminder_for": "<ISO date or null>",
  "counts_as_outreach": <true|false>
}
```

Expect HTTP 201 with:

```json
{
  "touch": {...},
  "bullet_appended": {...} | null,
  "next_followup_due_at": "..." | null,
  "last_contacted_at_bumped": true | false
}
```

## Phase 5 — Confirm

Tight one-liner confirming what happened:

```
✓ Logged. Sarah's dossier will pick this up next time you process her, and she'll surface in your follow-ups on Tue May 6.
```

If no reminder was set, drop the second clause:

```
✓ Logged. Sarah's dossier will pick this up next time you process her.
```

If `last_contacted_at_bumped` was true (outreach-counting touch), no need to mention it explicitly — that's expected for calls/meetings.

If `bullet_appended` is null (no note), the dossier line doesn't apply — just confirm:

```
✓ Logged.
```

## Things you must NOT do

- Do NOT prompt the user for fields they didn't volunteer ("what was the duration?", "how did it go?", "any next steps?"). They typed a sentence; respect the brevity.
- Do NOT auto-set a reminder if the user didn't ask for one. Optional.
- Do NOT log a touch for a campaign that isn't the project's pinned campaign. To log into another campaign, the user opens that project. Hard rule.
- Do NOT create the contact if no match — direct the user to `/hhq:ingest-contacts` for the create flow. (V2: inline create with first_name + company.)
- Do NOT log a touch with no contact reference, no type, AND no note. That's nothing.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update jwt / jwt_expires_at after a refresh.
- Do NOT log the JWT, licence key, or auth file contents.

## Edge cases to handle gracefully

- **User says "log a call I just had with Sarah"** but no note — log it with `note: null`, no bullet appended, no dossier impact, but `last_contacted_at` bumps. Useful for "just want it on the record".
- **User says "follow up with Sarah on Tuesday"** without describing a touch → that's not a touch log, that's a reminder set. Re-route: "Want me to log a touch and set a reminder, or just set a reminder? If just a reminder, you can use `/hhq:contact Sarah` and set it from her record."
- **User says "had three calls today: Sarah, Marcus, Priya"** → handle as three separate touches sequentially. Resolve each contact, confirm each, POST each.
- **Reminder date in the past** → reject with: "That reminder date (`<date>`) is in the past. Did you mean `<next occurrence>`?"
- **Type ambiguous** ("caught up with Sarah" — call? meeting?) → default to `meeting` (more common interpretation), but mention it in the confirm: "Logging as **meeting** — say 'change type to call' if it was a phone call."
- **Reminder in past for "follow up next Tuesday"** when today IS Tuesday → resolve to NEXT week's Tuesday, not today.
