---
description: Admin Helper — surface inbox threads where you sent the last message N+ days ago, pick one at a time, draft a follow-up in your voice, push to Gmail. Default threshold 5 days. Pass a number to override (e.g. /hhq:manage-followups 10).
---

# /hhq:manage-followups — Manage Gmail follow-ups

When invoked, immediately trigger the `manage-followups` skill, passing along anything the user typed after the command as the threshold hint (e.g. `10` → 10-day threshold). Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles auth check, queue building, render, per-pick draft, and push.

If invoked with no argument, the skill uses the default 5-day threshold.

## What this command does NOT do

- Does NOT send email. Helper HQ never sends — pushes to drafts only. The user opens Gmail and clicks send.
- Does NOT operate on Sales Helper pipeline contacts specifically. For sales conversations with the dossier + four-tier ranking, use `/hhq:followups` instead.
- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — the skill handles all backend interaction.
- Does NOT work without extended Gmail access. The skill checks; if missing, it routes the user to `/hhq:connect-gmail`.
