---
description: Submit (or check) a request for extended Gmail access — the prerequisite for /hhq:connect-gmail. Standalone alternative to /hhq:onboard Phase 7.5.
---

# /hhq:request-gmail-access — Submit a Gmail access request

When invoked, immediately trigger the `request-gmail-access` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles the state check, address prompt, and POST.

The user invoked this command because they want extended Gmail access without re-running full onboarding. Don't second-guess — just run the skill.

## What this command does NOT do

- Does NOT drive the OAuth handshake. That's `/hhq:connect-gmail`, runnable only after admin approval.
- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `request-gmail-access` handles all backend interaction.
- Does NOT prompt for offer / ICP / voice / signals. For first-time setup, use `/hhq:onboard`.
