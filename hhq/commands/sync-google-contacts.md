---
description: Backfill phone numbers onto your contacts from Google Contacts. Backend-scheduled daily, but you can run this on demand if you just added a contact on your phone and want it picked up now.
---

# /hhq:sync-google-contacts — Refresh phones from Google Contacts

When invoked, immediately trigger the `sync-google-contacts` skill. No args.

The user invoked this because they want a fresh Google Contacts pull right now (the normal cadence is a daily background job). Don't second-guess, don't show a menu — just run the skill. The skill handles auth, the backend trigger, polling, and the result summary.

## What this command does NOT do

- Does NOT render help (use `/hhq:help` for that).
- Does NOT walk Google Contacts itself — the backend does the People API work using your already-granted Google OAuth.
- Does NOT create new HHQ contacts. **Backfill-only** — only existing contacts get phone numbers / blank-field updates.
- Does NOT read message bodies, addresses, photos, or anything else from Google. Names + emails + phones + company only.
