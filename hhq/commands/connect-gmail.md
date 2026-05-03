---
description: Complete the OAuth handshake to connect your Gmail to Helper HQ's own OAuth app. Use this after you receive the "your extended Gmail access is approved" email. Drives the Google consent screen and verifies the connection landed.
---

# /hhq:connect-gmail — Finish extended Gmail connection

When invoked, immediately trigger the `connect-gmail` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles the state check, OAuth begin, browser handoff, and connection poll.

The user invoked this command because they want to complete extended Gmail access. They've already (a) submitted a request via `/hhq:onboard` Phase 7.5 AND (b) received an approval email. Don't second-guess — just run the skill.

## What this command does NOT do

- Does NOT submit a NEW access request. For that, run `/hhq:onboard` and complete the extended Gmail step.
- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `connect-gmail` handles all backend interaction.
