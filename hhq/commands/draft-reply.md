---
description: Admin Helper — draft a reply to one Gmail thread in your voice, pushed as a draft to Gmail for you to review and send. Requires extended Gmail access. Pass an optional reference (sender name, topic, Gmail URL, or thread ID).
---

# /hhq:draft-reply — Draft a Gmail reply

When invoked, immediately trigger the `draft-reply` skill, passing along anything the user typed after the command as the thread reference. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles auth check, thread identification, drafting, and the push.

If the user invoked `/hhq:draft-reply Sarah's pricing email`, treat "Sarah's pricing email" as the thread reference and run the skill with that hint.

If the user invoked `/hhq:draft-reply` with no argument, run the skill and let it prompt for the reference.

## What this command does NOT do

- Does NOT send the email. Helper HQ never sends — always pushes to drafts. The user opens Gmail and clicks send.
- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `draft-reply` handles all backend interaction.
- Does NOT work without extended Gmail access. The skill checks; if missing, it routes the user to `/hhq:connect-gmail`.
