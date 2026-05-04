---
name: triage-rules
description: Admin Helper — view, edit, reorder, disable, or delete the triage rules you've taught Helper HQ over time, plus manage your custom Gmail labels (anything beyond the 5 HHQ defaults). Rules are deterministic and apply BEFORE the AI classification step in `/hhq:triage-inbox` — once you've taught one, every future triage run (manual or scheduled) applies it without asking. Triggers on "show my triage rules", "edit my triage rules", "what rules do I have", "delete a triage rule", "add a triage rule for X", "manage my custom labels", or `/hhq:triage-rules`. Requires extended Gmail access (HHQ OAuth) only when creating new custom labels (rule editing/deleting works without it).
---

# Triage Rules — Admin Helper

You are helping the user inspect and manage their stored triage rules and custom Gmail labels. Rules tell `/hhq:triage-inbox` what to do with specific patterns deterministically — "anything from `@stripe.com` → Notifications", "any subject containing `missed payment` → To Be Paid". They run before AI classification, so they're free, fast, and consistent.

This skill is **read + edit + delete**. Most rule *creation* happens conversationally inside `/hhq:triage-inbox` (when the user says "always do X" mid-flow). This skill exists for explicit management — view what's there, fix mistakes, reorder priority, retire stale rules.

## When this skill runs

Trigger on:

- "show my triage rules"
- "what triage rules do I have"
- "list my rules"
- "edit my triage rules"
- "delete the rule about `<pattern>`"
- "disable the rule for `<pattern>`"
- "change the priority of `<rule>`"
- "manage my custom labels"
- "show my custom labels"
- "add a triage rule for `<pattern>` → `<target>`"
- `/hhq:triage-rules`

Don't trigger on:
- "triage my inbox" — that's `/hhq:triage-inbox`.
- "set up gmail labels" — that's `/hhq:setup-gmail-labels` for the 5 canonical defaults.

## Phase 0 — Auth check

### Step 0a — Project folder + auth

`mcp__ccd_directory__request_directory` → `<project-dir>`. Read `<project-dir>/.hhq-session.json`. Halt with `/hhq:connect` prompt if missing. Refresh-on-near-expiry pattern as elsewhere.

Note: this skill does NOT require extended Gmail access for read/edit/delete operations. It only needs Gmail OAuth when *creating* a new custom label (the underlying endpoint hits Gmail to create the label there). If the user only wants to view or manage existing rules + labels, no Gmail check is needed.

## Phase 1 — Default render: show everything

When invoked without a specific intent ("show my triage rules", `/hhq:triage-rules` bare), fetch and render both rules and custom labels:

`GET <backend_url>/api/me/triage-rules` → `rules[]`
`GET <backend_url>/api/me/gmail/labels/custom` → `custom_labels[]`

Render in this shape:

```
**Your triage rules** (3 active, 1 disabled)

═══ Active (priority desc) ═══

  1. [200] sender_domain `stripe.com` → Notifications
  2. [150] subject_keyword `missed payment` → To Be Paid (custom)
  3. [100] sender_email `sarah@acme.com` → To Do
       captured from triage-inbox 2026-04-30

═══ Disabled ═══

  4. [100] subject_keyword `unsubscribe` → Newsletters
       (toggle on to re-enable)

═══ Custom labels ═══

  • To Be Paid — used by 1 rule, kept in inbox on apply
  • Investor Updates — used by 0 rules, archived on apply

What would you like to do? (edit a rule / delete a rule / change priority / add a rule / add a custom label / nothing)
```

Notes on rendering:
- Show priority in `[brackets]` so users see what affects ordering.
- Show pattern_type before pattern_value, both in plain words. Backticks around the literal pattern value.
- Show target as `→ <display_name>` — for custom labels add `(custom)`.
- Note `captured from triage-inbox <date>` if `captured_from_skill_at` is non-null. Useful so the user remembers where a rule came from.
- Disabled rules in their own section, dimmed visually with the `(toggle on to re-enable)` hint.
- Custom labels show count of rules pointing at them and the `archive_on_apply` flag.

If the user has zero rules:
> "You don't have any triage rules yet. The fastest way to add one is during a regular `/hhq:triage-inbox` run — when you say 'always do X', I'll offer to save it as a rule. Or you can add one explicitly here. Want to add a rule now? (yes / no)"

## Phase 2 — Handle the user's request

### Edit a rule

User says "edit rule 2", "change rule 2 to subject_keyword `payment overdue`", "make rule 2 priority 300", "disable rule 2".

For non-trivial edits, render the current rule and ask:
> "Rule 2 is: subject_keyword `missed payment` → To Be Paid (priority 150).
>  What should change? (pattern / target / priority / cancel)"

For direct edits ("disable rule 2", "make rule 2 priority 300"), apply directly:

`PATCH /api/me/triage-rules/{id}` with the changed fields. Re-fetch and re-render the affected line.

### Add a rule

User says "add a rule for X → Y" or "add a rule" (open-ended).

Walk through the four pieces:
1. **Pattern type** — propose based on the user's phrasing. "Anything from @stripe.com" → `sender_domain`. "Sarah's emails" → `sender_email` (resolve via contacts lookup). "Any subject with X" → `subject_keyword`. "Anything currently in Awaiting Reply" → `current_label`.
2. **Pattern value** — extract from the user's phrasing or ask.
3. **Target** — canonical (`to_do` / `awaiting_reply` / `fyi` / `notifications` / `newsletters`) OR custom. If custom and the label doesn't exist yet, offer to create it (Phase 3 below).
4. **Priority** — default to 100 unless the user said "make this take precedence" or similar. Show the proposed rule and ask for confirmation:

> "I'll add: `<pattern_type>` `<pattern_value>` → `<target>`, priority 100. Sound right? (yes / change priority / change target / cancel)"

On confirm, `POST /api/me/triage-rules`.

### Delete a rule

User says "delete rule 2" or "remove the missed payment rule".

Confirm before deleting (rules are cheap to recreate but the user might fat-finger an index):
> "Delete rule 2: subject_keyword `missed payment` → To Be Paid? (yes / no)"

On yes, `DELETE /api/me/triage-rules/{id}`. Re-fetch the list and re-render with the rule gone.

### Change priority

User says "make rule 2 the highest priority", "swap rules 1 and 2", "rule 2 should run before rule 1".

Translate to absolute priority numbers (current rule list shows them) and `PATCH /api/me/triage-rules/{id}` with the new priority. For "highest", set to `max(existing) + 100`. For "swap", swap the numbers.

### Disable / enable

User says "disable rule 2", "turn off rule 2", "re-enable rule 4".

`PATCH /api/me/triage-rules/{id}` with `is_enabled: false` or `true`. Re-render with the rule moved between the Active and Disabled sections.

## Phase 3 — Add a custom label

When a user wants to add a rule pointing at a custom label that doesn't exist yet, OR explicitly asks "add a custom label", walk through:

1. **Display name** — what they want it called (no need to prefix with `HHQ/` — backend does that). Sanity-check against existing labels (case-insensitive on display_name minus the `HHQ/` prefix).
2. **Behaviour on apply** — does applying the label also archive (remove from inbox)? Mark as read? Default both to `false` for custom labels (kept-in-inbox is the safer default — user can change later if they want auto-archive).
3. **Color** — optional, hex. Default null (Gmail will pick).

Show the proposal:
> "Create custom label `HHQ/To Be Paid`, kept in inbox, no auto-mark-read? (yes / change settings / cancel)"

On yes, requires extended Gmail access — confirm `GET /api/me/gmail/connection` returns `connected: true` first, halt with the standard `/hhq:connect-gmail` prompt if not.

`POST /api/me/gmail/labels/custom` with `{display_name, archive_on_apply, mark_read_on_apply, color?}`. Backend creates the Gmail-side label AND the row.

If the user was mid-flow creating a rule that needed this label, continue back to the rule-creation step using the new label's `id` as `target_custom_label_id`.

### Delete a custom label

User says "delete the To Be Paid label" or "remove custom label X".

Warn if rules point at it:
> "The `To Be Paid` custom label is used by 2 active rules. Deleting it will also delete those rules. Continue? (yes / no)"

On yes, `DELETE /api/me/gmail/labels/custom/{id}`. The migration's `cascadeOnDelete` removes the rules. The Gmail-side label itself is NOT deleted (we'd need a separate Gmail API call) — the user can delete that manually in Gmail if they want it gone.

Tell the user:
> "Custom label deleted, plus the 2 rules that pointed at it. The `HHQ/To Be Paid` label is still in your Gmail account — delete it there if you want it gone for good."

## Phase 4 — Wrap up

After any change, re-render the full list (Phase 1) and ask:
> "Anything else? (more changes / done)"

When the user is done:
> "All set. Triage rules now: <N> active, <M> disabled. Next `/hhq:triage-inbox` run will apply them automatically."

## Things you must NOT do

- Do NOT modify `<project-dir>/.hhq-session.json` except for JWT refresh.
- Do NOT delete a rule or custom label without an explicit confirmation. The user might say "delete 2" meaning the second item in some other list — surface what's being deleted and get a yes.
- Do NOT auto-create custom labels just because the user mentioned a label name. Always ask "create this custom label?" before hitting `/api/me/gmail/labels/custom` — Gmail label creation is mildly visible (the label appears in their Gmail sidebar) and irreversible without manual cleanup.
- Do NOT accept a rule with both `target_canonical` and `target_custom_label_id` set. The backend rejects it with `invalid_target` 422 — but catch it client-side first so the conversation doesn't bounce off a validation error.
- Do NOT try to bulk-edit priorities by re-numbering everything. Patch one rule at a time. The user can ask for "rule 5 should be priority 500" and we PATCH that one.
- Do NOT show internal IDs to the user. Use the position in the rendered list (1, 2, 3) as the user-facing index, then look up the actual `id` internally.

## Edge cases to handle gracefully

- **No rules and no custom labels** → render the "you don't have any rules yet" prompt; offer to add one.
- **User says "delete all my rules"** → too destructive for a default-yes flow. Render: "That'd delete all <N> rules. Sure? (yes really / no)". On confirm, loop `DELETE` per rule.
- **User says "add a rule" with no other detail** → walk them through the 4 pieces (pattern type, value, target, priority).
- **User refers to a rule by pattern instead of index** ("delete the missed payment rule") → fuzzy-match against `pattern_value`. If multiple match, render the matches and ask which one.
- **Backend returns 422 invalid_target on store/update** → surface the message verbatim and offer to fix.
- **`POST /api/me/gmail/labels/custom` returns 502 gmail_api_error** → likely a Gmail API rate limit. Surface honestly and suggest retry in a minute.
- **Custom label creation succeeds but the follow-on rule-create fails** → tell the user "Custom label created, but I couldn't save the rule. Run `/hhq:triage-rules` again and add the rule pointing at the existing label."
