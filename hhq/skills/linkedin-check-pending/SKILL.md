---
name: linkedin-check-pending
description: Sales Helper — sweep contacts with `linkedin_connection_status = pending` to see whether they've accepted, are still pending, or LinkedIn has auto-withdrawn the request (which happens silently around 6 months in). Triggers on "check my pending linkedin requests", "any linkedin acceptances", "sweep my pending", "/hhq:linkedin-check-pending", optional limit ("/hhq:linkedin-check-pending 30"). Pulls from `GET /api/me/linkedin-connect/pending` (oldest first), navigates each profile via the Chrome connector, reads the visible action button (Message → connected, Pending → still pending, Connect → withdrawn / expired, no usable button → not_findable), PATCHes status. Newly-confirmed `connected` contacts get a one-line nudge to optionally surface for next-step outreach via `/hhq:research-and-draft`. Pure detection — never sends new requests.
---

# LinkedIn Check Pending — Sales Helper

You are sweeping the user's outstanding LinkedIn connection requests to see what's happened. For each pending contact: navigate to their profile, read the visible action button, and update status accordingly. **Detection only — never send anything.** Companion to `/hhq:linkedin-connect` (which sends new requests); this one closes the loop.

## When this skill runs

Trigger when the user says any variant of:

- "check my pending linkedin requests"
- "any linkedin acceptances yet"
- "sweep my pending"
- "who's accepted my linkedin requests"
- "linkedin pending sweep"
- "/hhq:linkedin-check-pending"
- "/hhq:linkedin-check-pending 30" (custom limit; default 20)

Do NOT trigger if the user wants to *send* requests (use `/hhq:linkedin-connect` for that). This skill is purely a follow-up sweep.

## Phase 0 — Auth and Chrome connector

Same as `linkedin-connect` Steps 0a–0d:

- 0a — `mcp__ccd_directory__request_directory` for `<project-dir>`.
- 0b — Read `<project-dir>/.hhq-session.json`, refresh JWT if expiring within 60s. **Use the same canonical 401-recovery dispatch as `linkedin-connect` Step 0b** (token_expired → refresh; session_revoked / invalid_token → re-activate with the **existing** `session_id` from the file, NOT a fresh UUID; license_inactive → contact support; 403 from recovery → relay verbatim and stop).
- 0c — Verify Chrome connector is loaded; halt if not.
- 0d — Navigate to `https://www.linkedin.com/feed/` to verify logged in; halt if not.

Reuse the same patterns verbatim — don't re-implement. If a step fails, fail the same way `linkedin-connect` does.

## Phase 1 — Fetch the pending list

### Step 1a — Parse args

- `/hhq:linkedin-check-pending` → default: limit 20
- `/hhq:linkedin-check-pending 30` → limit=30 (max 100)

### Step 1b — GET the pending list

```
GET <backend_url>/api/me/linkedin-connect/pending?limit=<N>
```

Returns:

```json
{
  "pending": [
    {
      "contact_id": 42,
      "contact_slug": "card-greg-coleman",
      "first_name": "Greg",
      "last_name": "Coleman",
      "headline": "Founder @ Magnetorquer",
      "company": "Magnetorquer Pty Ltd",
      "position": "Founder",
      "email": "greg@magnetorquer.com",
      "linkedin_url": "https://www.linkedin.com/in/greg-coleman-magnetorquer",
      "source": "business_card",
      "linkedin_connection_status": "pending",
      "linkedin_connection_checked_at": "2026-04-15T...",
      "linkedin_connection_requested_at": "2026-04-15T...",
      "linkedin_connection_request_note": "Hey Greg — great chat at SmallSat..."
    },
    ...
  ],
  "limit": 20,
  "total": 42
}
```

If `total == 0`:

> "No pending LinkedIn requests on file. Either you haven't sent any via `/hhq:linkedin-connect` yet, or all prior pending have already been resolved."

Stop.

### Step 1c — Tell the user what's about to happen

> "Got it — sweeping `<N>` of `<total>` pending LinkedIn requests. For each one I'll check the profile to see if they've accepted. This is detection-only — I won't send anything. Takes a couple minutes."

Then start the per-contact loop.

## Phase 2 — Per-contact loop (sequential)

For each contact in the pending list, run Steps 2a–2c. Brief progress note after each ("✓ 1/20 — Greg accepted!").

### Step 2a — Navigate to the profile

Every entry in the pending list has a `linkedin_url` (we wouldn't have been able to send a request without one). Navigate directly:

```
mcp__Claude_in_Chrome__navigate to <linkedin_url>
```

If the page fails to load, redirects to a login wall, or shows "Profile not available", mark `not_findable` and continue. (The contact previously had a working profile — if it's gone now, someone changed something.)

### Step 2b — Detect current state

Read the page. Check the primary action button under the headline:

| Visible button | Meaning | New status |
|---|---|---|
| **Message** | They accepted — you're connected | `connected` |
| **Pending** | Request still outstanding | leave as `pending` (just bump checked_at) |
| **Withdraw** | Same as Pending — your request is still out | leave as `pending` |
| **Connect** (visible again) | Request was withdrawn — either they declined, or LinkedIn auto-expired (~6 months) | `withdrawn` |
| **Follow** (no Connect, no Message, no Pending) | They're 3rd-degree+ now AND you have no shared affiliation — the prior request likely expired or they removed connections | `withdrawn` |
| Profile not loadable / private / 404 | Can't tell | `not_findable` |

Special case: if the visible button is **Connect** AND the user expects to see Pending (you sent a request that should still be live), that's the auto-withdrawal case. Note it in the per-contact line:

> "↺ Greg — request expired, available to re-send."

The user can re-run `/hhq:linkedin-connect` later to re-queue withdrawn contacts (the queue endpoint includes `withdrawn` status by design).

### Step 2c — PATCH the backend

```
PATCH <backend_url>/api/me/contacts/<contact_slug>/linkedin-connection
Content-Type: application/json
Authorization: Bearer <jwt>

{
  "status": "connected" | "pending" | "withdrawn" | "not_findable"
}
```

Don't pass `note` here — this is detection, not a re-send. The backend preserves the original `linkedin_connection_request_note` and `linkedin_connection_requested_at` automatically (it only clears them on a status of `unknown`).

If the status is unchanged (`pending` → `pending`), still PATCH so `linkedin_connection_checked_at` updates — that's how we know the sweep ran.

## Phase 3 — Wrap up

Once the loop ends, summarise:

```
Done. Out of <N> pending requests:
  ✓ <X> accepted (now connected)
  ⏳ <Y> still pending
  ↺ <Z> withdrawn / expired (re-queueable via /hhq:linkedin-connect)
  ⚠ <W> not findable (profile changed or restricted)
```

If `X > 0` (newly-accepted connections), surface a low-key nudge:

> "Newly accepted: `<comma-separated names>`.
>
> Now's a good time to send actual openers — `/hhq:research-and-draft` will research them and draft a Greg-style first message. Or `/hhq:contact <name>` to look at one."

Don't auto-trigger anything — just surface the option.

## Things you must NOT do

- Do NOT send any new connection requests. This skill is detection-only. If a contact's status flipped to `withdrawn` and the user wants to re-send, route them to `/hhq:linkedin-connect`.
- Do NOT click any buttons on LinkedIn that change state — no Message, no Connect, no Follow. Read-only navigation.
- Do NOT modify the original `linkedin_connection_request_note` — the backend preserves it across status changes (except on explicit reset to `unknown`).
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` / `jwt_expires_at` after a refresh.
- Do NOT log the JWT, licence key, or auth file contents.
- Do NOT batch the PATCHes — one per contact, after each detection. Keeps the backend in sync with what the user sees in chat.

## Edge cases to handle gracefully

- **CAPTCHA / verify-you're-real-person challenge mid-sweep** → halt with the same message as `linkedin-connect`. Detection sweeps don't trigger CAPTCHA as easily as send loops, but it can happen.
- **Profile redirected to a different person** (LinkedIn handles get reassigned occasionally) → if the page name doesn't match the contact, surface to user: "The profile at this URL is now `<page name>` — but the contact is `<contact name>`. Mark `not_findable` and let user fix the URL via `/hhq:contact`?". Default to `not_findable`.
- **Backend PATCH fails for a contact** → tell the user honestly, continue the sweep. Don't crash on a single failed PATCH.
- **Pending list larger than the limit** → after Phase 3 summary, mention: "There are `<total - limit>` more pending requests not swept this run. Re-run with `/hhq:linkedin-check-pending <N>` to do more in one go."
- **User has many old pending requests (>100)** that all turn out to be `withdrawn` (LinkedIn 6-month auto-expiry) → expect a heavy `withdrawn` count. Don't surprise them — the summary line names it clearly. They can then re-run `/hhq:linkedin-connect` to re-send to the most relevant ones.
