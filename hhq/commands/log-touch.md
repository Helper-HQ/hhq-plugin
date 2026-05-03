---
description: Quick capture for offline interactions — phone calls, meetings, in-person chats, voicemails. Logs the touch, optionally sets a follow-up reminder, and feeds the dossier.
---

# /hhq:log-touch — Log an offline interaction

When invoked, immediately trigger the `log-touch` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which parses the user's free-form sentence, fuzzy-matches the contact, and POSTs the touch.

If the user invoked the bare command without any sentence after it, the skill will ask: "What did you do, with whom?" Don't pre-empt that.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `log-touch` handles all backend interaction.
