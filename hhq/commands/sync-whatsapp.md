---
description: Walk your WhatsApp Web inbox to refresh "last messaged" data on contacts. First run goes back 6 months; ongoing runs default to 30 days. Override with `/hhq:sync-whatsapp 90d`. Header-only — no message bodies are read.
---

# /hhq:sync-whatsapp — Refresh WhatsApp DM activity

When invoked, immediately trigger the `sync-whatsapp` skill. Pass through any args verbatim — the skill parses them.

Examples of what the user might type:

- `/hhq:sync-whatsapp` → use the watermark default (6mo first run, 30d ongoing)
- `/hhq:sync-whatsapp 30d` → 30 days
- `/hhq:sync-whatsapp 90d` → 90 days
- `/hhq:sync-whatsapp 6mo` → 6 months
- `/hhq:sync-whatsapp 1y` → 365 days

The user invoked this command because they want to refresh WhatsApp DM data. Don't second-guess them, don't show a menu — just run the skill. The skill handles auth, Chrome connector check, WhatsApp Web login state, the inbox walk, the digest POST, and the watermark stamp.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `sync-whatsapp` handles all backend interaction.
- Does NOT read message bodies — header-only ingest (contact label + timestamp + direction prefix).
- Does NOT send WhatsApp messages or modify any WhatsApp state — read-only.
- Does NOT auto-create new HHQ contacts. Phones / names that don't match an existing contact are reported as `unmatched` and skipped.
