---
description: Connect an already-onboarded Helper HQ account to this Cowork project. Paste your licence key, pick a campaign to pin. No offer/ICP/voice/signals questions — for first-time setup, use /hhq:onboard.
---

# /hhq:connect — Connect this project to an existing Helper HQ account

When invoked, immediately trigger the `connect` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles everything from licence-key prompt through campaign pin.

The user invoked this command because they want to connect an already-set-up account to this project. Don't second-guess them, don't show a menu, don't render help — just run the skill.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `connect` handles all backend interaction.
- Does NOT ask onboarding questions (offer, ICP, voice, signals). For first-time setup, run `/hhq:onboard` instead.
- Does NOT read or write `<project>/.hhq-session.json` or `<project>/.hhq-campaign.json` directly — `connect` handles that.
