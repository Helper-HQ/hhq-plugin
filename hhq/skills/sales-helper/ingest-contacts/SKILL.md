---
name: ingest-contacts
description: Imports the user's LinkedIn export into the master contact list. Triggers when the user says they've got their LinkedIn export, drops a CSV file in chat, asks to "import my contacts", or otherwise indicates a LinkedIn export is ready. Parses the Connections CSV (skipping LinkedIn's notes preamble if present), normalises rows, generates a stable per-prospect slug, and idempotently merges into `~/.hhq/sales-helper/contacts-master.csv` — new prospects added as `status: new`, existing prospects updated for mutable fields (company / position) without touching pipeline status or last_surfaced_date. Run AFTER onboard-user. If `~/.hhq/sales-helper/config.json` does not exist, route the user to onboard-user first.
---

# Ingest Contacts — Sales Helper Lite

You are importing the user's LinkedIn export into their master contact list. This is a mechanical, transactional skill — one operation, do it cleanly, report honestly. Conversational only at the edges (start and finish).

## When this skill runs

Trigger when the user:
- Says they've got their LinkedIn export ("I've got my LinkedIn export", "the export came through", "LinkedIn just emailed me", etc.)
- Drops a `.csv` file into chat
- Asks to "import my contacts", "load my LinkedIn", "ingest my contacts", or similar

Do NOT trigger if the user is mid-onboarding or asking general questions about the export process.

## Auth check (V1 stub)

Before doing anything else, run the auth check.

For V1 dogfood this is a stub that always returns `true`. When the licence backend ships (Stage 2), this becomes a signed-token verification against the locally stored token. Skill code should call the check, branch on the result, and never run skill logic if it returns `false`.

For V1: assume `auth_ok = true` and proceed. Leave a code-comment-style note in your reasoning that this is the auth seam, so future-you knows where to wire the real check.

## Pre-flight — config check

Determine the user-level config directory:
- Windows: `%USERPROFILE%\.hhq\sales-helper\`
- macOS / Linux: `~/.hhq/sales-helper/`

If `<user-config-dir>/config.json` does NOT exist:
- Tell the user they need to run onboarding first ("Looks like you haven't onboarded yet — let's do that first, it takes about 15 minutes.").
- Stop. Do not parse any CSV. Hand off to onboard-user.

If config exists, continue.

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

LinkedIn's real Connections.csv has 3+ leading lines starting with `Notes:` and disclaimer text, then a blank line, then the real CSV header. Ash's mock test file does NOT have this preamble. Handle both:

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
3. **Parse `Connected On`** from `5 Jan 2022` format (DD Mon YYYY) into ISO date `2022-01-05`. If the date is unparseable, store the raw string and flag the row.
4. **Build the slug** (see "Slug strategy" below).
5. **Choose the primary key** for dedup:
   - If `URL` is present and non-empty → use the URL (it's stable per LinkedIn member).
   - Else → use the slug.

### Slug strategy

Format: `lastname-firstname-companyhint`

- Lowercase, ASCII-fold (`ñ` → `n`, `é` → `e`, etc.), kebab-case.
- `companyhint`: take the company name, strip common corporate suffixes (`Pty Ltd`, `Pty`, `Ltd`, `Inc`, `LLC`, `GmbH`, `Limited`, `Corporation`, `Corp`, `Co`), lowercase, kebab-case, max 30 chars. Empty company → `unknown`.
- Strip non-alphanumeric (except hyphens) from each segment.

Examples:
- `Greg Coleman` at `Magnetorquer Pty Ltd` → `coleman-greg-magnetorquer`
- `John Henderson` at `EY` → `henderson-john-ey`
- `Robert Cook` at `Independent` → `cook-robert-independent`
- `Martha Alvarez` at `` (empty) → `alvarez-martha-unknown`

Collisions: if a generated slug already exists in the master for a *different* primary-key prospect, append `-2`, `-3`, etc. until unique.

## Idempotent merge with existing master

Read `<user-config-dir>/contacts-master.csv` if it exists. The master has these columns (locked schema for V1):

```
slug, first_name, last_name, url, email, company, position, connected_on, status, last_surfaced_date, notes
```

For each parsed row:

- **New** (primary key not in master): append with `status: new`, `last_surfaced_date: ""` (empty), `notes: ""`. Increment `new_count`.
- **Existing** (primary key matches): update only `company`, `position`, and `email` if they've changed. NEVER overwrite `status`, `last_surfaced_date`, or `notes`. If any field changed, increment `updated_count`. If nothing changed, increment `unchanged_count`.
- **Removed-from-export but still in master**: do NOT delete. They're still real prospects from a prior export. Leave them in place.

Write the master back atomically: write to a temp file in the same directory, then rename to `contacts-master.csv`. This avoids a half-written master if something fails mid-write.

If the master did not exist, create it with the header row above before appending.

## Report honestly

After the merge, give a brief summary in plain language. Match this shape:

> "Imported `<filename>`.
>
> - **`<new_count>` new** prospects added.
> - **`<updated_count>` updated** (company or position changed).
> - **`<unchanged_count>` already had** — left untouched.
>
> `<flagged_count>` rows had issues — `<short description e.g. 'missing company on 12 rows, unparseable date on 3'>`. They're in the master with what we could parse; you can clean them later if you want.
>
> Master now has **`<total>` prospects** at `~/.hhq/sales-helper/contacts-master.csv`.
>
> Ready when you are — when you want to start working through them, just say 'get me the next 5 prospects' or similar."

Do NOT echo the full list. Do NOT show example rows.

## Things you must NOT do

- Do NOT create per-prospect folders (`contacts/<slug>/`) here. That's `surface-next-5`'s job — only prospects we actually surface get folders.
- Do NOT do any LinkedIn enrichment, profile fetch, or web lookup. Pure CSV-to-master.
- Do NOT modify `config.json`.
- Do NOT call any MCP tools or external APIs.
- Do NOT delete prospects from the master if they're missing from a fresh export.
- Do NOT overwrite `status`, `last_surfaced_date`, or `notes` on existing prospects, ever.
- Do NOT write outside `~/.hhq/sales-helper/`.
- Do NOT attempt to parse `.zip` archives — instruct the user to extract Connections.csv first.
- Do NOT promise enrichment, deduplication beyond the slug rule, or any V2 behaviour.

## Edge cases to handle gracefully

- **Empty company** → `companyhint = unknown`, slug still works, flag for the user.
- **Non-ASCII names** (e.g. `Renée Ñuñez`) → ASCII-fold for the slug, keep original in `first_name` / `last_name` columns.
- **Duplicate full names + same company** in the same export → second one gets `-2` slug suffix.
- **CSV with embedded commas in quoted fields** (e.g. `"Smith, Jr."` as a last name) → use a real CSV parser, not naive split.
- **CRLF vs LF line endings** → handle both.
- **BOM at start of file** (Windows-saved CSVs) → strip it before parsing the header.
- **Column order varies** → match by header name, not position.
