---
description: Walk your LinkedIn messaging inbox to refresh "last messaged" data on contacts. First run goes back 6 months; ongoing runs default to 30 days. Override with an explicit window like `/hhq:sync-linkedin-dms 90d`.
---

# /hhq:sync-linkedin-dms — Refresh LinkedIn DM activity

When invoked, immediately trigger the `sync-linkedin-dms` skill. Pass through any args verbatim — the skill parses them.

Examples of what the user might type:

- `/hhq:sync-linkedin-dms` → use the watermark default (6mo first run, 30d ongoing)
- `/hhq:sync-linkedin-dms 30d` → 30 days
- `/hhq:sync-linkedin-dms 90d` → 90 days
- `/hhq:sync-linkedin-dms 6mo` → 6 months
- `/hhq:sync-linkedin-dms 1y` → 365 days

The user invoked this command because they want to refresh LinkedIn DM data. Don't second-guess them, don't show a menu — just run the skill. The skill handles auth, Chrome connector check, the inbox walk, the digest POST, and the watermark stamp.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `sync-linkedin-dms` handles all backend interaction.
- Does NOT read message bodies — header-only ingest.
- Does NOT send messages or modify any LinkedIn state — read-only.
