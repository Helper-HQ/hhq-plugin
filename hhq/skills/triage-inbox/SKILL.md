---
name: triage-inbox
description: Admin Helper — pulls the user's recent Gmail inbox, AI-categorises each thread into one of four buckets (action required, waiting on reply, low priority, can archive), presents the categorised list, and on user approval applies labels + archives in bulk via the HHQ Gmail MCP. Triggers on "triage my inbox", "clean up my inbox", "what needs my attention", "let's go through my inbox", "/hhq:triage-inbox", or after morning standup phrases like "what's in my inbox today". Requires extended Gmail access (HHQ OAuth) — routes the user to /hhq:connect-gmail if not connected. Looks up senders against the user's contacts so prospects, customers, and partners are flagged inline. Default UX is fast — one yes/no to approve all proposed actions; user can drop / change individual items before applying. Never sends, never permanent-deletes (uses Gmail's soft-trash via trash_thread, recoverable for 30 days).
---

# Triage Inbox — Admin Helper

You are helping the user clear and categorise their inbox. Pull the most recent threads, look at headers (sender + subject + snippet), classify each into one of four buckets, present the result in a tight list, and on approval apply the labels + archives in bulk. Done in two-to-three minutes for a typical morning inbox.

This is a **read-then-write** skill — read everything first, propose actions, ask for one approval, apply in bulk. Don't act on individual items as you go.

## When this skill runs

Trigger on:

- "triage my inbox"
- "clean up my inbox"
- "what needs my attention"
- "what's in my inbox today"
- "let's go through my inbox"
- "process my inbox"
- "show me what's waiting"
- `/hhq:triage-inbox`

Don't trigger on generic "check email" — only on explicit triage / cleanup intent.

## Architecture note — labels do most of the work

This skill assumes the user has run `/hhq:setup-gmail-labels` (auto-chained from `/hhq:connect-gmail`). The 5 HHQ labels (To Do / Awaiting Reply / FYI / Notifications / Newsletters) are the targets for triage classification.

Two of those labels — **Notifications** and **Newsletters** — are also auto-applied by the background sync (every 15 min), so by the time the user runs triage, most newsletters / machine notifications are already out of the inbox. Triage focuses on the **3 judgment buckets** (To Do / Awaiting Reply / FYI) for what's still in inbox after the auto-rules ran.

If the user hasn't set up labels yet, fall back to the legacy 4-bucket flow described in the "Legacy fallback" section at the bottom — applies a single `HHQ/Low priority` label + archives the can-archive bucket. Keeps triage usable even before label setup.

## Phase 0 — Auth + Gmail connection check

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory`. Save as `<project-dir>`. Fallback `~/.hhq/sales-helper/`.

### Step 0b — Resolve auth

Read `<project-dir>/.hhq-session.json`. Same refresh-on-near-expiry pattern as every other skill. Halt with `/hhq:connect` prompt if missing.

### Step 0c — Confirm extended Gmail access

`GET <backend_url>/api/me/gmail/connection` with `Authorization: Bearer <jwt>`.

Branch on the response:

- `{"connected": true, "email": "..."}` → continue.
- `{"connected": false}` → halt with:
  > "Triage uses Helper HQ's direct Gmail integration (extended Gmail access — beta). You're not connected yet. Run `/hhq:onboard` to opt in, or `/hhq:connect-gmail` if you already have approval."
- Any error → surface honestly, halt.

This skill **does not work** with Cowork's generic Gmail connector — that connector doesn't expose `archive_thread`, `trash_thread`, `label_thread`, or `list_inbox`. Extended Gmail access is the only path.

### Step 0d — Resolve label config

`GET <backend_url>/api/me/gmail/labels/config`.

- `is_setup: true` AND every enabled label has a `gmail_label_id` → use the **5-bucket flow** (Phase 1 onwards). Cache the `{canonical: gmail_label_id}` map.
- `is_setup: false` → user hasn't run `/hhq:setup-gmail-labels`. Two options:
  - Offer to run setup inline: "You haven't set up Helper HQ labels yet — want to do that now? It takes ~30 seconds and unlocks the full triage flow. (yes / use legacy single-label triage / cancel)"
  - If user says "use legacy", drop into the **Legacy fallback** flow at the bottom of this skill.
  - If user says "yes", trigger the `setup-gmail-labels` skill, then resume here.

## Phase 1 — Pull the inbox

`POST <backend_url>/api/mcp/gmail/list_inbox` with body:

```json
{ "max_results": 50 }
```

The default 50 is right for a morning triage — large enough to clear a backlog, small enough that AI classification is fast. The user can ask for more by saying "triage 100" or similar — bump `max_results` accordingly (cap 100 per the API).

Optional: if the user said "unread only" or similar, add `"query": "is:unread"`. Standard Gmail search syntax works in the `query` field.

Response shape:

```json
{
  "threads": [
    {
      "thread_id": "abc123",
      "snippet": "Hi Brad, just following up on...",
      "from": "Sarah Khan <sarah@acme.com>",
      "subject": "Re: Pricing for 50 seats",
      "date": "Wed, 1 May 2026 09:00:00 +0000",
      "message_count": 3,
      "label_ids": ["INBOX", "UNREAD"]
    }, ...
  ],
  "next_page_token": null,
  "result_size_estimate": 50
}
```

If `threads` is empty, tell the user "Nothing in your inbox to triage — all clear." and stop.

## Phase 2 — Look up senders against contacts

For each thread, parse the sender email out of the `from` field (e.g. `Sarah Khan <sarah@acme.com>` → `sarah@acme.com`). Then for the unique senders, batch-look-up against the user's contacts:

`GET <backend_url>/api/me/contacts?per_page=500` (paged if needed) — same endpoint sync-gmail uses for context. Build a map `{lowercase_email: {first_name, company, pipeline_stage_id, relationship_type}}`.

Use this to enrich each thread with sender context for display:

- If sender is a known contact with `relationship_type` containing `prospect` AND a non-terminal pipeline stage → "**prospect** in your sales pipeline"
- If `customer` → "**customer**"
- If `partner` / `supplier` → "**partner**" / "**supplier**"
- If `email_contact` only → just show the name, no badge
- If no match → unknown sender (leave neutral)

This context is for HUMAN reading in Phase 4 — it doesn't change classification rules in Phase 3. But it should bias the user's review (an "action required" item from a known prospect is higher-stakes than from an unknown sender).

## Phase 3 — AI-categorise each thread (model-side, in this session)

For each thread, classify into ONE of four buckets based on `subject`, `from`, `snippet`, and `label_ids`. **Use only the data we already have — DO NOT call `get_thread` for bodies.** Triage runs on metadata; per-thread body reads happen later (in `draft-reply` or `surface-followups`) when the user picks one to action.

### Buckets (5-label flow)

The 5 canonical labels map to AI classification buckets. Most newsletters / notifications are already filtered out by the auto-rules before triage runs, so in practice you'll see far fewer of those — but the rules still need to handle the case where one slipped through.

**`to_do`** → `HHQ/To Do` label, **stays in inbox**.
Needs action / response / decision from the user.
Signal patterns: questions in subject ("Quick question", "Can you...?"), explicit asks ("Could you let me know", "Need your sign-off", "Waiting on you"), `UNREAD` label, threads where the latest message is from someone else, anything from a known prospect / customer / partner that looks like substantive content. Calendar invites needing RSVP.

**`awaiting_reply`** → `HHQ/Awaiting Reply` label, **archived from inbox**.
User has already responded; ball is in the other party's court. Should be tracked but doesn't need inbox attention until they reply.
Signal patterns: latest message in thread is from the user, no recent inbound after, "will get back to you" style replies the user just sent.

**`fyi`** → `HHQ/FYI` label, **archived + marked read**.
Info-only — read it once, no action. Includes "thanks!" replies, project status updates, scheduling confirmations the user isn't the primary actor on.
Signal patterns: short acknowledgements ("got it, thanks"), cc'd threads where the user isn't the primary recipient, automated reports from people (not machines), non-urgent FYIs.

**`notifications`** → `HHQ/Notifications` label, **archived + marked read**.
Machine-sent — should mostly be auto-handled by the background sync, but catches any that slipped through.
Signal patterns: From `noreply@` / `no-reply@` / `notifications@` / `system@`, password resets, verification codes, system alerts.

**`newsletters`** → `HHQ/Newsletters` label, **archived + marked read**.
Subscription / marketing — should mostly be auto-handled by the background sync (List-Unsubscribe header detection), but catches any without that header.
Signal patterns: "Your weekly digest", "Update from <brand>", marketing language, footers mentioning unsubscribe even when no header present.

### Disambiguation rules

- If unsure between `to_do` and `fyi`: prefer `to_do` for unknown senders (safer false positive), prefer `fyi` for known low-stakes senders (CCs from your own team about non-actioned items, etc.).
- If from a known **prospect** in active pipeline stage AND looks like substantive content: at minimum `to_do`. Never `notifications` or `newsletters` even if pattern would match.
- If from a known **customer**: same — never `notifications` / `newsletters`.
- Anything from a sender on the user's `personal` relationship_type list: `fyi` minimum, hands-off bias.
- Calendar invites needing RSVP from any known sender → `to_do`.
- If a label is **disabled** in the user's config (e.g. they turned off `awaiting_reply`), don't classify into that bucket — pick the next best fit.

### Why the auto-rules matter

If the background sync has been running, by the time the user runs triage:
- Most newsletters are already labelled + archived (List-Unsubscribe detection)
- Most machine notifications are already labelled + archived (`noreply@` patterns)
- The inbox you're triaging is mostly **human-sent emails needing judgment** — exactly the cases that need AI categorisation

So in practice the `notifications` and `newsletters` buckets in triage will be small or empty for users with the auto-rules running. That's the intended behaviour.

## Phase 4 — Present the categorised list

Group the threads by bucket. Only render buckets that have items. Skip empty buckets entirely.

```
**Inbox triage — 35 threads**

═══ HHQ/To Do (12) — stays in inbox ═══

  • Sarah Khan (prospect) — Re: Pricing for 50 seats
  • Greg Cole (prospect) — Quick question on integration
  • Marcus Jones — Can you confirm the Tuesday call?
  ... (etc)

═══ HHQ/Awaiting Reply (8) — archived ═══

  • Anna Patel (customer) — Re: Quarterly review
  ... (etc)

═══ HHQ/FYI (12) — archived + marked read ═══

  • Team standup notes — yesterday
  • Project status weekly — Sprint 14
  ... (etc)

═══ HHQ/Notifications (2) — archived + marked read ═══

  • Stripe — Verification code
  ... (auto-rules usually catch these — these are stragglers)

═══ HHQ/Newsletters (1) — archived + marked read ═══

  • Some sender without unsubscribe header — Update
  ... (auto-rules usually catch these too)

**Proposed actions per the bucket settings:**
- Apply HHQ/To Do to 12 threads — kept in inbox
- Apply HHQ/Awaiting Reply to 8 threads — archived from inbox
- Apply HHQ/FYI to 12 threads — archived + marked read
- Apply HHQ/Notifications to 2 threads — archived + marked read
- Apply HHQ/Newsletters to 1 thread — archived + marked read

Apply all? (yes / no / edit)
```

Notes on rendering:

- **Cap at ~15 threads visible per bucket** (down from earlier 25 — five buckets means more total). Show "(plus N more)" if exceeded.
- **One line per thread:** sender name (with badge if a known contact), em-dash, subject. No date, no snippet.
- **No thread IDs visible** — track internally.
- **Skip empty buckets** in the rendered list to keep it scannable.
- **Use the user's display_name** for each label (from `/api/me/gmail/labels/config`) — if they renamed `HHQ/To Do` to `Action`, show `Action (12)` not `HHQ/To Do (12)`.

## Phase 5 — Handle the user's response

### Yes / approve / "go" / similar

Apply the actions per the user's label config (from Phase 0d). For each thread in each bucket:

1. **Look up the canonical label entry** — get `gmail_label_id`, `archive_on_apply`, `mark_read_on_apply`.
2. **Build the label modification:**
   - Always add the canonical label_id
   - If `archive_on_apply: true` → also remove `INBOX`
   - If `mark_read_on_apply: true` → also remove `UNREAD`
3. **Apply via `POST /api/mcp/gmail/label_thread`** for the add, AND `POST /api/mcp/gmail/unlabel_thread` for each remove. One thread can take 1-3 API calls.

   *(Implementation note: the MCP currently exposes single-action label/unlabel/archive endpoints. To save API calls, future optimisation could batch these into a single `modify_thread` endpoint that takes both add + remove arrays — same shape as the backend's `GmailService::modifyThread` already supports internally. For V1 the sequential calls are acceptable.)*

   Skip any label entry where `gmail_label_id` is missing — that label wasn't set up. Surface to the user: "Skipping `<canonical>` (label wasn't created — run `/hhq:setup-gmail-labels` to fix)."

### No / cancel / "actually no"

Stop. Apply nothing. Tell the user:
> "Nothing applied — your inbox is unchanged. Run again any time."

### Edit / "let me change some"

Conversational mode. The user can say:
- "Move Sarah Khan to action required" → recategorise that thread
- "Don't archive the LinkedIn one — keep it" → move from `can_archive` to `low_priority` (or whatever they want)
- "Archive everything in low priority too" → flip all `low_priority` to `can_archive`
- "Skip the apply" → cancel without applying

Accept changes, re-render the updated list, ask "Apply now? (yes / no / more changes)".

## Phase 6 — Report what was done

After the actions complete, brief summary:

> "Triage done.
>
> - **HHQ/To Do** (12) — labelled, kept in inbox.
> - **HHQ/Awaiting Reply** (8) — labelled + archived.
> - **HHQ/FYI** (12) — labelled + archived + marked read.
> - **HHQ/Notifications** (2) — labelled + archived + marked read.
> - **HHQ/Newsletters** (1) — labelled + archived + marked read.
>
> Your inbox now shows the 12 items that need you. Open Gmail to action them."

Use the user's display_name for each label in the summary.

If any action failed (e.g. Gmail API error on a specific thread), report it honestly:
> "Heads up: failed to apply HHQ/To Do to 1 thread (Gmail returned 404 — it may have been deleted in another tab). The other 11 applied cleanly."

## Legacy fallback (pre-label-setup users)

If the user runs triage before completing `/hhq:setup-gmail-labels` AND chose to use legacy mode (Phase 0d), fall back to the simpler 4-bucket flow:

- `action_required` → leave in inbox, no label
- `waiting_on_reply` → leave in inbox, no label
- `low_priority` → apply `HHQ/Low priority` label (create if missing via `create_label`)
- `can_archive` → archive via `archive_thread`

This is the pre-v0.21 behaviour. It works without label setup but loses the auto-rules + the richer 5-bucket categorisation. Encourage the user to run `/hhq:setup-gmail-labels` to upgrade.

## Things you must NOT do

- Do NOT call `get_thread` during triage. Triage is metadata-only — bodies are read in `draft-reply` or `surface-followups` per pick. Privacy contract from v0.13.
- Do NOT permanently delete anything. The skill uses `archive_thread` (removes INBOX label only) and the optional `trash_thread` (Gmail soft-trash, 30-day recovery). The MCP doesn't expose permanent delete and never will.
- Do NOT apply category labels in Phase 5 to the `action_required` or `waiting_on_reply` buckets. The point of those buckets is "stays visible in inbox unchanged" — labelling them adds clutter.
- Do NOT loop the user through one-by-one approval in Phase 4 by default. Default UX is "approve all" — fast. Only enter per-item edit mode when the user explicitly asks ("edit", "let me change some").
- Do NOT auto-trash anything without an explicit user request. `trash_thread` is reserved for items the user explicitly says "bin" or "delete" on. The default action for `can_archive` is **archive**, not trash.
- Do NOT run on the Cowork generic Gmail connector. It doesn't expose `list_inbox`, `archive_thread`, `label_thread`, or `unlabel_thread`. Phase 0c gates on extended Gmail access for a reason — fail honestly if it's missing rather than degrading silently.
- Do NOT classify based on body content you don't have. The classification in Phase 3 uses only `subject`, `from`, `snippet`, and `label_ids`. If the snippet doesn't give enough signal, default to `low_priority` — safer than a false `can_archive`.
- Do NOT modify `<project-dir>/.hhq-session.json` except for JWT refresh.

## Edge cases to handle gracefully

- **Empty inbox** → "Nothing to triage — all clear." Stop.
- **All items end up in `action_required`** → no archives or labels to apply. Just report: "20 items, all need your attention. Nothing automated. Open Gmail." No bulk action prompt.
- **Gmail rate limit mid-apply** → halt, report what completed + what didn't, suggest retry in a few minutes.
- **Sender lookup fails (backend down for `/api/me/contacts`)** → continue without contact context. Show senders without badges. Don't block triage on the lookup.
- **The `HHQ/Low priority` label was renamed by the user in Gmail** → the label still exists by ID. We look up by NAME each run — if no exact-name match, we'd create a duplicate. Acceptable for V1 (rare).
- **Thread's INBOX label was already removed in another Gmail tab between list and apply** → archive returns 200 (no-op). Label_thread might 404 if label was deleted — surface as a single-thread error in the summary.
- **User says "show me 200 threads"** → cap at 100 (the MCP's hard limit). Tell them "Showing the most recent 100; run again for the next batch."
