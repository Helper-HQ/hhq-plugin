---
name: review-imports
description: Walks the user through the import review queue — incoming contact rows that fuzzy-matched an existing contact (same name + company, no email or LinkedIn URL to confirm identity) and were held back from auto-import. For each pending item, shows the existing contact and the incoming row side by side with the match score and reason, then asks the user to merge (confirm same person, fill blanks on the existing contact), create a new contact (different person who happens to share a name + company), or skip (decide later). Triggers when the user says "review my imports", "show review queue", "any contacts to review", "I have N to review" (after an ingest summary), or similar. Run AFTER ingest-contacts has reported review_count > 0. The queue is mostly populated by business card scans and CRM/spreadsheet imports — LinkedIn-only users will rarely see anything here. Confirms via PATCH to /api/me/import-review-queue/{id} with one of confirmed / rejected / skipped.
---

# Review Imports — Sales Helper Lite

You are walking the user through pending fuzzy-match items in their import review queue. For each item, the backend has a row that *looks like* an existing contact (same name + company, similarity ≥80%) but couldn't confirm identity via email or LinkedIn URL. The user decides what each one is.

This is a conversational, one-at-a-time skill. Three resolutions per item:

- **Confirm (merge)** — same person. Fill blank fields on the existing contact from the incoming row. Don't overwrite anything non-empty.
- **Reject (create new)** — different people who happen to share a name + company (twins, common names at large firms). Create a new contact from the row.
- **Skip** — leave pending, decide later. Resurfaces on the next review.

## When this skill runs

Trigger when the user says any variant of:

- "review my imports"
- "show review queue"
- "any contacts to review"
- "let's review the queue"
- "I have N to review" (often after seeing an ingest summary mentioning review_count)
- "deal with the pending imports"

Also trigger inline if `ingest-contacts` reports `review_count > 0` AND the user replies with intent to review now ("yes review them now", "let's clear the queue").

## Phase 0 — Auth

Use `mcp__ccd_directory__request_directory` to get the project folder. Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` in local Claude Code CLI.

Read `~/.hhq/machine.json`.

- **Found** → parse `backend_url`, `license_key`, `machine_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but `<project-dir>/.hhq-auth.json` exists** → legacy file from before v0.10. Migrate inline: `mkdir -p ~/.hhq`, copy → `~/.hhq/machine.json`, delete legacy. Continue.
- **Not found, no legacy file** → "No auth — say 'set me up' to onboard." Stop.

Refresh / re-activate if expired (same flow as other skills; save updates to `~/.hhq/machine.json`). Use `Authorization: Bearer <jwt>` and `curl -sk` for all calls. Never log the JWT or licence key.

## Phase 1 — Fetch pending items

```
GET <backend_url>/api/me/import-review-queue
```

Response:

```json
{
  "items": [
    {
      "id": 47,
      "source": "business_card",
      "raw_data": { ...the incoming row that was held back... },
      "match_score": 95.5,
      "match_reason": "exact name match, different company",
      "status": "pending",
      "created_at": "2026-04-28T10:00:00+00:00",
      "matched_contact": {
        "id": 12,
        "slug": "greg-smith-magnetorquer",
        "first_name": "Greg",
        "last_name": "Smith",
        "company": "Magnetorquer",
        "position": "VP Engineering",
        "email": "greg@magnetorquer.com",
        "phone": null,
        "website": null,
        "linkedin_url": "https://linkedin.com/in/greg",
        "source": "linkedin_csv",
        "pipeline_stage_id": 3,
        "last_messaged_at": null,
        "message_count": 0
      }
    }
  ],
  "total": 1
}
```

If `total === 0`:

> "Review queue is empty — nothing to look at right now. Run an import (`ingest-contacts`) and I'll flag any fuzzy matches for review here."

Stop.

If `total > 0`:

> "You have **`<total>` to review**. I'll walk you through them one at a time — for each one I'll show what's already in your contacts and what the incoming row says, and you decide whether they're the same person.
>
> Ready? (yes / no)"

If no → stop. If yes → into the loop.

## Phase 2 — Per-item loop

For each item in the response (sorted by match_score descending — high-confidence matches first), show side-by-side and ask. Format:

```
─── Review 1 of <total> — <source>, score <match_score>% ───
Match reason: <match_reason>

EXISTING CONTACT:
  <first_name> <last_name> — <position>, <company>
  email: <email>
  phone: <phone>
  linkedin: <linkedin_url>
  website: <website>
  source: <source>
  last messaged: <last_messaged_at or "never">

INCOMING ROW:
  <first_name> <last_name> — <position>, <company>
  email: <email or "—">
  phone: <phone or "—">
  linkedin: <linkedin_url or "—">
  website: <website or "—">
  source: <source>
  notes: <notes or "—">

Same person, or different?
  • say 'merge' / 'same' — fill blanks on the existing contact from the incoming row
  • say 'new' / 'different' — create a separate contact (sometimes two people genuinely share a name + company)
  • say 'skip' — leave pending, decide later
  • say 'stop' — pause the review, the rest stay pending
```

Use `—` for null fields so the user sees the gap clearly. Don't show the contact `id` or queue `id` in the UI — they're plumbing.

## Phase 3 — Resolve

Based on the user's response:

**`merge` / `same` / `confirm` / `yes same`:**

```
PATCH <backend_url>/api/me/import-review-queue/<id>
Authorization: Bearer <jwt>
Content-Type: application/json

{ "status": "confirmed" }
```

Expected response: `{ "resolution": "confirmed", "merged_into_contact_id": 12, "merged_into_slug": "greg-smith-magnetorquer" }`. Tell the user briefly: "Merged into Greg Smith — filled in missing fields." Move to next item.

**`new` / `different` / `reject` / `not the same`:**

```
PATCH <backend_url>/api/me/import-review-queue/<id>
{ "status": "rejected" }
```

Expected response: `{ "resolution": "rejected", "new_contact_id": 87, "new_contact_slug": "greg-smith-magnetorquer-2" }`. Tell the user briefly: "Created as a separate contact." Move to next item.

**`skip` / `later` / `not sure`:**

```
PATCH <backend_url>/api/me/import-review-queue/<id>
{ "status": "skipped" }
```

Expected response: `{ "resolution": "skipped" }`. Tell the user briefly: "Skipped — I'll surface this again next review." Move to next item.

**`stop` / `pause` / `let's come back`:**

End the loop early. Do NOT PATCH anything for items not yet shown. Tell the user how many remain, then stop:

> "Paused — `<remaining>` still pending. Say 'review my imports' again when you're ready."

## Phase 4 — Summary

After the loop completes (or the user stopped), give a brief tally:

> "Review done.
> - `<merged>` merged into existing contacts
> - `<created>` created as separate contacts
> - `<skipped>` skipped (still pending)
> *(if any remain after a 'stop')* `<remaining>` still pending — run 'review my imports' again any time."

## Error handling

- **HTTP 401** → run the refresh / re-activate flow once, retry. If it fails again, tell the user to re-onboard. Stop.
- **HTTP 404 `queue_item_not_found`** → the item was deleted between listing and resolving (rare — could happen if the user is reviewing in two places at once). Tell the user briefly, skip to the next item.
- **HTTP 422 `queue_item_already_resolved`** → another session resolved this same item. Skip to the next item silently.
- **HTTP 5xx / network** → tell the user the backend's not responding. Mid-loop, stop and report what was resolved so far so the user knows where they left off.

## Things you must NOT do

- Do NOT batch-resolve items. One at a time, with the user's confirmation per item. The point of the queue is the user's judgment — automating it would defeat the purpose.
- Do NOT show or echo internal IDs (queue id, contact id) in the prose UI. Show slugs / names only.
- Do NOT modify `~/.hhq/machine.json` except to update `jwt` / `jwt_expires_at` after a refresh / re-activate.
- Do NOT show internal scoring math beyond the single `match_score` percentage and `match_reason` text the backend sends.
- Do NOT log the JWT or licence key.
- Do NOT pre-fetch all items and PATCH them eagerly — fetch the list once at the top, walk through it sequentially. If a 422-already-resolved or 404-not-found pops up, the user is reviewing in another session; just skip it.

## Edge cases to handle gracefully

- **User says nothing definitive** ("hmm I'm not sure") → treat as `skip`. Don't grind on it; the queue is meant to be revisitable.
- **User wants to edit the matched contact rather than merge or skip** ("the existing one has the wrong email — fix that first") → handle gracefully: tell them you can update fields via the contact detail PUT separately, but the queue item itself only takes confirm/reject/skip. Skip it for now and they can revisit after editing.
- **A queue item's `matched_contact` is null** in the GET response — happens if the matched contact was deleted after the queue item was created. The backend's `confirmed` resolution path detects this and falls through to `rejected` (creates a new contact). Show the user the row data only and ask "no existing match anymore — import as new?" — yes → confirm (which the backend treats as create-new), no → skip.
- **Very large queue (hundreds of items)** — go through them, but offer a checkpoint after every ~20: "Halfway through — keep going? (yes / pause)". Don't make the user grind on a hundred-item review without a break.
- **User asks "what does merge actually change"** → tell them: "Merge fills any blank fields on the existing contact from the incoming row. It never overwrites a field that already has data — your manual edits are safe. If you want to overwrite a specific field, edit the contact directly afterwards."
