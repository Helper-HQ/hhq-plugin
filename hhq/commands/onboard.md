---
description: Start (or restart) Helper HQ onboarding for the Sales Helper plugin. Activates the user's licence, kicks off the LinkedIn export, and walks them through offer / ICP / signals / voice. Run this for a first-time setup, or to re-onboard with a fresh config.
---

# /hhq:onboard — Run Helper HQ onboarding

When invoked, immediately trigger the `onboard-user` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles everything from licence activation through to the warm close.

The user invoked this command because they want to onboard. Don't second-guess them, don't show a menu, don't render help — just run the skill.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `onboard-user` handles all backend interaction.
- Does NOT read or write `<project>/.hhq-session.json` or `<project>/.hhq-campaign.json` directly — `onboard-user` handles that.
