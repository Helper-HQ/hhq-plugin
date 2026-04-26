# Helper HQ (`hhq`)

A Claude Code / Cowork plugin family. Helper HQ ships as a single `hhq` plugin containing one or more **helpers**, each tier-gated by a server-issued signed token.

V1 ships **Sales Helper Lite** — an on-demand sales assistant for solo coaches and consultants. Open a chat, ask for the next 5 prospects, get short tailored openers drafted from real signals.

The magic is in the messages. Sending is the easy part.

## Status

**V1 MVP — in development.** V1 is **Sales Helper Lite** only. Dogfooding against Sunburnt Space.

Roadmap:
- **Lite** (V1, in build) — LinkedIn warm-network only, on-demand, 5-at-a-time
- **Pro** (V2) — LinkedIn + email + phone, deeper research, voice profile
- **Elite** (V2+) — new prospect finding via LinkedIn / Apollo enrichment
- **Marketing Helper, Ops Helper, …** (V2+) — additional helpers in the same `hhq` plugin

The plugin manifest name is `hhq`. Tier and helper entitlements are read from a signed token issued by the HHQ licensing server. V1 dogfood ships with the auth check stubbed.

## Getting started

After installing the plugin (`/plugin install hhq@helper-hq`):

| You say | What runs |
|---|---|
| *"set me up"* | `onboard-user` — captures licence key, offer, ICP, signals; kicks off LinkedIn export |
| *"I've got my LinkedIn export"* (or drop the CSV in chat) | `ingest-contacts` — uploads connections to backend |
| *"get me the next 5 prospects"* | `surface-next-5` — ranks and presents 5 with reasoning |
| *"let's go"* (after surfacing) | `research-and-draft` — researches each, drafts openers |
| `/hhq:help` | Lists skills active on your tier |

Full walkthrough in the [top-level README](../README.md#getting-started).

## How it works (V1, Sales Helper Lite)

1. **Once:** run onboarding — capture what you offer, who your target is, and up to 5 weighted signals that mark a prospect as worth opening a conversation with. Kick off your LinkedIn connections export (LinkedIn takes up to 24h to email it back).
2. **Once the export arrives:** drop the CSV in a chat. Plugin parses it and uploads to the Helper HQ backend.
3. **Whenever you're ready:** open a fresh chat and say *"get me the next 5 prospects."* Plugin returns 5 candidates with one-line reasoning. Say *"let's go"* — plugin researches each, drafts a Greg-style opener, and shows you the openers to copy/edit/send.

No weekly cadence. No automation. Run when you want more.

## Local development

V1 is distributed via the `helper-hq` marketplace at https://github.com/Helper-HQ/claude-plugin.

### Cowork

Edit a `SKILL.md` under `skills/<skill>/`, commit, push. There's no slash command to refresh installed marketplace plugins in Cowork — clear Cowork's plugin cache folder, or uninstall + reinstall the plugin via the Customize menu.

### Claude Code CLI

Edit, commit, push, then `/plugin marketplace update helper-hq`. Or for tighter iteration, install the plugin from a local path so edits hot-reload without a push:

```
/plugin marketplace add C:\Code\HelperHQ\Claude-Plugin
/plugin install hhq@helper-hq
/reload-plugins              # after each edit
```

## Architecture

- **Single `hhq` plugin.** All helpers (sales, future marketing, future ops) and all tiers (lite/pro/elite) live inside, gated by a signed token.
- **Hybrid skill model.** Low-IP skills (CSV parsing, status updates, onboarding) ship as full-local skill markdown. High-IP skills (prospect ranking, research analysis, opener drafting) are *structured* as hollow skills that will move to server-side MCP calls in V2 — the methodology lives behind the API, not on the user's disk.
- **Auth check** at the top of every skill. Reads `.hhq-auth.json` from the project folder and runs a refresh-or-reactivate fallback before any backend call.
- **Backend API.** All durable state (config, contacts, current batch, per-prospect research and message history) lives on the Helper HQ backend at `https://hhq.helperhq.com` (V1 dogfood: a Herd / Expose tunnel; production swaps in a stable domain).

## Skills (V1, Sales Helper)

| Skill | Triggers on... | Purpose |
|---|---|---|
| `onboard-user` | *"set me up"*, auto on first use | Activate licence; capture offer + ICP + 5 weighted signals; start LinkedIn export |
| `ingest-contacts` | *"I've got my export"*, CSV dropped in chat | Parse LinkedIn CSV, upload to backend |
| `surface-next-5` | *"get me the next 5 prospects"* | Filter master list by signal weights → show 5 with reasoning |
| `research-and-draft` | *"let's go"* (after surface) | Research each prospect via Chrome connector, draft Greg-style opener |

Plus one slash command:

| Command | Purpose |
|---|---|
| `/hhq:help` | List skills the user's token entitles them to, with one-line descriptions |

## Storage (V1)

Almost everything lives on the Helper HQ backend now:

- `config` (offer, ICP, 5 weighted signals) — `PUT /api/me/config`
- `contacts` master list — `POST /api/me/contacts/import`, `GET/PUT /api/me/contacts/{slug}`
- `current_batch` (the 5 in-flight prospects) — `GET/PUT /api/me/current-batch`
- per-prospect research and drafted messages — fields on the contact record

Two things stay local in the project folder:

- `.hhq-auth.json` — your licence key, machine ID, cached JWT, backend URL. The plugin reads this on every invocation.
- `contacts/<first-last-company>/notes.md` — per-prospect freeform notes you own. The plugin creates a placeholder when it first researches a prospect; you edit; the plugin reads your notes the next time it drafts. Slug format is `firstname-lastname-companyhint` for browsability.

In Cowork, the project folder is supplied via `mcp__ccd_directory__request_directory` and persists across chats in the same project. In local Claude Code CLI it falls back to `~/.hhq/sales-helper/`.

No multi-project support in V1. Each Cowork project is isolated and uses one of your 3 machine slots — open multiple projects only if you really need to.

No pipeline tab, no weekly log, no voice profile capture. See `memory/project_backlog.md` for everything cut and queued for V2.

## Architecture references

- [`memory/project_product_reframe.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_product_reframe.md) — what the product actually is (signal-driven, not warm-network)
- [`memory/project_v1_scope.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_v1_scope.md) — locked V1 scope
- [`memory/project_plugin_architecture.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_plugin_architecture.md) — single HHQ plugin decision
- [`memory/project_auth_architecture.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_auth_architecture.md) — signed token + hybrid skills
- [`memory/project_backlog.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_backlog.md) — V2+ items
