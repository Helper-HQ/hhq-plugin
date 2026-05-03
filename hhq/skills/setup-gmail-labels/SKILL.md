---
name: setup-gmail-labels
description: Admin Helper — one-time setup of the 5 default Helper HQ Gmail labels (To Do / Awaiting Reply / FYI / Notifications / Newsletters) on the user's Gmail account. Walks the user through the proposed label scheme, lets them rename or skip individual labels, then creates the labels in Gmail and persists the mapping. Required before triage-inbox and the background auto-rules (Newsletters / Notifications detection) can apply anything. Triggers on "set up gmail labels", "label setup", "configure my labels", "/hhq:setup-gmail-labels", or auto-chained from /hhq:connect-gmail after a successful OAuth completion. Idempotent — re-running only creates labels that don't exist yet.
---

# Setup Gmail Labels — Admin Helper

You are walking the user through their one-time Gmail label setup for Helper HQ. Five default labels — To Do / Awaiting Reply / FYI / Notifications / Newsletters — get created on their Gmail account, namespaced under `HHQ/` so they don't clash with anything the user already has. The user can rename or skip individual labels before we create them.

This is a **once-per-user setup** that unlocks: (a) the AI triage in `triage-inbox` (uses these labels for the bucket categorisation), and (b) the server-side auto-rules running on every background sync (Newsletters auto-archived via `List-Unsubscribe` detection, Notifications via `noreply@` patterns).

## When this skill runs

Trigger on:

- "set up gmail labels"
- "configure my labels"
- "label setup"
- "let's set up the labels"
- `/hhq:setup-gmail-labels`
- **Auto-chained from `/hhq:connect-gmail`** after a successful OAuth completion if labels haven't been set up yet

Don't trigger on generic "set up gmail" — that's `/hhq:onboard` Phase 7.5 territory (the access request opt-in).

## Phase 0 — Auth + Gmail connection check

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory`. Save as `<project-dir>`. Fallback `~/.hhq/sales-helper/`.

### Step 0b — Resolve auth

Read `<project-dir>/.hhq-session.json`. Refresh JWT if near expiry. Halt with `/hhq:connect` if missing.

### Step 0c — Confirm extended Gmail access

`GET <backend_url>/api/me/gmail/connection`.

- `connected: true` → continue.
- `connected: false` → halt:
  > "Label setup needs Helper HQ's direct Gmail integration (extended Gmail access — beta). You're not connected yet. Run `/hhq:onboard` to opt in, or `/hhq:connect-gmail` if you already have approval."

## Phase 1 — Check current label state

`GET <backend_url>/api/me/gmail/labels/config`.

Response:

```json
{
  "is_setup": false,
  "setup_completed_at": null,
  "labels": [
    {"canonical": "to_do", "display_name": "HHQ/To Do", "gmail_label_id": null, "color": "#b84a2e", "is_enabled": true, "archive_on_apply": false, "mark_read_on_apply": false, "description": "..."},
    ... (5 labels total)
  ]
}
```

Branch on `is_setup`:

- **`is_setup: true`** AND every enabled label has a `gmail_label_id` → already done. Tell the user briefly:
  > "Your Helper HQ labels are already set up:
  >
  > - HHQ/To Do
  > - HHQ/Awaiting Reply
  > - HHQ/FYI
  > - HHQ/Notifications
  > - HHQ/Newsletters
  >
  > Want to rename any of them, or disable some? (rename / disable / no, all good)"
  >
  > If "rename" or "disable", drop into Phase 3 with the existing entries. Otherwise stop.
- **`is_setup: false`** OR some enabled labels lack `gmail_label_id` → fresh setup or partial recovery. Continue to Phase 2.

## Phase 2 — Propose the labels

Render the proposed scheme:

```
**Helper HQ Gmail labels — proposed setup**

Each label sits under "HHQ/" in your Gmail sidebar (so they don't clash with your existing labels).

  1. **HHQ/To Do** — Incoming emails that need action from you.
     Stays in inbox. (red)

  2. **HHQ/Awaiting Reply** — Threads where you replied last and are
     waiting for a response. Out of inbox. (yellow)

  3. **HHQ/FYI** — Info-only messages you do not need to act on.
     Archived and marked read. (green)

  4. **HHQ/Notifications** — Machine notifications (passwords,
     verification codes). Auto-applied to anything from `noreply@`
     senders. Archived and marked read. (gray)

  5. **HHQ/Newsletters** — Subscription / marketing emails (anything
     with an unsubscribe link). Auto-applied during background sync.
     Archived and marked read. (purple)

Labels 4 + 5 will be applied automatically on every sync (every 15 min)
once setup completes. Labels 1, 2, 3 get applied during inbox triage
when you run `/hhq:triage-inbox`.

**Options:** apply all (recommended) / customise / cancel
```

## Phase 3 — Handle the user's response

### "Apply all" / "yes" / "looks good" / similar

Skip to Phase 4 with no overrides.

### "Customise" / "let me change some"

Drop into a per-label conversational loop. For each canonical label, the user can:

- **Rename** — change `display_name`. The HHQ/ prefix is recommended but not enforced; the canonical name (`to_do`, etc.) stays internal.
- **Disable** — set `is_enabled: false` so we don't create the label or apply it.
- **Skip / next** — leave as default and move on.
- **Done** — exit the loop, apply with whatever overrides have been collected.

Render the current state after each change so the user can see their progress. When they're done, summarise the final configuration and ask "Apply now? (yes / no)".

Build the `overrides` object as `{canonical: {display_name?, is_enabled?, archive_on_apply?, mark_read_on_apply?}}` for the API call.

### "Cancel" / "no"

Stop. No labels created.

> "No labels created. You can run this any time with `/hhq:setup-gmail-labels`. Note: triage-inbox and the auto-rules won't apply anything until labels are set up."

## Phase 4 — POST the setup

```
POST <backend_url>/api/me/gmail/labels/setup
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "overrides": { ... }   // empty object if "apply all"
}
```

Response on success:

```json
{
  "is_setup": true,
  "setup_completed_at": "...",
  "labels": [
    {"canonical": "to_do", "display_name": "HHQ/To Do", "gmail_label_id": "Label_42", "color": "#b84a2e", "is_enabled": true, ...},
    ...
  ]
}
```

Error responses to handle:

- `409 gmail_not_connected` → Phase 0c gating should have caught this; retry the connection check and route accordingly.
- `502 gmail_api_error` → Gmail API failure during label creation. Surface honestly: "Couldn't reach Gmail to create the labels — message: `<msg>`. Try again in a minute."
- `422 validation` → bad colour or display_name. Surface the validation error and re-prompt.

## Phase 5 — Confirm

```
**Done — labels created in Gmail.**

  ✓ HHQ/To Do
  ✓ HHQ/Awaiting Reply
  ✓ HHQ/FYI
  ✓ HHQ/Notifications
  ✓ HHQ/Newsletters

What happens next:

- Background sync runs every 15 minutes. The next sync will start
  auto-labelling Newsletters + Notifications in your inbox.
- Run `/hhq:triage-inbox` any time to categorise what's left into
  To Do / FYI / Awaiting Reply.
- Run `/hhq:manage-followups` to chase threads where you replied
  last and they've gone quiet.
```

If any labels were skipped or renamed, list them with their final names so the user can confirm.

## Things you must NOT do

- Do NOT create labels outside the canonical 5. Custom labels are a future concern (`manage-labels` skill, V1.1+).
- Do NOT modify gmail_label_id directly. The backend assigns it during setup; the PATCH endpoint deliberately ignores attempts to change it.
- Do NOT prompt for colour customisation in V1. We have sensible defaults (red/yellow/green/gray/purple). Custom colour editing is V1.1.
- Do NOT silently re-create labels that already exist. The setup endpoint is idempotent — re-running it only creates labels missing `gmail_label_id`. If Phase 1 detects all 5 are set up, route to "rename / disable" rather than re-creating.
- Do NOT modify `<project-dir>/.hhq-session.json` except for JWT refresh.
- Do NOT run on the Cowork generic Gmail connector. Phase 0c gates on extended Gmail access.

## Edge cases to handle gracefully

- **Partial existing setup** (e.g. user previously created 3 labels then their network died) → setup is idempotent. POSTing again only creates the missing 2. Confirm honestly: "Created 2 labels (Notifications, Newsletters); the other 3 already existed."
- **User has labels named exactly like ours already** (e.g. they manually created "HHQ/To Do") → Gmail's `users.labels.create` returns 409 Conflict. The backend would surface this as 502 `gmail_api_error`. Suggest renaming our label or theirs.
- **User customises a display_name with special characters** → Gmail allows most characters in labels but max 225 chars. The backend validates `max:225`. Surface validation errors to the user.
- **Setup runs successfully but `triage-inbox` complains "no labels"** → race condition where `setup_completed_at` is set but the label entries haven't been refreshed in a separate session. Tell user to re-run `/hhq:setup-gmail-labels` (idempotent).
