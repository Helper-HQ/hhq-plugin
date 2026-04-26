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

The plugin manifest name is `hhq`. Tier and helper entitlements are read from a signed token issued by the HHQ licensing server (V2). V1 dogfood ships with the auth check stubbed.

## How it works (V1, Sales Helper Lite)

1. **Once:** run onboarding — capture what you offer, who your target is, and up to 5 weighted signals that mark a prospect as worth opening a conversation with. Kick off your LinkedIn connections export (LinkedIn takes up to 24h to email it back).
2. **Once the export arrives:** drop the CSV in a chat. Plugin parses it into your master prospect list.
3. **Whenever you're ready:** open a fresh chat and say *"get me the next 5 prospects."* Plugin returns 5 candidates with one-line reasoning. Say *"let's go"* — plugin researches each, drafts a Greg-style opener, saves to per-prospect folders, and shows you the openers to copy/edit/send.

No weekly cadence. No automation. Run when you want more.

## Local development

V1 targets Cowork. Bundle the plugin folder into a `.plugin` file and install via Cowork's **Customize → Browse plugins**. To iterate on a skill, edit the relevant `SKILL.md` under `skills/sales-helper/<skill>/`, re-bundle, and reinstall (or use `/reload-plugins` in a CLI session if you have the Claude Code CLI).

## Architecture

- **Single `hhq` plugin.** All helpers (sales, future marketing, future ops) and all tiers (lite/pro/elite) live inside, gated by a signed token.
- **Hybrid skill model.** Low-IP skills (CSV parsing, file management, onboarding) ship as full-local skill markdown. High-IP skills (prospect ranking, opener drafting) are *structured* as hollow skills that will move to server-side MCP calls in V2 — the methodology lives behind the API, not on the user's disk.
- **Auth check stub** at the top of every skill. V1 dogfood always returns `true`. Stage 2 wires the real signed-token verification (JWT-style, server-signed, locally verified via embedded public key).

See `memory/project_plugin_architecture.md` and `memory/project_auth_architecture.md` for full details.

## Skills (V1, Sales Helper)

| Skill | Status | Triggers on... | Purpose |
|---|---|---|---|
| `onboard-user` | **Drafted** | "set me up", auto on first use | Capture offer + ICP + 5 weighted signals; start LinkedIn export |
| `ingest-contacts` | **Drafted** | "I've got my export", CSV dropped in chat | Parse LinkedIn CSV into master prospect list |
| `surface-next-5` | **Drafted** | "get me the next 5 prospects" | Filter master list by signal weights → show 5 with reasoning |
| `research-and-draft` | **Drafted** | "let's go" (after surface) | Research each prospect, draft Greg-style opener, save to folders |

Plus one slash command (in build):

| Command | Purpose |
|---|---|
| `/hhq:help` | List all skills the user's token entitles them to, with one-line descriptions |

## Storage (V1)

User-level only at `~/.hhq/sales-helper/` (Windows: `%USERPROFILE%\.hhq\sales-helper\`):

```
~/.hhq/sales-helper/
  config.json                       tier, offer, ICP, 5 weighted signals
  contacts-master.csv               parsed LinkedIn CSV + status column
  current-batch.json                the 5 prospects in flight (cleared after draft)
  contacts/
    <prospect-slug>/
      research.md                   what Claude found
      notes.md                      your notes
      messages.md                   drafted/sent messages, chronological
```

Future helpers nest under their own subdir (e.g. `~/.hhq/marketing-helper/`).

No multi-project support in V1. No pipeline tab, no weekly log, no voice profile capture. See `memory/project_backlog.md` for everything cut and queued for V2.

## Architecture references

- [`memory/project_product_reframe.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_product_reframe.md) — what the product actually is (signal-driven, not warm-network)
- [`memory/project_v1_scope.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_v1_scope.md) — locked V1 scope
- [`memory/project_plugin_architecture.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_plugin_architecture.md) — single HHQ plugin decision
- [`memory/project_auth_architecture.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_auth_architecture.md) — signed token + hybrid skills
- [`memory/project_backlog.md`](../../Users/Brad/.claude/projects/C--Code-HelperHQ-Claude-Plugin/memory/project_backlog.md) — V2+ items
