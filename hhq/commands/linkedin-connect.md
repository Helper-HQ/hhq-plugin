---
description: Send personalised LinkedIn connection requests to contacts you're not yet connected to — works through a queue (default 10) one at a time, drafts a short note in your voice, you approve before each send. Optional source filter or limit.
---

# /hhq:linkedin-connect — Send LinkedIn connection requests

When invoked, immediately trigger the `linkedin-connect` skill. Pass through any args verbatim — the skill parses them.

Examples of what the user might type:

- `/hhq:linkedin-connect` → default: 10 contacts, all sources
- `/hhq:linkedin-connect business-cards` → 10 contacts from business card scans only
- `/hhq:linkedin-connect 20` → 20 contacts, all sources
- `/hhq:linkedin-connect 5 crm` → 5 contacts from CRM imports

The user invoked this command because they want to clear connection requests for the day. Don't second-guess them, don't show a menu — just run the skill. The skill handles auth, Chrome connector check, the queue fetch, and walks the user through each contact one at a time.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT call any backend API directly — `linkedin-connect` handles all backend interaction.
- Does NOT read or write `<project>/.hhq-session.json` directly — `linkedin-connect` handles that.
- Does NOT auto-send anything — every request goes through a per-contact yes/no in the skill.
