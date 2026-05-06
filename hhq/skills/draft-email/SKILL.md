---
name: draft-email
description: Admin Helper — drafts a fresh outbound email (with subject line) to a contact and pushes it to Gmail as a new draft. Built for people who didn't come through the LinkedIn opener flow — follow-ups to non-responders, first-touch emails to people met in person, referrals, conference contacts. The user identifies the recipient (by name or email), gives a one-line purpose ("follow up after our coffee"), and the skill drafts in their voice referencing what's already known about the contact, then pushes the draft to Gmail. User reviews + sends in Gmail. Triggers on "email <name>", "draft an email to <name>", "write a follow-up to <name>", "send something to <name> about <topic>", "/hhq:draft-email <name>". Requires extended Gmail access (HHQ OAuth).
---

# Draft Email — Admin Helper

You are composing a NEW outbound email (not a reply) to one person the user knows. The recipient hasn't come through the LinkedIn opener flow — they're someone the user met in person, was referred to, exchanged business cards with, had a phone call with, or messaged on LinkedIn without a response. Identify the recipient, gather what's known about them and what the user wants out of the email, draft in their voice, push to Gmail as a draft, tell the user to review + send.

This is a **single-recipient, single-draft** skill. For replying to an existing thread, the user wants `draft-reply`. For batch follow-ups across pipeline contacts, they want `surface-followups` (Sales Helper) or `manage-followups` (Admin Helper). This skill is "I want to send a fresh email to one specific person right now."

## When this skill runs

Trigger on:

- "email `<name>`"
- "draft an email to `<name>`"
- "write a follow-up to `<name>`"
- "send something to `<name>` about `<topic>`"
- "follow up with `<name>` from `<event>`"
- "compose a note to `<name>`"
- "/hhq:draft-email `<name>`"

Don't trigger on:
- "reply to `<thread>`" — that's `draft-reply`.
- "send X to Y" — Helper HQ doesn't send. Push as draft, user sends from Gmail.
- Generic "what should I email someone" — coaching, not a draft request.

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
  > "Drafting outbound emails uses Helper HQ's direct Gmail integration (extended Gmail access — beta). You're not connected yet. Run `/hhq:onboard` to opt in, or `/hhq:connect-gmail` if you already have approval."

Cowork's generic Gmail connector doesn't expose the new-draft endpoint we need. Extended Gmail access is the only path.

## Phase 1 — Identify the recipient

Three ways the user might refer to someone, in priority order:

### 1. Direct email address

If the user pasted something that looks like an email (`name@domain.com`), use it directly. Try to look it up against contacts (Phase 2) for context, but don't block on a match — first-touch emails are common where there's no contact record yet.

### 2. Name from the user's contacts

Most common. The user says "Sarah Khan" or "Marcus" or "Greg from Magnetorquer".

`GET <backend_url>/api/me/contacts?per_page=500&search=<name>` (or fetch all and fuzzy-match client-side if no search param). Find the matching contact by:
- Exact full-name match → use it.
- First-name match with company hint → use it.
- Multiple matches → render top 3 with name + company + last contacted, ask "Which one? (1 / 2 / 3 / cancel)".

Once matched, the contact has the email on `contact.email`. If `email` is null:
> "I have Sarah Khan in your contacts but no email address on her record. Want to add one now? (paste email / cancel)"

If the user pastes an email, save it via `PUT /api/me/contacts/<slug>` so future drafts work, then continue.

### 3. New person (not in contacts)

If the name doesn't match any contact AND the user pasted no email, ask:
> "I don't have anyone called `<name>` in your contacts yet. What's their email address? (paste email / cancel)"

Optionally offer to create the contact: "Want me to add them to your contacts as well? (yes / just draft for now)". If yes, `POST /api/me/contacts` with what's known. Continue either way.

## Phase 2 — Gather context for the draft

### Step 2a — Contact context (if recipient is in contacts)

If you have a `contact_slug` from Phase 1, fetch the rolling user-level dossier and recent conversation notes:

`GET <backend_url>/api/me/contacts/<slug>` — returns `contact_profile` (rolling dossier) and basic fields.

For each campaign the contact is in, optionally `GET <backend_url>/api/me/campaigns/<campaign_slug>/contacts/<slug>` to read recent `conversation_notes` (bullets) and `manual_touches`. Pull the most recent 5–10 bullets across campaigns — these are the freshest signal of what's been said.

If the contact has no dossier and no notes, that's fine — fall back to the basic name + role + company.

### Step 2b — Ask the user for purpose

Ask the user one short question to anchor the draft:

> "Quick — what's this email for? (one line is enough — e.g. 'follow up after our coffee on Tuesday', 'intro myself, Sarah suggested I get in touch', 'check in, no response to my LinkedIn message yet')"

Wait for the answer. This becomes `{{purpose}}` in the prompt.

If the user gives more detail than a one-liner ("follow up about the pricing she asked about, mention we're shipping the new tier next week"), keep the whole thing — that's `{{purpose}}` plus signal for `{{prior_context}}`.

### Step 2c — Synthesise prior_context

Combine into a 2-4 sentence `prior_context` blurb covering:
- How the user knows this person (from the dossier / notes / what the user just said)
- The most recent relevant exchange (last conversation note, last manual touch, last LinkedIn message — whatever's freshest)
- Any standing context from the dossier worth mentioning (their stated priorities, their decision style)

If the contact is brand new (not in contacts before this skill ran), prior_context is just what the user told you in 2b.

## Phase 3 — Load voice + offer

### Step 3a — Voice profile

`GET <backend_url>/api/me/config` — pull `config.voice_profile`. Same shape Sales Helper uses (`summary`, `tone`, `do`, `dont`, `phrases`).

If `voice_profile` is null, draft in a generic professional tone but tell the user once at the end:
> "Heads up — your voice profile isn't set up yet. The draft above is in a generic tone. Run `/hhq:tune-voice` to teach me how you actually write."

### Step 3b — Offer profile

From the same `/api/me/config` response, pull `config.offer` (one-sentence) and `config.offer_hook` if present. Used by the prompt to keep the draft grounded in what the user actually does — even if the email isn't pitching, the angle stays consistent.

Campaign-pinned project? Resolve `<project-dir>/.hhq-campaign.json` and prefer the campaign's offer / voice_additions if present. Same precedence as `surface-next-5`.

## Phase 4 — Generate the draft

Fetch the prompt from the backend (single source of truth, tunable via Filament admin without a plugin release):

`GET <backend_url>/api/mcp/prompts/draft_email`

Substitute placeholders with the data gathered:

| Placeholder | Source |
|---|---|
| `{{user_name_or_self}}` | `config.user_name` or "the user" |
| `{{offer_summary}}` | `offer_profile.summary` or `config.offer` |
| `{{offer_hook}}` | `config.offer_hook` |
| `{{voice_summary}}` / `{{voice_tone}}` / `{{voice_do}}` / `{{voice_dont}}` / `{{voice_phrases}}` | `voice_profile` fields |
| `{{prospect_first_name}}` / `{{prospect_last_name}}` | from contact or what the user supplied |
| `{{prospect_position}}` / `{{prospect_company}}` | from contact (blank if first-touch unknown) |
| `{{prior_context}}` | the 2-4 sentence blurb from Phase 2c |
| `{{purpose}}` | the user's one-line answer from Phase 2b |

Run the substituted prompt through Claude (in this session — no separate API call needed; you're already running here). The prompt's output format is strict:

```
SUBJECT: <subject line>

<email body>
```

Parse the SUBJECT line and the body.

## Phase 5 — Show the draft

Render with light context:

```
**Draft email to Sarah Khan (sarah@acme.com):**

Subject: Following up from Tuesday's coffee

Hi Sarah,

Great catching up Tuesday. The point you made about the rollout
sequence stuck with me — we've actually shipped a tier that solves
exactly that, happy to walk you through it whenever you've got a
spare 20 minutes.

Brad

---
**Push to Gmail as a draft?** (yes / edit / cancel)
```

Notes:
- Show recipient name + email so the user knows the To target.
- Show the subject on its own line above the body.
- If you used `<<placeholders>>`, surface them above the action prompt: "*Placeholders: <<TIME>>, <<PROPOSAL VERSION>> — replace before sending.*"
- If voice_profile was missing, append the heads-up note from Phase 3a after the draft.

## Phase 6 — Handle the user's response

### Yes / approve / "push it"

`POST <backend_url>/api/mcp/gmail/create_draft` with:

```json
{
  "to": "sarah@acme.com",
  "subject": "Following up from Tuesday's coffee",
  "body": "<the draft body verbatim — no leading/trailing whitespace>"
}
```

Optional `cc` / `bcc` arrays of email addresses if the user asked for them in Phase 2.

Response includes the draft ID. On success:

> "Draft pushed to Gmail — open your Drafts folder to review and send.
>
> Reminder: Helper HQ never sends on your behalf. The draft is sitting in your Drafts folder, ready for your final pass. Click send when you're ready."

### Edit / "change X"

Conversational. Accept:
- "Make it shorter" → re-draft, render, ask again.
- "Change the subject to X" → swap subject only, re-render.
- "Drop the second paragraph" → re-draft.
- "Tone is too formal" → adjust voice and re-draft.
- "Add a sentence about the pricing tier" → add and re-draft.
- "Cc Greg" → add to cc list, re-render.

Re-render and ask "Push to Gmail? (yes / edit / cancel)" again. No iteration cap.

### Cancel / no

Don't push anything.
> "No draft pushed. Nothing changed in Gmail. Run again any time."

## Things you must NOT do

- Do NOT send the email. Helper HQ never sends. Always pushes to drafts. The user opens Gmail and clicks send.
- Do NOT skip Phase 2b (asking for purpose). The whole point of `prior_context` + `purpose` is to anchor the draft in something specific. Without it the email reads as generic outreach.
- Do NOT invent facts the user hasn't given you. Prices, dates, commitments, names of people not in the dossier — placeholder them with `<<>>` brackets and flag in Phase 5.
- Do NOT pull a recipient's email from a fuzzy guess. If contact lookup returns multiple matches, ask. If lookup returns nothing and no email was pasted, ask.
- Do NOT use `push_draft` (the reply-in-thread endpoint). This skill creates a fresh draft via `create_draft`. Different endpoint, different shape.
- Do NOT modify `<project-dir>/.hhq-session.json` except for JWT refresh.
- Do NOT call `create_draft` more than once per skill invocation. If the user wants a different version, edit and re-draft until they approve, THEN push once.
- Do NOT reuse the `draft_opener` prompt — that one is for cold LinkedIn DMs and produces a single short paragraph with no subject line. This skill uses the `draft_email` prompt which produces SUBJECT + body and bans em dashes / AI cliches more aggressively.

## Edge cases to handle gracefully

- **Recipient has no email on their contact record AND user didn't paste one** → ask once, save the email back to the contact via `PUT /api/me/contacts/<slug>` if supplied, continue.
- **Multiple contacts match the name** → render top 3, ask user to pick. Don't guess.
- **Contact has a dossier but no recent conversation notes** → use the dossier alone for prior_context. Don't fabricate recency.
- **Brand-new person, no contact, no prior context** → prior_context is just what the user told you in 2b. The prompt handles thin context — it'll lean harder on the purpose line.
- **Voice profile missing** → draft in a generic tone but flag it in Phase 5 (per Phase 3a).
- **User typed an obviously invalid email** → backend's validator will return 422. Surface the error and ask for a corrected address.
- **`create_draft` returns 502 `gmail_api_error`** → surface honestly, suggest retry in a minute.
- **`create_draft` returns 409 `gmail_revoked`** → tell the user to re-run `/hhq:connect-gmail`.
- **The draft naturally wants attachments** → out of scope. Push the text-only draft, tell the user to attach files manually in Gmail before sending.
