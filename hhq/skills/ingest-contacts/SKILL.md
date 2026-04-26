---
name: ingest-contacts
description: Imports the user's LinkedIn export into the master contact list on the Helper HQ backend. Triggers when the user says they've got their LinkedIn export, drops a CSV file in chat, asks to "import my contacts", or otherwise indicates a LinkedIn export is ready. Parses the Connections CSV (skipping LinkedIn's notes preamble if present), normalises rows, and POSTs them to /api/me/contacts/import — the backend dedupes by linkedin_url, preserves user-added notes/research/messages on existing prospects, and returns counts. Run AFTER onboard-user. If `.hhq-auth.json` does not exist in the project folder, route the user to onboard-user first.
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

## Locate the CSV

The file may have arrived via one of:
1. **Attached to the current chat** — if you can see a `.csv` attachment, use it.
2. **An absolute path the user provides** — e.g. `C:\Users\Brad\Downloads\Connections.csv`. Read directly.
3. **The user mentions a folder** — ask them for the full path to the CSV file.

If you cannot find a CSV after a brief check, ask:

> "Drop the CSV in here, or paste the full path to the file."

If the user uploaded a `.zip` (the LinkedIn data archive), tell them:

> "That's the full archive — I just need the `Connections.csv` file from inside it. Extract that one file and drop it in."

Do NOT attempt to unzip in V1.

## Yes/no gate before parsing

Once you have the CSV, confirm before you ingest:

> "Found `<filename>` with roughly `<line_count>` rows. Want me to import it now? (yes / no)"

(Use a quick line count — don't parse the full file just to count rows.)

If no → stop, wait for instruction.
If yes → proceed.

## Parse

### Step 1 — Skip LinkedIn's notes preamble

LinkedIn's real Connections.csv has 3+ leading lines starting with `Notes:` and disclaimer text, then a blank line, then the real CSV header. Mock test files may not have this preamble. Handle both:

- Read lines from the top.
- The header row is the first row whose first cell is exactly `First Name`.
- Everything above the header row is discarded.

### Step 2 — Validate columns

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

### Step 3 — Normalise each row

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

## Send to the backend

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

Error handling:
- **HTTP 401** → token wasn't accepted. Run the refresh / re-activate flow from Phase 0 once, then retry the import. If the second try also fails, tell the user their session is broken and to re-onboard.
- **HTTP 422** → one or more rows failed validation. The error body has `errors.contacts.{idx}.{field}`. Tell the user honestly: "N rows had problems and weren't imported (e.g. row 12: missing first_name). The rest went through if any." Show the first 2-3 indices and field issues. Don't dump the whole error.
- **HTTP 5xx / network** → tell the user the backend's not responding, suggest retry in a moment. Don't keep retrying automatically.

## Report honestly

After a successful import, give a brief summary in plain language. Match this shape:

> "Imported `<filename>`.
>
> - **`<created>` new** prospects added.
> - **`<updated>` updated** (company, position, or other CSV-derived fields changed).
> - **`<unchanged>` already had** — nothing changed.
>
> Total in your master now: **`<total imported>` from this file** (plus anything from earlier imports).
>
> Ready when you are — when you want to start working through them, just say 'get me the next 5 prospects' or similar."

Do NOT echo the full list. Do NOT show example rows.

## Things you must NOT do

- Do NOT write a local `contacts-master.csv` or any contact data to disk. The backend is the master.
- Do NOT compute slugs, do dedup logic, or merge state client-side. The backend does all of that.
- Do NOT modify `.hhq-auth.json` except to update `jwt` and `jwt_expires_at` after a refresh / re-activate.
- Do NOT call any other API endpoint or external service.
- Do NOT delete prospects from the master if they're missing from a fresh export — the backend keeps everything by design (you couldn't delete them even if you tried; the import endpoint is upsert-only).
- Do NOT attempt to parse `.zip` archives — instruct the user to extract Connections.csv first.
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
