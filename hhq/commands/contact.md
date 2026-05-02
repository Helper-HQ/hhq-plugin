---
description: Show one contact's full record — dossier, recent conversation bullets across campaigns, manual touches, pipeline status. Edit the dossier or drop bullets conversationally.
---

# /hhq:contact — View and manage one contact

When invoked, immediately trigger the `contact` skill. Do NOT do any other rendering, asking, or routing — just invoke the skill, which handles fuzzy-matching the contact name, fetching the full record, and walking the user through edits.

If the user invoked the bare command without a name after it, the skill will ask: "Which contact?" Don't pre-empt that.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `contact` handles all backend interaction.
- Does NOT delete contacts (no V1 delete endpoint by design — soft-delete via `Not a fit` stage).
