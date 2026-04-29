---
name: ingest-contacts
description: Imports the user's contacts into the Helper HQ backend from any of four file-based sources — (1) LinkedIn export Connections.csv + messages.csv, (2) generic spreadsheets (.xlsx, .csv, .xls) with arbitrary column layouts, (3) CRM exports from HubSpot, Salesforce, Pipedrive, etc., (4) business card scans (PDF or image — jpg, png, heic, webp — one or many cards per file). Detects file type and routes to the correct branch. For LinkedIn Connections.csv, parses + POSTs to /api/me/contacts/import (dedup by email → linkedin_url → fuzzy → create, returns created/updated/unchanged/review_count). For LinkedIn messages.csv, digests client-side into {linkedin_url, last_messaged_at, message_count} per conversation partner (message bodies NOT persisted) and POSTs to /api/me/messages/import. For spreadsheets/CRM exports, Claude maps the file's columns onto the contacts schema, asks the user to confirm the mapping, optionally maps stage labels onto Helper HQ pipeline stages, then POSTs with source=spreadsheet or crm_csv. For business card scans, fetches the server-controlled extract_business_card vision prompt from /api/mcp/prompts, runs a vision pass on the image/PDF returning a per-card array with status enum (ok/partial/no_individual_name/event_card/blank_or_back), routes flagged cards through user review or auto-skip per the brief, then POSTs ok/partial/company-as-contact rows with source=business_card. Triggers when the user drops a CSV, xlsx, PDF, or image in chat, says they've got their LinkedIn export, says "import my contacts", "import my spreadsheet", "I have a CRM export", "scan my cards", "import my business cards", "ingest from HubSpot/Salesforce/Pipedrive", or similar. Run AFTER onboard-user. If `.hhq-auth.json` does not exist in the project folder, route the user to onboard-user first.
---

# Ingest Contacts — Sales Helper Lite

You are importing the user's contacts into their master contact list on the Helper HQ backend from a file the user dropped in chat. The skill routes across four branches based on what the file is:

- **LinkedIn Connections** (Connections.csv) — fixed shape, well-known headers
- **LinkedIn Messages** (messages.csv) — digest only, never store bodies
- **Spreadsheet / CRM export** (xlsx, csv, xls) — arbitrary columns, Claude maps them with the user's confirmation
- **Business card scan** (PDF or image — jpg, png, heic, webp) — vision extraction of one or many cards per file, per-card user review

Mechanical, transactional skill — one operation per file, do it cleanly, report honestly. Conversational only at the edges (file detection, mapping/extraction confirmation, finish).

## When this skill runs

Trigger when the user:
- Drops a `.csv`, `.xlsx`, `.xls`, `.pdf`, `.jpg`, `.jpeg`, `.png`, `.heic`, or `.webp` file into chat
- Says they've got their LinkedIn export ("I've got my LinkedIn export", "the export came through", "LinkedIn just emailed me")
- Asks to "import my contacts", "load my LinkedIn", "ingest my contacts"
- Asks to "import my spreadsheet", "I have a CRM export", "ingest from HubSpot / Salesforce / Pipedrive", "import my customer list"
- Asks to "scan my cards", "import my business cards", "I just got back from a conference, here are the cards"

Do NOT trigger if the user is mid-onboarding (let `onboard-user` route into this skill inline) or asking general questions about the export process.

## Phase 0 — Auth

Use `mcp__ccd_directory__request_directory` (no arguments) to get the project folder. Save as `<project-dir>`. Fall back to `~/.hhq/sales-helper/` if the tool isn't registered (rare CLI case).

Read `<project-dir>/.hhq-session.json` (per-project session auth, v0.11+).

- **Found** → parse `backend_url`, `license_key`, `machine_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but legacy `<project-dir>/.hhq-auth.json` exists** → migrate by renaming to `.hhq-session.json`. Continue.
- **Neither found** → "This project isn't connected to Helper HQ — say `/hhq:connect` to link it (or `/hhq:onboard` if you're brand-new)." Stop. Do not parse any CSV.

If `jwt_expires_at` is in the future and more than 60 seconds away → use `jwt` as the bearer for API calls.

If `jwt_expires_at` has passed (or is within 60 seconds of expiry):
1. Refresh: `POST <backend_url>/api/refresh` with header `Authorization: Bearer <old jwt>`. On 200, save the new token + expires_at to `.hhq-session.json` (preserving other fields).
2. On 401, the session may have been released — tell the user: "Your session was released — say `/hhq:connect` to re-link this project." Stop. Do NOT auto-re-activate.

All API calls below include `Authorization: Bearer <jwt>`. Use `curl -sk`.

Never log the JWT or licence key in chat output.

Note: contacts are **user-level master data** shared across all campaigns — this skill imports into the master pool. Per-campaign state (status, last_surfaced_at, research, messages) is created lazily as surface-next-5 / research-and-draft run within a specific campaign.

## Locate the file(s)

The user can drop any of three sources:

- **LinkedIn Connections.csv** + **messages.csv** — extracted from LinkedIn's `.zip` archive (they no longer let users pick individual files; the user extracts the two we want and ignores the rest).
- **Generic spreadsheet** — `.xlsx`, `.csv`, or `.xls` with arbitrary columns. Could be a customer list the user maintains in Excel/Google Sheets.
- **CRM export** — same file extensions, columns specific to HubSpot / Salesforce / Pipedrive / etc.

Files may arrive via:
1. **Attached to the current chat** — if you can see one or more file attachments, use them.
2. **Absolute paths the user provides** — e.g. `C:\Users\Brad\Downloads\customers.xlsx`. Read directly.
3. **The user mentions a folder** — ask them for the full path(s).

If you cannot find any file after a brief check, ask:

> "Drop the file(s) here or paste the full path. I can take a LinkedIn `Connections.csv` / `messages.csv`, a spreadsheet (xlsx / csv / xls), or a CRM export."

If the user uploaded a `.zip` itself (or a folder containing the whole LinkedIn archive), tell them:

> "That's the full archive — I can't unzip it for you in V1. Extract `Connections.csv` and `messages.csv` from it and drop those in."

Do NOT attempt to unzip in V1.

## Detect file type

For each file the user gives you, identify which of the four branches it belongs to. The check order matters — extension-based branches first (PDF/image is exclusive), then LinkedIn (well-known fixed CSV headers), then fall through to the generic spreadsheet/CRM branch.

1. **Business card scan** — file extension is `.pdf`, `.jpg`, `.jpeg`, `.png`, `.heic`, or `.webp`. Route to **Card scan branch**.
2. **LinkedIn Connections** — file is `.csv` AND header row first cell is exactly `First Name` AND header contains `Last Name`, `Company`, `Position`, `Connected On`. Route to **Connections branch**.
3. **LinkedIn Messages** — file is `.csv` AND header row first cell is exactly `CONVERSATION ID`, OR header contains all of `FROM`, `CONTENT`, `FOLDER`. Route to **Messages branch**.
4. **Anything else** — generic spreadsheet or CRM export (`.csv`, `.xlsx`, `.xls`). Route to **Spreadsheet / CRM branch**.

For CSV files (and CSVs converted from xlsx/xls — see "Reading xlsx/xls files" below), peek the header row first to do steps 2-3. Skip LinkedIn's notes preamble (lines starting with `Notes:` or blank lines at the top).

If multiple files are dropped together, process them sequentially: confirm them all with the user, then parse + import in order:
- LinkedIn Connections first (since LinkedIn Messages references contacts via `linkedin_url`).
- LinkedIn Messages second.
- Business card scans next (they're independent — no email or linkedin_url cross-refs).
- Spreadsheet / CRM exports last (they may overlap with all of the above; the four-step dedup will catch matches via email or fuzzy name+company).

### Reading xlsx/xls files

Plain `.csv` reads via the standard Read tool. `.xlsx` / `.xls` are binary — handle as follows, in priority order:

1. If your environment can read xlsx directly (e.g. the `Read` tool returns parsed cells, or a spreadsheet-aware skill is available), use it.
2. Else, run a one-liner via Bash to convert: `python -c "import pandas; pandas.read_excel('<path>', dtype=str).to_csv('/tmp/hhq-spreadsheet.csv', index=False)"`. Then read the resulting CSV. Cleanup `/tmp/hhq-spreadsheet.csv` afterwards.
3. If neither path works in this environment, ask the user: "I can't read xlsx in this environment — could you save it as CSV and drop that in?" Stop and wait.

Do NOT attempt to parse xlsx by hand.

## Yes/no gate before parsing

Once file types are confirmed, summarise and confirm:

> "Found `<filename1>` (`<branch — Connections / Messages / Spreadsheet / CRM>`, ~`<rows>` rows)`<and filename2 if applicable>`. Import now? (yes / no)"

For **Spreadsheet / CRM** files there's a column-mapping step BEFORE the import — see Step S2 in the Spreadsheet branch. The yes/no above just confirms intent to start; mapping confirmation comes inline.

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
  "review_count": 0,
  "total": 170
}
```

`review_count` is the number of incoming rows that fuzzy-matched an existing contact but neither email nor `linkedin_url` confirmed identity, so the backend held them in the `import_review_queue` for later user confirmation. For a LinkedIn Connections re-import this should always be 0 (every row has a `linkedin_url` and matches step 2 of the dedup flow). If non-zero, surface it in the report — see "Report honestly" below.

---

## Messages branch

LinkedIn's `messages.csv` records every direct message the user has sent or received. We **do not** persist message bodies on the backend — that's privacy-noisy and we never use them. Instead, we digest the export client-side into one row per conversation partner: `{linkedin_url, last_messaged_at, message_count}`. The backend stamps those two fields onto matching contacts. That's it.

The digest powers a 30-day "don't message someone you just messaged" filter in `surface-next-5`, plus the `message_count` gives us a soft signal for "how talkative is this prospect with you." If we ever need the actual content of a recent thread, `research-and-draft` does a live Chrome read at draft time.

Process Messages AFTER Connections (if both are present) so the contacts they reference exist when the digest lands.

### Step M1 — Skip preamble + read header

Same preamble-stripping logic as Step C1. The header row is the first row whose first cell is exactly `CONVERSATION ID`.

### Step M2 — Validate columns

Required columns (LinkedIn uses uppercase headers):
- `FROM`
- `DATE`
- `FOLDER`

Useful optional columns:
- `SENDER PROFILE URL`
- `RECIPIENT PROFILE URLS`
- `CONTENT`

If a required column is missing, stop and tell the user the file doesn't look like a LinkedIn Messages export.

We do NOT need `CONTENT` to build the digest, but we DO need a profile URL to identify the other party — see Step M3.

### Step M3 — Build the digest client-side

Walk each row and group by the **other party's LinkedIn URL** (not the user's). For each row:

1. **Trim whitespace** on every cell.
2. **Skip the row if** there's no usable URL on either side (rare — LinkedIn occasionally exports rows for deleted accounts).
3. **Identify the other party** from `FOLDER`:
   - `FOLDER == "SENT"` → other party = `RECIPIENT PROFILE URLS` (use the first URL if comma/semicolon-separated)
   - Otherwise (INBOX, ARCHIVED, etc.) → other party = `SENDER PROFILE URL`
4. **Parse `DATE`** with the format LinkedIn uses (e.g. `2024-03-14 10:30:00 UTC`).
5. **Update the digest map** at key `<other_party_url>`:
   - Set `last_messaged_at` to the max of (current value, this row's date).
   - Increment `message_count` by 1.

After walking all rows, the digest is a flat list:

```json
[
  { "linkedin_url": "https://www.linkedin.com/in/coleman-greg", "last_messaged_at": "2026-04-20T10:30:00Z", "message_count": 47 },
  { "linkedin_url": "https://www.linkedin.com/in/marina-park", "last_messaged_at": "2026-03-02T14:05:00Z", "message_count": 3 }
]
```

**Do NOT include any message bodies, subjects, or conversation titles in the digest.** Privacy and storage hygiene.

### Step M4 — POST to /api/me/messages/import

```
POST <backend_url>/api/me/messages/import
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "messages": [ <digest row 1>, <digest row 2>, ... ]
}
```

Expected response (HTTP 200):

```json
{
  "updated": 412,
  "unchanged": 50,
  "unmatched": 835,
  "total": 1297
}
```

- `updated` — contacts whose `last_messaged_at` and/or `message_count` got newer values.
- `unchanged` — contacts where the digest matched what was already stored (e.g. re-importing the same export).
- `unmatched` — digest entries with a `linkedin_url` that doesn't exist in the user's contacts. Not an error — happens when someone's messaged someone outside their connections list.

Re-importing is safe: the backend won't regress timestamps and only updates when something changed.

---

## Spreadsheet / CRM branch

Catch-all branch for any file that isn't a LinkedIn Connections.csv or messages.csv. Could be a customer list the user keeps in Excel / Google Sheets, or a CRM export from HubSpot / Salesforce / Pipedrive / etc. We don't know the column layout in advance, so the mapping is the user-confirmed core of this branch.

### Step S1 — Read the file as text

For a `.csv`, read directly via the `Read` tool.

For `.xlsx` / `.xls`, follow the conversion priority order under "Reading xlsx/xls files" above. Once you have a text representation (CSV in memory or on /tmp), proceed.

If the file is empty or unparseable as a table, tell the user honestly and stop.

### Step S2 — Peek the headers + a few sample rows

Read the header row plus the first 3 data rows. You need both — sample rows let you sanity-check column meaning when header names are ambiguous (e.g. "Name" vs "Full Name", "Company" vs "Account").

### Step S3 — Propose a column mapping (Claude-driven, in-context)

Map every column in the file to one of the contacts schema fields, or to "ignore". The schema fields the backend accepts on import:

- `first_name` (required for every row)
- `last_name`
- `email`
- `phone`
- `company`
- `position`
- `linkedin_url`
- `website`
- `notes`

Plus one special pseudo-field:

- `pipeline_stage` — the column containing the contact's status / stage / lifecycle. Maps to `pipeline_stage_id` after the user confirms a stage-label mapping in Step S4.

Heuristics for proposing the mapping:

- Match on header name (case-insensitive) — `Email`, `Email Address`, `Email1` → `email`. `Phone`, `Phone Number`, `Mobile` → `phone`. `Company`, `Account Name`, `Organization`, `Org` → `company`. `Lifecycle Stage`, `Status`, `Stage`, `Lead Status` → `pipeline_stage`.
- If a single column holds a full name with no separate first/last (e.g. `Name` = "Greg Smith"), propose splitting on first whitespace into `first_name` + `last_name`. Note this in the mapping output as `Name → first_name + last_name (split on first whitespace)`.
- CRM-specific signals — if you see HubSpot's `Lifecycle Stage`, Salesforce's `LeadSource`, or Pipedrive's `Person ID`, lean toward `crm_csv` as the import source. Otherwise default to `spreadsheet`. (Source value is set in Step S6.)
- Any column that doesn't map cleanly → mark `ignore`. Don't guess.

Show the proposed mapping to the user as a short readable block:

> "Here's how I'm reading the columns:
>
> - `First Name` → first_name
> - `Last Name` → last_name
> - `Work Email` → email
> - `Mobile` → phone
> - `Account Name` → company
> - `Title` → position
> - `Lifecycle Stage` → pipeline_stage (I'll ask you about the stage labels next)
> - `HubSpot Score` → ignore
> - `Created Date` → ignore
>
> Look right? Adjust anything — or say 'go' to proceed."

If the user adjusts ("change Title to position", "ignore Mobile", "Account Name is actually the position not the company"), apply the change and re-show the mapping. Don't pad more than 2 rounds — if the user is grinding on it, gently suggest skipping the file and importing manually later.

### Step S4 — Stage label mapping (only if a `pipeline_stage` column was mapped)

Skip this step if no column was mapped to `pipeline_stage`.

Pull the unique values from that column across the entire file (case-insensitive dedup, trim whitespace). Common shapes:

- HubSpot: `Lead`, `Marketing Qualified Lead`, `Sales Qualified Lead`, `Opportunity`, `Customer`
- Salesforce: `Open - Not Contacted`, `Working - Contacted`, `Closed - Converted`, `Closed - Not Converted`
- Pipedrive: `Lead`, `Qualified`, `Won`, `Lost`
- Generic spreadsheet: anything — `Hot`, `Cold`, `Customer`, `Past client`, `Don't bother`, etc.

Fetch the user's Helper HQ pipeline stages:

```
GET <backend_url>/api/me/pipeline-stages
Authorization: Bearer <jwt>
```

Response: `{ "stages": [{id, slug, name, order}, ...] }` — seven default stages: Lead, Outreach sent, In conversation, Meeting booked, Proposal sent, Customer, Not a fit.

Propose a label-to-stage mapping using sensible defaults:

- Anything matching `lead` / `prospect` / `mql` / `cold` → `Lead`
- Anything matching `contacted` / `working` / `outreach sent` → `Outreach sent`
- Anything matching `qualified` / `sql` / `engaged` / `replied` → `In conversation`
- Anything matching `meeting` / `demo` / `discovery` → `Meeting booked`
- Anything matching `proposal` / `quote` / `pricing` / `negotiation` / `opportunity` → `Proposal sent`
- Anything matching `customer` / `won` / `closed-won` / `client` / `paying` → `Customer`
- Anything matching `lost` / `dead` / `closed-lost` / `disqualified` / `not a fit` / `unsubscribed` → `Not a fit`
- Anything else → ask the user (don't guess).

Show the proposed stage mapping:

> "Your file uses these status values. Here's how I'd map them onto your Helper HQ stages:
>
> - `Lead` → Lead
> - `MQL` → Lead
> - `SQL` → In conversation
> - `Customer` → Customer
> - `Closed - Lost` → Not a fit
> - `Past client` → Customer (or 'Not a fit'? your call)
>
> Look right? Adjust anything, or say 'go'."

Apply edits, re-show, gate at "go".

### Step S5 — Build the import payload

For every data row in the file:

1. **Trim whitespace** on every cell.
2. **Skip the row if** the column mapped to `first_name` is empty AND `email` is empty (no identity anchor — would just become a junk contact).
3. **Apply the column mapping** — for each non-ignored column, copy the cell into the corresponding schema field. If `Name` was mapped to `first_name + last_name (split)`, do the split.
4. **Apply the stage mapping** — look up the row's stage label in the user-confirmed mapping from Step S4, and set `pipeline_stage_id` to the matching HHQ stage's id (from the GET in Step S4). If the row's stage label wasn't in the mapping (an edge case the user didn't pre-classify), leave `pipeline_stage_id` null — it'll land in the Lead bucket.
5. **Build the row object**:

```json
{
  "first_name": "Greg",
  "last_name": "Smith",
  "email": "greg@magnetorquer.com",
  "phone": "+61 400 000 000",
  "company": "Magnetorquer",
  "position": "VP Eng",
  "website": "https://magnetorquer.com",
  "linkedin_url": null,
  "notes": null,
  "pipeline_stage_id": 47,
  "source": "spreadsheet",
  "raw_csv": { ...the original row as a key/value object... }
}
```

`source` is `"spreadsheet"` by default, or `"crm_csv"` if Step S3 detected CRM-specific signals. Preserve the full original row in `raw_csv` so the backend can re-derive things later.

Do NOT compute slugs client-side or do any dedup work — that all happens server-side via `ContactDeduplicator`.

### Step S6 — POST to /api/me/contacts/import

Same endpoint as the LinkedIn Connections branch:

```
POST <backend_url>/api/me/contacts/import
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "contacts": [ <row 1>, <row 2>, ... ]
}
```

Response shape:

```json
{
  "created": 73,
  "updated": 12,
  "unchanged": 4,
  "review_count": 8,
  "total": 97
}
```

For spreadsheet / CRM imports, expect `review_count` to be non-zero — rows without `email` or `linkedin_url` that fuzzy-match an existing contact (name + company) will land in the import review queue rather than auto-merging. That's the dedup flow doing its job.

---

## Card scan branch

Vision-based extraction for one or many business cards in a single PDF or image (jpg, png, heic, webp). The actual extraction prompt is server-controlled and tuned in Filament — fetch it at runtime via `/api/mcp/prompts/extract_business_card`. Don't reimplement the prompt locally; if the fetch fails, halt and tell the user the backend is unreachable.

Soft cap: ~50 cards per upload. If a single file looks like it has more (rare), tell the user to split into multiple uploads. The vision pass quality degrades on large multi-card layouts.

### Step B1 — Read the file

Use the `Read` tool — it handles PDFs and images natively in Claude Code / Cowork. Hold the visual content in conversation context for the vision pass below.

If the file is more than ~10 MB or `Read` chokes on it, tell the user honestly and stop — they may need to compress the scan or split it.

### Step B2 — Fetch the extract_business_card prompt from backend

```
GET <backend_url>/api/mcp/prompts/extract_business_card
Authorization: Bearer <jwt>
```

Response: `{ name, template, version, updated_at }`. The template has no `{{...}}` substitutions — use it as-is. Cache for the session in case the user uploads multiple card files in one chat.

If the GET returns 404 or 5xx, halt:

> "I couldn't fetch the card-extraction prompt from the backend — it might be down. Try again in a few minutes."

Do NOT fall back to a local prompt. The remote prompt is the canonical IP and we want it tuned in one place.

### Step B3 — Vision pass: extract all cards

Run a vision pass on the image/PDF using the fetched prompt as the instructions. The prompt asks for a JSON array under `{ "cards": [...] }`, one object per card detected, each with: `first_name`, `last_name`, `company`, `position`, `email`, `phone`, `website`, `linkedin_url`, `notes`, and a `status` enum (`ok` | `partial` | `no_individual_name` | `event_card` | `blank_or_back`).

The plugin runs in Claude Code / Cowork — Claude already has vision capabilities natively. No separate API call is needed; just instruct Claude to look at the attached image and return the JSON shape per the prompt.

Parse the response. If parsing fails (Claude returned malformed JSON or didn't follow the shape), retry once. If it fails twice, halt and tell the user:

> "Vision extraction didn't return clean data this time. Try uploading a clearer scan, or split a multi-card file into smaller ones."

### Step B4 — Route by per-card status

Group the extracted cards by status:

| status | What to do |
|---|---|
| `ok` | Goes into the user-review list as-is. |
| `partial` | Goes into the user-review list with gaps highlighted (which fields are null). User fills in or removes. |
| `no_individual_name` | Show separately and ask: import as a company contact (first_name = company name) or skip? |
| `event_card` | Auto-skip. List in the "skipped" summary so user knows what was dropped. |
| `blank_or_back` | Auto-skip. Same. |

### Step B5 — User review

Present the to-review cards numbered. Format per card:

```
1. **<first_name> <last_name>** — <position>, <company>
   email: <email>  |  phone: <phone>  |  linkedin: <linkedin_url>
   website: <website>
   notes: <notes>
```

For `partial` cards, replace null fields with an italic `(missing — fill in?)` so the user sees the gaps:

```
2. **Greg Smith** — *(missing — fill in?)*, Magnetorquer
   email: greg@magnetorquer.com  |  phone: *(missing)*
   website: *(missing)*
   notes: *(none)*
```

For `no_individual_name` cards, ask in-line:

> "Card N has no individual name — just `<company name>` and `<email if any>`. Import as a company contact, or skip?"

For auto-skipped cards, list them briefly at the bottom:

> "Skipped (no contact data extractable):
> - Card 4: event card (Stratasys / 41st Space Symposium)
> - Card 7: back of card / blank scan"

Then a single edit/confirm prompt:

> "Look right? You can edit any field ('change Greg's company to Magnetorquer Pty Ltd'), drop a card ('skip card 3'), fill a gap ('Greg's phone is +61 400 000 000'), or say 'go' to import all the ones in the to-review list."

Loop edits/drops until the user says go. Two rounds of edits is fine; if the user is grinding past three, gently suggest re-uploading a cleaner scan.

### Step B6 — Build the import payload + POST

For each confirmed card (including any `no_individual_name` cards the user opted to import as company contacts):

1. Build the row:

```json
{
  "first_name": "Greg",
  "last_name": "Smith",
  "company": "Magnetorquer Pty Ltd",
  "position": "VP Engineering",
  "email": "greg@magnetorquer.com",
  "phone": "+61 400 000 000",
  "website": "https://magnetorquer.com",
  "linkedin_url": null,
  "notes": "Met at AWS re:Invent. ETA Feb.",
  "source": "business_card"
}
```

2. For company-as-contact rows: set `first_name` to the company name, leave `last_name` null, set `company` to the same name. Slug will become `<company-slug>-<company-slug>` which is a bit ugly but harmless — fix in V2 if it bites.

3. POST as one batch:

```
POST <backend_url>/api/me/contacts/import
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "contacts": [ <card 1>, <card 2>, ... ]
}
```

Response shape (same as other branches):

```json
{
  "created": 7,
  "updated": 1,
  "unchanged": 0,
  "review_count": 1,
  "total": 9
}
```

For card-scan imports, expect `review_count` to occasionally be non-zero — a card without `email` or `linkedin_url` that fuzzy-matches an existing contact (same person, met previously) lands in the review queue rather than auto-merging.

---

## Error handling (all branches)

- **HTTP 401** → token wasn't accepted. Run the refresh / re-activate flow from Phase 0 once, then retry. If the second try also fails, tell the user their session is broken and to re-onboard.
- **HTTP 422** → validation errors. The error body has `errors.contacts.{idx}.{field}` or `errors.messages.{idx}.{field}`. Tell the user honestly: "N rows had problems and weren't imported (e.g. row 12: missing linkedin_url). The rest went through if any." Show the first 2-3 indices and fields. Don't dump the whole error.
- **HTTP 5xx / network** → tell the user the backend's not responding, suggest retry in a moment. Don't keep retrying automatically.

## Report honestly

After a successful import, give a brief summary in plain language.

**Connections-only result:**

> "Imported `<filename>`.
>
> - **`<created>` new** prospects added.
> - **`<updated>` updated** (company, position, or other CSV-derived fields changed).
> - **`<unchanged>` already had** — nothing changed.
> - *(if `review_count` > 0)* **`<review_count>` need your review** — incoming rows that look similar to an existing contact but neither email nor LinkedIn URL confirmed identity. I'll walk you through these when a review skill ships in a later patch.
>
> Total in your master now: **`<total imported>` from this file** (plus anything from earlier imports).
>
> Ready when you are — when you want to start working through them, just say 'get me the next 5 prospects' or similar."

For LinkedIn Connections imports `review_count` should always be 0 (every row has a LinkedIn URL → matches step 2 of the dedup flow). The line in the template above is conditional — only mention it when `review_count > 0`. The non-zero path will start firing once spreadsheet/CRM CSV, business card, and Gmail ingest sources land in subsequent plugin patches.

**Messages-only result:**

> "Digested `<filename>`.
>
> - **`<updated>`** contacts now have a 'last messaged' date and message count attached.
> - **`<unchanged>`** were already up to date.
> - **`<unmatched>`** are message threads with people not in your contacts list — that's expected, especially for one-off conversations.
>
> What this means: when you ask for the next 5 prospects, I'll skip anyone you've messaged in the last 30 days. The message bodies themselves stay in LinkedIn — I only kept the timestamps and counts."

**Both files imported in one go:** show both blocks back-to-back, brief.

**Spreadsheet / CRM result:**

> "Imported `<filename>` (`<spreadsheet | CRM export>`).
>
> - **`<created>` new** contacts added.
> - **`<updated>` updated** (matched an existing contact by email or LinkedIn URL — fields filled in or refreshed).
> - **`<unchanged>` already had** — nothing changed.
> - *(if `review_count` > 0)* **`<review_count>` need your review** — rows that look similar to an existing contact (same name + company) but neither email nor LinkedIn URL confirmed it. I'll walk you through these when a review skill ships in a later patch.
>
> *(if a stage column was mapped)* **`<N>` of those landed in your active pipeline** at the stages you mapped. View: `https://hhq.ngrok.dev/pipeline`.
>
> Total in your master now: **`<created + updated + unchanged>` from this file** (plus anything from earlier imports)."

**Business card scan result:**

> "Scanned `<filename>` — `<total cards detected>` cards in the image.
>
> - **`<created>` new** contacts added.
> - *(if `updated` > 0)* **`<updated>` updated** (someone you already had in your contacts — fields filled in from the card).
> - *(if `unchanged` > 0)* **`<unchanged>` already had** — same details as last time.
> - *(if `review_count` > 0)* **`<review_count>` need your review** — same name + company as someone you already have, but no shared email or LinkedIn URL. I'll walk you through these later.
> - *(if any auto-skipped)* **`<N>` skipped** — `<list reasons briefly: '1 event card, 2 blank scans'>`.
>
> Ready when you are — the new ones are in your Lead bucket, surface them any time with 'get me the next 5'."

Do NOT echo full lists. Do NOT show example rows.

## Things you must NOT do

- Do NOT write a local `contacts-master.csv` or any contact data to disk. The backend is the master.
- Do NOT compute slugs, do dedup logic, or merge state client-side. The backend does all of that.
- Do NOT modify `<project-dir>/.hhq-session.json` except to update `jwt` and `jwt_expires_at` after a refresh.
- Do NOT call any other API endpoint or external service.
- Do NOT delete prospects from the master if they're missing from a fresh export — the backend keeps everything by design (you couldn't delete them even if you tried; the import endpoint is upsert-only).
- Do NOT attempt to parse `.zip` archives — instruct the user to extract `Connections.csv` (and `messages.csv` if present) first.
- Do NOT mix rows from Connections and Messages into the same POST. Each file goes to its own endpoint with its own row schema.
- Do NOT include message bodies, subjects, or conversation titles in the messages digest. The backend doesn't accept them — and we don't want them stored.
- Do NOT try to derive direction from anything other than `FOLDER` (LinkedIn's own classification). Don't guess from FROM/TO names.
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
