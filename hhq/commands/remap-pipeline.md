---
description: Edit your pipeline stages — rename, reorder, add custom stages, or delete custom stages — then re-lock. Default stages can be renamed and reordered but never deleted (sync-gmail's stage-advance keys off them). Use this when the seven HHQ defaults don't match how you actually sell.
---

# /hhq:remap-pipeline — Edit your pipeline stages

When invoked, immediately trigger the `remap-pipeline` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles everything from unlock through edit loop to re-lock.

The user invoked this command because they want to edit their pipeline. Don't second-guess them, don't show a menu, don't render help — just run the skill.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `remap-pipeline` handles all backend interaction.
- Does NOT modify contacts or any other data — only the user's pipeline stage list.
