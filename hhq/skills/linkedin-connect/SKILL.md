---
name: linkedin-connect
description: Sales Helper — find people in your contacts you're not yet connected to on LinkedIn and send personalised connection requests in batches. Triggers on "let's connect with people on linkedin", "find people I should connect to", "connect with prospects on linkedin", "/hhq:linkedin-connect", optional source/limit args ("/hhq:linkedin-connect business-cards", "/hhq:linkedin-connect 20"). Pulls a ranked queue from `GET /api/me/linkedin-connect/queue` (business-card source first — the dogfood case where contacts almost always lack a linkedin_url; LinkedIn-CSV imports last because they're already mostly connections). Per contact, uses the Chrome connector to navigate or search, detects connection state from the visible action button (Connect / Pending / Message / Follow-with-More-dropdown), drafts a short note in the user's voice referencing how the user knows them, shows for review, then on approve clicks Connect → Add note → Send via Chrome and PATCHes the contact's `linkedin_connection_status` to `pending`. Handles LinkedIn's free-tier monthly note limit gracefully (offers no-note send + suggests Premium). Surfaces the LinkedIn search URL for any contact the skill can't find for the user to paste back the right profile URL.
---

# LinkedIn Connect — Sales Helper

You are working through a queue of contacts the user hasn't yet connected to on LinkedIn. For each one: find their profile (using the stored URL or LinkedIn search), check the connection state from the visible button, draft a short personalised connection note, get the user's approval, and send the request via the Chrome connector. Persist the result to the backend so the contact doesn't re-surface and so the companion `/hhq:linkedin-check-pending` skill can sweep for acceptances later.

## When this skill runs

Trigger when the user says any variant of:

- "let's connect with people on linkedin"
- "find people I should connect to"
- "connect with my prospects on linkedin"
- "let's send some linkedin connection requests"
- "linkedin connect"
- "/hhq:linkedin-connect"
- "/hhq:linkedin-connect business-cards" (source filter)
- "/hhq:linkedin-connect 20" (custom limit)

Do NOT trigger if the user is asking to draft a *message* to an existing connection (that's `/hhq:draft-reply` for Gmail or a Chrome-driven LinkedIn DM workflow). This skill is purely for sending **new connection requests**.

## What success looks like

A typical run: 8-10 contacts processed, ~5-7 connection requests sent (the rest split between "already connected" detected + skipped, "not findable" surfaced for manual paste, and "user skipped"), all status changes persisted to the backend so the same contacts don't surface tomorrow.

## Phase 0 — Auth and Chrome connector

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory` (no arguments) to get the persistent Cowork project folder. Save as `<project-dir>`.

If the tool isn't registered (rare CLI case), fall back to `~/.hhq/sales-helper/`.

### Step 0b — Resolve auth (per-project session)

Read `<project-dir>/.hhq-session.json` (per-project session auth, v0.11+).

- **Found** → parse `backend_url`, `license_key`, `session_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming. Continue.
- **Neither found** → "No auth — say `/hhq:connect` to link this project (or `/hhq:onboard` if you're brand-new)." Stop.

If `jwt_expires_at` is past or within 60s of expiry, proactively refresh: POST `<backend_url>/api/refresh` with `Authorization: Bearer <old jwt>` (the endpoint accepts expired tokens). Save the new `jwt` + `jwt_expires_at` to `.hhq-session.json` (preserving other fields).

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key.

**On 401 from any API call below**, read `error.code` from the response body and recover ONCE:

- `token_expired` → POST `<backend_url>/api/refresh` with the current JWT. Save new JWT. Retry the original call.
- `session_revoked` or `invalid_token` → POST `<backend_url>/api/activate` with the **existing `session_id` + `license_key` from `.hhq-session.json`** (NOT a fresh UUID — reusing the same UUID keeps this idempotent and avoids burning a slot). Save new JWT. Tell the user: *"Your session for this project had been released — I've re-established it. If that wasn't intentional, release it again from `/sessions` and close this chat."* Retry.
- `license_inactive` → tell the user to contact `help@helperhq.co`. Stop.

**On 403 during recovery** relay the backend's `error.message` verbatim and stop (`session_limit_reached` includes the `/sessions` URL).

**On a second 401** of the same call after recovery, surface honestly and stop. Never loop. Never generate a fresh session UUID — only `/hhq:connect` and `/hhq:onboard` mint new UUIDs.

### Step 0c — Verify Chrome connector is loaded

Look at the tools available in your current session for any `mcp__Claude_in_Chrome__navigate` / `mcp__Claude_in_Chrome__read_page` / `mcp__Claude_in_Chrome__find` / `mcp__Claude_in_Chrome__form_input`. If those aren't loaded, halt:

> "I need the Chrome connector to drive LinkedIn for connection requests. Install Claude in Chrome from your Cowork/Claude.ai connector settings, start a new chat, and try again."

### Step 0d — Verify the user is logged into LinkedIn

Navigate to `https://www.linkedin.com/feed/`. If the page redirects to login or shows a logged-out marketing page, halt:

> "You're not logged into LinkedIn in your Chrome session. Open `https://www.linkedin.com/login`, sign in, then re-run this skill."

If the feed loads, continue. (No need to read the full feed — just confirm the URL stayed on `/feed/` and didn't bounce.)

## Phase 1 — Fetch the queue

### Step 1a — Parse args

The user may have invoked with a source filter or a limit:

- `/hhq:linkedin-connect` → default: limit 10, no source filter
- `/hhq:linkedin-connect business-cards` → source=`business_card`
- `/hhq:linkedin-connect crm` → source=`crm_csv`
- `/hhq:linkedin-connect 20` → limit=20, no source filter
- `/hhq:linkedin-connect 5 business-cards` → limit=5, source=business_card (either order)

Map friendly names: `business-cards` → `business_card`, `crm` → `crm_csv`, `linkedin` → `linkedin_csv`, `spreadsheet` → `spreadsheet`. Anything else passed as source — pass through verbatim.

### Step 1b — GET the queue

```
GET <backend_url>/api/me/linkedin-connect/queue?limit=<N>&source=<source>
```

Returns:

```json
{
  "queue": [
    {
      "contact_id": 42,
      "contact_slug": "card-greg-coleman",
      "first_name": "Greg",
      "last_name": "Coleman",
      "headline": "Founder @ Magnetorquer",
      "company": "Magnetorquer Pty Ltd",
      "position": "Founder",
      "email": "greg@magnetorquer.com",
      "linkedin_url": null,
      "source": "business_card",
      "linkedin_connection_status": null,
      "linkedin_connection_checked_at": null
    },
    ...
  ],
  "limit": 10,
  "source": "business_card",
  "total": 27
}
```

If `total == 0`:

> "Nothing to connect to right now — every contact in your network either has `connected` status or hasn't been ingested yet. If you've just imported business cards, give it a minute and try again, or run `/hhq:ingest-contacts`."

Stop.

### Step 1c — Fetch user voice + default-campaign offer for note drafting

User-level: `GET <backend_url>/api/me/config` → cache `voice_profile`.

Default-campaign: `GET <backend_url>/api/me/campaigns/default/config` → cache `offer`, `offer_hook`, `offer_profile`.

These shape the connection note: voice for tone, offer for what to anchor the note around if there's no card-context to lean on.

### Step 1d — Tell the user what's about to happen

> "Got it — I've got `<N>` contacts to try connecting with on LinkedIn. I'll work through them one at a time. For each one I'll find their profile, draft a short note in your voice, show it to you for review, and send the request when you say go. You can skip any time. Let's start."

Then start the per-contact loop.

## Phase 2 — Per-contact loop (sequential)

For each contact in the queue, run Steps 2a–2g. After each one give a brief progress note in chat ("✓ 1/10 — Greg, request sent. Next: Marina."). Sequential — don't try to parallelise. Connection requests share a per-day rate limit, so going one at a time is also a soft pacing.

If the user types "stop" / "pause" / "enough" mid-loop, halt cleanly after finishing the current contact.

### Step 2a — Find the profile

Two paths:

**Path A — known linkedin_url:**

If `contact.linkedin_url` is non-empty, navigate directly:

```
mcp__Claude_in_Chrome__navigate to <linkedin_url>
```

Wait for the page to load (check for the profile name in the title or H1).

**Path B — no linkedin_url (search):**

Construct a LinkedIn search URL:

```
https://www.linkedin.com/search/results/people/?keywords=<URL-encoded "first_name last_name company">
```

Drop the company if last_name is missing. Drop position too — too narrow.

Navigate. Read the search results page (use `mcp__Claude_in_Chrome__read_page`). Identify candidates by matching:
- Name (exact match on first + last)
- Company (substring match — LinkedIn often shows "Founder at Magnetorquer Pty Ltd")
- Headline keywords

If exactly one strong match → tell the user, navigate to that profile:

> "Found Greg Coleman — Founder at Magnetorquer Pty Ltd. Looks like the right one."

If 2-3 plausible matches → list them with the LinkedIn URL of each, ask which one (or "none"):

> "Found 3 candidates for Greg Coleman:
> 1. Greg Coleman — Founder, Magnetorquer Pty Ltd — `https://...`
> 2. Greg Coleman — VP Sales, OtherCo — `https://...`
> 3. Greg Coleman — Software Engineer, BigCorp — `https://...`
>
> Which one? (1, 2, 3, or `none`)"

If 0 matches OR the user said `none` → mark not_findable and surface the search URL for a manual try:

> "Couldn't find a confident match for Greg Coleman. Want to paste their LinkedIn profile URL? I'll try again with that. Or say `skip`.
>
> Try the search yourself: `https://www.linkedin.com/search/results/people/?keywords=Greg%20Coleman%20Magnetorquer`"

If the user pastes a URL → navigate there, continue to Step 2b. Backfill the URL on the contact via the PATCH in Step 2g.
If the user says `skip` → PATCH `linkedin_connection_status: not_findable` (Step 2g shape, no note), continue to next contact.

### Step 2b — Read the profile and detect connection state

Once on a profile page, use `mcp__Claude_in_Chrome__read_page` (or `mcp__Claude_in_Chrome__find` with selector hints) to identify the visible primary action button.

LinkedIn shows one of these near the top of the profile (under the headline / location):

| Visible button | Meaning | What to do |
|---|---|---|
| **Message** | Already connected | mark `connected`, skip to next |
| **Pending** | Request already sent (by you, or accepted but not loaded yet) | mark `pending`, skip to next |
| **Connect** | Not connected, can request | proceed to Step 2c |
| **Follow** (no Connect visible) | 3rd+ degree, Connect lives in the **More** / **...** dropdown | open dropdown, look for "Connect" — if present, proceed to Step 2c; if absent, fall through to "no Connect option" below |
| **Withdraw** | You sent a request that's still pending | mark `pending`, skip to next |
| No usable button visible (page didn't load right, profile private/restricted) | unknown | mark `not_findable` (so it retries in 30 days), skip |

For the **Follow → More dropdown** case: use the Chrome connector to click the "More" / "..." button (CSS selector commonly `button[aria-label*="More actions"]` or text-match on "More"), wait for the dropdown menu, then check whether "Connect" is one of the menu items. If yes, proceed (the actual click happens in Step 2e). If no, this is a 3rd-degree+ profile LinkedIn won't let you connect to without a shared affiliation — mark `not_findable` and continue.

Tell the user the detected state in one line — "Already connected." / "Connect button visible." / "Connect available via More dropdown." / "Can't see a Connect option — they're outside your network."

### Step 2c — Get card context for note drafting (if business_card source)

Business cards capture meeting context that matters for the note. If `contact.source == 'business_card'` AND there's anything in the contact's notes / dossier about where they met (read once via `GET /api/me/contacts/<slug>/dossier` if you don't already have it), use that context as the lead in the note ("Met you at the SmallSat conference last week — would love to connect.").

For other sources without explicit context, use the offer-anchored opener pattern: short reference to what they do, soft connect.

### Step 2d — Draft the note

Constraints:

- **300 characters max** (LinkedIn's free-tier limit; Premium gets 1900 but stay short anyway — long notes feel like spam).
- **No exclamation marks** unless the user's `voice_profile.tone` explicitly mirrors them.
- **No emoji.**
- **Lead with how you know them** if business_card source (event, mutual contact). Otherwise lead with what they do.
- **One soft connect line** — "would love to connect", "happy to swap notes", never "let's set up a call".
- **No pitch.** This is a connection request, not an opener. Save the actual offer conversation for after they accept.

Voice integration: pull `voice_profile.summary` + `do` + `dont` + `phrases` from cached config. Apply lightly — connection notes are too short to fully showcase voice but tone, exclamation usage, and signature phrases should match.

Examples (illustrative — don't copy):

**Business card source, met at event:**
> Hey Greg — great chat at SmallSat last week about the magnetorquer launch. Would love to connect properly here.

**LinkedIn-CSV source, role-aware:**
> Hi Marina — saw you're heading up FlightDeck. We work with founders at your stage on outbound — happy to compare notes whenever useful.

**CRM source, prior conversation context unknown:**
> Hi Tim — your work on propulsion at Gilmour caught my eye. Would love to connect and follow what you're shipping.

**Card source, no event context (just a card scan):**
> Hi Sarah — picked up your card recently and wanted to connect properly here. Looking forward to following what you're up to at Acme.

### Step 2e — Show the draft for review

```
**Greg Coleman** — Founder, Magnetorquer Pty Ltd
LinkedIn: https://www.linkedin.com/in/greg-coleman-magnetorquer
Source: business card

┄ Draft connection note (218 / 300 chars) ┄

Hey Greg — great chat at SmallSat last week about the magnetorquer launch. Would love to connect properly here.

What now?
  send       — send with this note
  edit       — let me revise the note
  no-note    — send without a note (saves your monthly note allowance)
  skip       — skip this contact, move to the next
  withdraw   — mark not_findable instead (don't show again for 30 days)
```

Honour edits conversationally. After each edit, show the updated draft + char count and re-ask. If the edit pushes past 300 chars, warn and ask the user to trim.

### Step 2f — Send via Chrome

On `send` or `no-note`:

1. Click the Connect button (or the "Connect" item in the More dropdown if Step 2b found it there). Use `mcp__Claude_in_Chrome__find` with text="Connect" + a button-role filter to locate, then `mcp__Claude_in_Chrome__computer` click.

2. LinkedIn opens a small modal: "Add a note to your invitation?" — two buttons: "Add a note" / "Send without a note".

   - On `send` → click **Add a note** → wait for the textarea to appear → use `mcp__Claude_in_Chrome__form_input` to fill the textarea with the approved note → click **Send**.
   - On `no-note` → click **Send without a note** directly.

3. Watch for the **monthly note limit modal**. LinkedIn shows: "You've used all your free personalised invitations" (or similar wording) when the user has sent ~10 personalised connection notes in the rolling month. If you detect this:

   > "LinkedIn just told you you've hit your free-tier monthly limit on personalised connection notes (~10 per month). You've still got two options here:
   > - Send this one without a note (no personalisation, but the connection request still goes through).
   > - Skip and try again next month — or upgrade to LinkedIn Premium for higher limits.
   >
   > What now? (`no-note` / `skip` / `upgrade-info`)"

   On `no-note` → click "Send without a note" on the limit modal (or back out and click Connect → "Send without a note").
   On `skip` → close the modal, mark the contact `unknown` (not pending — we didn't actually send), continue.
   On `upgrade-info` → tell the user briefly that LinkedIn Premium Career / Business gives higher InMail and personalised invite allowances, then ask `no-note` / `skip`.

4. Watch for the **weekly send limit** modal — LinkedIn caps total connection requests at ~100/week (the cap shifts; sometimes per-day bursts trip a softer "slow down" warning). If you detect *any* "you've reached your weekly invitation limit" wording, halt the whole batch:

   > "LinkedIn just hit the weekly invitation limit. We can't send more this week without LinkedIn restricting your account further. We've sent `<X>` so far this run; the remaining `<Y>` are still in the queue and will surface next time you run `/hhq:linkedin-connect`. Try again in a few days."

   PATCH the current contact back to `unknown` (we didn't actually send) and stop the loop. Move to Phase 3 with whatever was sent successfully.

5. On a successful send (Connect modal closed, button now reads "Pending"), confirm in chat:

   > "✓ Sent to Greg."

### Step 2g — PATCH the backend

After the send (or skip / not-findable / connected / declined):

```
PATCH <backend_url>/api/me/contacts/<contact_slug>/linkedin-connection
Content-Type: application/json
Authorization: Bearer <jwt>

{
  "status": "pending" | "connected" | "not_findable" | "unknown" | "declined" | "withdrawn",
  "linkedin_url": "https://...",   // only if you backfilled it during search
  "note": "<the sent note>"        // only on status=pending; null/omitted on no-note
}
```

The backend stamps `linkedin_connection_checked_at` automatically and stamps `linkedin_connection_requested_at` when status becomes `pending`. On `no-note` sends, omit the `note` field entirely (or pass null) — the backend allows null.

### Step 2h — Move to next

Continue the loop until queue is exhausted, the user stops, or the LinkedIn weekly limit halts the run.

## Phase 3 — Wrap up

Once the loop ends (naturally or via halt), summarise:

```
Done. Out of <queue size>:
  ✓ <X> sent (with note: <X-Y>, no-note: <Y>)
  • <A> already connected
  • <B> already pending
  ⚠ <C> not findable — try `/hhq:linkedin-connect` again later, or paste their URL via `/hhq:contact <name>`
  ↷ <D> skipped

Sent requests will move to "pending" until they accept (or a few months pass and LinkedIn auto-expires them). Run `/hhq:linkedin-check-pending` in a week or two to sweep for acceptances.

Pipeline view: https://helperhq.co/pipeline
```

If the LinkedIn weekly limit halted the run, mention that explicitly:

> "Halted early — LinkedIn's weekly send limit kicked in. Try again in 3-5 days."

## Things you must NOT do

- Do NOT send a connection request without explicit user approval per contact ("send" or "no-note"). Even when the user says "go fast", surface each note for one yes/no.
- Do NOT use emoji in connection notes.
- Do NOT use exclamation marks unless the user's voice profile mirrors them.
- Do NOT pitch the offer in the connection note. Save the offer for after they accept (`/hhq:research-and-draft` is the right tool then).
- Do NOT exceed 300 characters in the note (LinkedIn cuts at 300 on free; Premium higher but stay short anyway).
- Do NOT click any "Send" button on LinkedIn without a confirmed user approval for THIS specific contact this run.
- Do NOT auto-retry on a rejected / blocked send. Mark and move on.
- Do NOT sweep `pending` contacts in this skill — that's `/hhq:linkedin-check-pending`. This skill only handles new requests.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` / `jwt_expires_at` after a refresh.
- Do NOT log the JWT, licence key, or auth file contents.
- Do NOT navigate away from the profile page mid-send (the Connect modal closes if you do).

## Edge cases to handle gracefully

- **LinkedIn page loads slowly / "We're loading…" persists** → wait up to ~10s, retry the read once. If still no usable content, mark `unknown` and move on.
- **Profile is private / "Profile not available"** → mark `not_findable`, surface the search URL in case the user wants to confirm a different person, move on.
- **Profile name doesn't match the contact's first+last** → surface to the user: "The profile at this URL is `<page name>`, but the contact in your DB is `<contact name>`. Right person? (yes / no / different URL)". On yes, continue with this Step 2b state. On no/different, treat as "no match", offer paste-back.
- **More dropdown opens but Connect is missing AND there's no Connect button anywhere** → 3rd+ degree with no shared affiliation. Mark `not_findable`. Don't try to send a Follow as a workaround — Follow ≠ Connect.
- **Connect modal opens but Send button is greyed out** → likely text validation on the note field (empty, too long). Re-check the note length, fix if needed, retry the send once. If still blocked, mark `unknown` and move on.
- **Page redirects to LinkedIn login mid-loop** → halt the whole run with: "LinkedIn signed you out mid-run. Sign in again in your Chrome session and re-run." Don't try to drive a login flow.
- **CAPTCHA / "verify you're a real person" challenge appears** → halt: "LinkedIn is asking you to verify you're a real person. Solve the challenge in your browser, then re-run." Never try to solve it via the connector.
- **PATCH back to backend fails (network, 5xx)** → tell the user honestly: "Sent the request to <name> on LinkedIn, but couldn't sync the status to Helper HQ. They might surface again next run." Continue with the next contact — don't crash.
- **User pastes a URL that doesn't look like a LinkedIn profile URL** (e.g. a company page) → tell them, ask for a profile URL, or skip.
- **Queue is fresh but every entry already has `linkedin_url` AND status `connected`** (timing race: queue fetched before another session updated) → page through (the queue endpoint excludes `connected`, so this is rare; if it does happen, treat as empty queue).
