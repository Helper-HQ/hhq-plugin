---
name: ingest-contacts
description: Imports the user's LinkedIn export into the Helper HQ backend. Handles both Connections.csv (people in their network) and messages.csv (their direct-message history) — detects file type by header row and routes to the correct branch. Triggers when the user says they've got their LinkedIn export, drops a CSV in chat, asks to "import my contacts" / "import my messages", or otherwise indicates a LinkedIn export is ready. Connections POST to /api/me/contacts/import (dedup by linkedin_url, returns created/updated/unchanged); messages POST to /api/me/messages/import (dedup by hash of conversation+time+sender+body, links each message to an existing contact when from_url or to_urls match). Run AFTER onboard-user. If `.hhq-auth.json` does not exist in the project folder, route the user to onboard-user first.
---

# Ingest Contacts — Sales Helper Lite

You are importing the user's LinkedIn export into their master contact list on the Helper HQ backend. This is a mechanical, transactional skill — one operation, do it cleanly, report honestly. Conversational only at the edges (start and finish).

## When this skill runs

Trigger when the user:
- Says they've got their LinkedIn export ("I've got my LinkedIn export", "the export came through", "LinkedIn just emailed me", etc.)
- Drops a `.csv` file into chat
- Asks to "import my contacts", "load my LinkedIn", "ingest my contacts", or similar

Do NOT trigger if the user is mid-onboarding or asking general questions about the export process.

## Phase 0 — Auth

Use the `mcp__ccd_directory__request_directory` tool to get the project folder. Save the returned path as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` if that tool isn't available (local Claude Code CLI).

Read `<project-dir>/.hhq-auth.json`. If missing → tell the user "Looks like you haven't onboarded yet — let's do that first, it takes about 15 minutes." Stop. Do not parse any CSV.

Parse the auth file to get `backend_url`, `jwt`, `jwt_expires_at`, `license_key`, `machine_id`.

If `jwt_expires_at` is in the future and more than 60 seconds away → use `jwt` as the bearer for the API calls below.

If `jwt_expires_at` has passed (or is within 60 seconds of expiry):
1. Refresh: `POST <backend_url>/api/refresh` with header `Authorization: Bearer <old jwt>`, empty body.
2. On 200: parse the new `token` and `expires_at`, write the updated values back to `.hhq-auth.json` (preserve all other fields), use the new token below.
3. On 401 `invalid_token` (refresh rejected): re-activate. `POST <backend_url>/api/activate` with `{license_key, machine_id}` from the saved auth file. On 200, save the new token + expires_at to `.hhq-auth.json`. On 403 `license_inactive` or `machine_limit_reached`, tell the user and stop. On any other error, tell the user the backend isn't responding and stop.

All API calls below include `Authorization: Bearer <jwt>`. Use `curl -sk` (the `-k` flag tolerates the Expose tunnel's TLS during dogfood).

Never log the JWT or licence key in chat output.

## Locate the CSV(s)

The user may have one or two files from LinkedIn — `Connections.csv` (always) and/or `messages.csv` (if they ticked Messages during onboarding Phase 2).

Files may arrive via:
1. **Attached to the current chat** — if you can see one or more `.csv` attachments, use them.
2. **Absolute paths the user provides** — e.g. `C:\Users\Brad\Downloads\Connections.csv`. Read directly.
3. **The user mentions a folder** — ask them for the full path(s).

If you cannot find a CSV after a brief check, ask:

> "Drop the CSV(s) in here, or paste the full path. If you ticked Messages too on the LinkedIn export page, drop both files."

If the user uploaded a `.zip` (the LinkedIn data archive), tell them:

> "That's the full archive — I just need `Connections.csv` (and `messages.csv` if you ticked Messages) from inside it. Extract those files and drop them in."

Do NOT attempt to unzip in V1.

## Detect file type

For each CSV the user gives you, peek the header row to determine type. Skip LinkedIn's notes preamble first (lines starting with `Notes:` or blank lines at the top).

- If the header contains `First Name` → **Connections** file → parse via the Connections branch below.
- If the header contains `CONVERSATION ID` (or both `FROM` and `CONTENT` and `FOLDER`) → **Messages** file → parse via the Messages branch below.
- Otherwise → not a recognised LinkedIn export. Tell the user honestly: "This doesn't look like a LinkedIn Connections or Messages export. If you uploaded a different file by accident, drop the right one in."

If two files are dropped together, process them sequentially: confirm both with the user, then parse + import in order (Connections first if both present, since Messages can link to existing contacts via `linkedin_url`).

## Yes/no gate before parsing

Once file types are confirmed, summarise and confirm:

> "Found `<filename1>` (Connections, ~`<rows>` rows)`<and filename2 if applicable>`. Import now? (yes / no)"

If no → stop, wait for instruction.
If yes → proceed.

---

## Connections branch

### Step C1 — Skip LinkedIn's notes preamble

LinkedIn's real Connections.csv has 3+ leading lines starting with `Notes:` and disclaimer text, then a blank line, then the real CSV header. Mock test files may not have this preamble. Handle both:

- Read lines from the top.
- The header row is the first row whose first cell is exactly `First Name`.
- Everything above the header row is discarded.

### Step C2 — Validate columns

Required columns (must be present in the header in any order):
- `First Name`
- `Last Name`
- `Company`
- `Position`
- `Connected On`

Optional columns (used if present, ignored if missing):
- `URL`
- `Email Address`

If a required column is missing, stop and tell the user:

> "This doesn't look like a LinkedIn Connections export — I'm missing the `<col>` column. If you uploaded the wrong file by accident, drop the right one in. Otherwise let me know and I'll take a look."

Do NOT attempt to guess at non-LinkedIn formats in V1.

### Step C3 — Normalise each row

For every data row:

1. **Trim whitespace** on every cell.
2. **Skip the row if** First Name AND Last Name are both empty (LinkedIn occasionally exports blank rows).
3. **Parse `Connected On`** from `5 Jan 2022` format (DD Mon YYYY) into ISO date `2022-01-05`. If unparseable, leave it `null` — the backend accepts null.
4. **Build the row object** for the API:

```json
{
  "first_name": "Greg",
  "last_name": "Smith",
  "company": "Magnetorquer Pty Ltd",
  "position": "Founder",
  "connected_on": "2022-01-05",
  "linkedin_url": "https://www.linkedin.com/in/coleman-greg",
  "email": "greg@example.com",
  "raw_csv": { ...the original row as a key/value object... }
}
```

Optional fields (`linkedin_url`, `email`, `connected_on`) should be `null` when absent. Preserve the original CSV row in `raw_csv` so the backend can re-derive things later.

The backend handles slug generation, dedup against existing contacts, and field-change detection. Do NOT compute slugs client-side — that logic now lives server-side in `ContactSlugger`.

### Step C4 — POST to /api/me/contacts/import

POST the parsed rows in a single request:

```
POST <backend_url>/api/me/contacts/import
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "contacts": [ <row 1>, <row 2>, ... ]
}
```

Use `curl -sk` via the Bash tool. For very large CSVs (5k+ rows), the JSON body can be a few MB — that's fine for V1. If you hit a request-size limit (`413 Payload Too Large`), tell the user and stop; chunked import is V2.

Expected response (HTTP 200):

```json
{
  "created": 142,
  "updated": 23,
  "unchanged": 5,
  "total": 170
}
```

---

## Messages branch

LinkedIn's `messages.csv` records every direct message the user has sent or received. We use this for two things: (a) flagging existing conversations so we don't draft cold openers for ongoing dialogue, and (b) sampling the user's outgoing voice for drafting style. Process it AFTER Connections (if both are present) so received-message senders can be linked to existing contacts.

### Step M1 — Skip preamble + read header

Same preamble-stripping logic as Step C1. The header row is the first row whose first cell is exactly `CONVERSATION ID`.

### Step M2 — Validate columns

Required columns (LinkedIn uses uppercase headers):
- `CONVERSATION ID`
- `FROM`
- `DATE`
- `CONTENT`
- `FOLDER`

Useful optional columns:
- `CONVERSATION TITLE`
- `SENDER PROFILE URL`
- `TO`
- `RECIPIENT PROFILE URLS`
- `SUBJECT`

If a required column is missing, stop and tell the user the file doesn't look like a LinkedIn Messages export.

### Step M3 — Normalise each row

For every data row:

1. **Trim whitespace** on every cell.
2. **Skip the row if** `CONTENT` is empty (LinkedIn includes some empty rows for system events).
3. **Determine direction** from `FOLDER`:
   - `FOLDER == "SENT"` → `direction = "sent"`
   - Otherwise (INBOX, ARCHIVED, etc.) → `direction = "received"`
4. **Parse `DATE`** with the format LinkedIn uses (e.g. `2024-03-14 10:30:00 UTC`). If unparseable, leave `sent_at` as the raw string — the backend tries Carbon-parse and falls back to null.
5. **Build the row object** for the API:

```json
{
  "conversation_id": "<CONVERSATION ID>",
  "direction": "sent" | "received",
  "from_name": "<FROM>",
  "from_url": "<SENDER PROFILE URL>",
  "to_names": "<TO>",
  "to_urls": "<RECIPIENT PROFILE URLS>",
  "sent_at": "<DATE>",
  "subject": "<SUBJECT or null>",
  "body": "<CONTENT>",
  "folder": "<FOLDER>",
  "raw_csv": { ...the original row as a key/value object... }
}
```

Optional fields should be `null` when absent.

### Step M4 — POST to /api/me/messages/import

```
POST <backend_url>/api/me/messages/import
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "messages": [ <row 1>, <row 2>, ... ]
}
```

Expected response (HTTP 200):

```json
{
  "created": 1247,
  "skipped": 0,
  "linked": 412,
  "unlinked": 835,
  "total": 1247
}
```

`linked` = messages where the prospect (the *other* party) already exists in the user's contacts. `unlinked` = messages with people not in their network — kept anyway for voice analysis. Re-importing the same file is safe; the backend dedupes by hash of (conversation, time, sender, body).

---

## Error handling (both branches)

- **HTTP 401** → token wasn't accepted. Run the refresh / re-activate flow from Phase 0 once, then retry. If the second try also fails, tell the user their session is broken and to re-onboard.
- **HTTP 422** → validation errors. The error body has `errors.contacts.{idx}.{field}` or `errors.messages.{idx}.{field}`. Tell the user honestly: "N rows had problems and weren't imported (e.g. row 12: missing direction). The rest went through if any." Show the first 2-3 indices and fields. Don't dump the whole error.
- **HTTP 5xx / network** → tell the user the backend's not responding, suggest retry in a moment. Don't keep retrying automatically.

## Report honestly

After a successful import, give a brief summary in plain language.

**Connections-only result:**

> "Imported `<filename>`.
>
> - **`<created>` new** prospects added.
> - **`<updated>` updated** (company, position, or other CSV-derived fields changed).
> - **`<unchanged>` already had** — nothing changed.
>
> Total in your master now: **`<total imported>` from this file** (plus anything from earlier imports).
>
> Ready when you are — when you want to start working through them, just say 'get me the next 5 prospects' or similar."

**Messages-only result:**

> "Imported `<filename>`.
>
> - **`<created>` new** messages stored.
> - **`<linked>`** matched to people already in your contacts.
> - **`<unlinked>`** are with people outside your connections (kept anyway — useful for voice).
> - **`<skipped>`** were duplicates of messages already imported.
>
> Your message history is now part of the picture I'll use when I draft openers — I'll skip cold-opener drafts for prospects you've already been chatting with."

**Both files imported in one go:** show both blocks back-to-back, brief.

Do NOT echo full lists. Do NOT show example rows.

## Things you must NOT do

- Do NOT write a local `contacts-master.csv` or any contact data to disk. The backend is the master.
- Do NOT compute slugs, do dedup logic, or merge state client-side. The backend does all of that.
- Do NOT modify `.hhq-auth.json` except to update `jwt` and `jwt_expires_at` after a refresh / re-activate.
- Do NOT call any other API endpoint or external service.
- Do NOT delete prospects from the master if they're missing from a fresh export — the backend keeps everything by design (you couldn't delete them even if you tried; the import endpoint is upsert-only).
- Do NOT attempt to parse `.zip` archives — instruct the user to extract `Connections.csv` (and `messages.csv` if present) first.
- Do NOT mix rows from Connections and Messages into the same POST. Each file goes to its own endpoint with its own row schema.
- Do NOT try to derive `direction` from anything other than `FOLDER` (LinkedIn's own classification). Don't guess from FROM/TO names.
- Do NOT promise enrichment, deduplication beyond what the backend does, or any V2 behaviour.
- Do NOT log the licence key or JWT in chat.

## Edge cases to handle gracefully

- **Empty company** → send `null` for company. The backend slugs from name in that case.
- **Non-ASCII names** (e.g. `Renée Ñuñez`) → send the original Unicode. The backend handles slugging.
- **CSV with embedded commas in quoted fields** (e.g. `"Smith, Jr."` as a last name) → use a real CSV parser, not naive split.
- **CRLF vs LF line endings** → handle both.
- **BOM at start of file** (Windows-saved CSVs) → strip it before parsing the header.
- **Column order varies** → match by header name, not position.
- **5000+ rows** → fine for one POST in V1. The backend processes the whole batch in a single transaction.
