---
description: Run the daily follow-ups loop — show the queue of people waiting on a reply or with a manual reminder due, then walk through each one (live-read the thread, distil bullets, regenerate the dossier, draft a reply, push to Gmail).
---

# /hhq:followups — Run the follow-ups loop

When invoked, immediately trigger the `surface-followups` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles auto-syncing Gmail, fetching the queue, and walking the user through each pick.

The user invoked this command because they want to clear their follow-ups for the day. Don't second-guess them, don't show a menu — just run the skill.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `surface-followups` handles all backend interaction.
- Does NOT read or write `<project>/.hhq-session.json` or `<project>/.hhq-campaign.json` directly — `surface-followups` handles that.
