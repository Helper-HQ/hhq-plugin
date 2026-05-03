---
description: Admin Helper — set up the 5 default Helper HQ Gmail labels (To Do / Awaiting Reply / FYI / Notifications / Newsletters) on your Gmail account. One-time setup; required before triage-inbox and the background auto-rules can apply anything. Auto-runs after /hhq:connect-gmail succeeds.
---

# /hhq:setup-gmail-labels — Set up Helper HQ Gmail labels

When invoked, immediately trigger the `setup-gmail-labels` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles auth check, current-state lookup, label proposal, customisation, and the API call to create labels in Gmail.

The user invoked this command because they want to set up (or adjust) their HHQ label scheme.

## What this command does NOT do

- Does NOT create custom labels outside the canonical 5 (To Do / Awaiting Reply / FYI / Notifications / Newsletters). Custom labels are V1.1+.
- Does NOT delete existing Gmail labels — only creates new ones under `HHQ/`.
- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `setup-gmail-labels` handles all backend interaction.
- Does NOT work without extended Gmail access. The skill checks; if missing, it routes the user to `/hhq:connect-gmail`.
