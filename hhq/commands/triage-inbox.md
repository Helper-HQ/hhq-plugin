---
description: Admin Helper — pull your recent inbox, AI-categorise into action required / waiting on reply / low priority / can archive, apply labels + archives in bulk on your approval. Requires extended Gmail access.
---

# /hhq:triage-inbox — Inbox triage

When invoked, immediately trigger the `triage-inbox` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles auth check, inbox pull, classification, presentation, and bulk apply.

The user invoked this command because they want to clear their inbox. Don't second-guess — run the skill.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `triage-inbox` handles all backend interaction.
- Does NOT work without extended Gmail access. The skill checks; if missing, it routes the user to `/hhq:connect-gmail`.
- Does NOT permanently delete anything. Worst case is archive (kept in All Mail) or label.
