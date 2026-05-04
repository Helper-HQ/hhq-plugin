---
description: Sweep contacts with outstanding LinkedIn connection requests to see who's accepted, who's still pending, and which requests LinkedIn auto-expired. Detection only — never sends new requests.
---

# /hhq:linkedin-check-pending — Sweep pending LinkedIn requests

When invoked, immediately trigger the `linkedin-check-pending` skill. Pass through any args verbatim — the skill parses them.

Examples:

- `/hhq:linkedin-check-pending` → default: sweep 20 oldest pending
- `/hhq:linkedin-check-pending 30` → sweep 30 oldest pending (max 100)

The user invoked this because they want to know what's happened with the connection requests they've sent. Don't second-guess them, don't show a menu — just run the skill. The skill handles auth, Chrome connector check, the pending list fetch, and walks each profile one at a time.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT send any new connection requests — for that the user wants `/hhq:linkedin-connect`.
- Does NOT call any backend API directly — `linkedin-check-pending` handles that.
- Does NOT read or write `<project>/.hhq-session.json` directly — the skill handles that.
