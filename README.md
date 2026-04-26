# Helper HQ — Claude Plugin

A Claude Code / Cowork plugin family for solo founders, coaches, and consultants. The plugin prepares, you review and send.

This repo is both the **plugin** (`hhq`) and the **`helper-hq` marketplace** that distributes it.

## Status

**V1 MVP — public dogfood.** Currently shipping **Sales Helper Lite**: signal-driven outbound prospecting from your LinkedIn connections.

## Install

In Claude Code (CLI) or Cowork:

```
/plugin marketplace add https://github.com/Helper-HQ/claude-plugin
/plugin install hhq@helper-hq
```

After install, run `/hhq:help` to see what the plugin does and what skills are available on your tier.

## Getting started

Once the plugin's installed, here's the full flow end-to-end. About **15 minutes for setup, then 5–10 minutes whenever you want to surface fresh prospects.**

### 1. Onboard

In any chat, say:

> *"set me up"*

(Other phrases that work: *"I'm new"*, *"onboard me"*, *"start over"*.)

You'll be prompted to paste your **Helper HQ licence key** — it looks like `hhq_...` and was delivered in your purchase email. Onboarding will then:

1. Activate your licence against the backend
2. Walk you through the LinkedIn connections export (LinkedIn takes up to 24 hours to email it back, so we kick that off first)
3. Capture your **offer**, **ICP** (ideal customer profile), and the **5 weighted signals** you want prospects ranked by

That's a one-time, ~15-minute setup. The plugin saves your config to the Helper HQ backend and writes a small `.hhq-auth.json` to your project folder so it can recognise you in future chats.

### 2. Import your LinkedIn connections

When LinkedIn emails you the export (`Connections.csv`), open a fresh chat, drop the file in, and say:

> *"I've got my LinkedIn export"*

(Or just drop the CSV — the plugin will pick it up.)

The plugin parses the export and uploads every connection to your master prospect list on the Helper HQ backend.

### 3. Surface prospects

Whenever you want to work on outreach, open a fresh chat and say:

> *"get me the next 5 prospects"*

The plugin filters your master list by your weighted signals and presents 5 candidates with one-line reasoning per pick. You can drop one (*"not Greg"*), swap (*"someone in fintech instead"*), or confirm.

### 4. Research and draft

When the 5 look good, say:

> *"let's go"*

The plugin researches each prospect via their LinkedIn profile + recent posts, drafts a short Greg-style opener for each (specific, signal-referencing, soft ask), and presents all 5 ready to copy/edit/send.

### 5. Anything you want to track per prospect

Each surfaced prospect gets a folder at `<project-folder>/contacts/<first-last-company>/notes.md`. Open it any time — it's plain markdown, yours to edit. The plugin reads your notes when drafting future openers, so things like *"Greg's away till Jan, come back then"* or *"don't pitch testing — they have an in-house lab"* shape what you get back.

### 6. See what's available

Run `/hhq:help` any time to list skills active on your tier and what's locked behind upgrades.

## Updating

When this repo gets new commits:

- **Claude Code CLI:** run `/plugin marketplace update helper-hq` in any chat. Picks up the latest commit immediately.
- **Cowork:** there's no slash command for marketplace refresh in Cowork yet. The current workaround is to clear Cowork's plugin cache folder so the next chat re-pulls the marketplace, or uninstall + reinstall via the Customize menu.

## Roadmap

- **Lite** *(V1, in build)* — LinkedIn warm-network only, on-demand, 5 prospects at a time
- **Pro** *(V2)* — LinkedIn + email + phone, deeper research, voice profile learning, follow-up drafting
- **Elite** *(V2+)* — new prospect discovery via LinkedIn / Apollo enrichment
- **Marketing Helper, Ops Helper, …** — additional helpers in the same `hhq` plugin

Tier and helper entitlements are gated by a server-issued signed token (V2). V1 dogfood ships with the auth check stubbed.

## Architecture

- Single `hhq` plugin contains all helpers and tiers, gated by a signed token (V2)
- Hybrid skill model — low-IP skills full-local, high-IP skills structured to move to server-side MCP calls in V2
- Marketplace pulls from `main` branch on every CLI `/plugin marketplace update`

See [`hhq/README.md`](hhq/README.md) for plugin-level details (skills, storage layout, dev loop).

## Repo layout

```
helper-hq/claude-plugin/
├── .claude-plugin/
│   └── marketplace.json          ← this marketplace
├── hhq/                          ← the plugin
│   ├── .claude-plugin/plugin.json
│   ├── skills/                   ← flat, one folder per skill
│   ├── commands/help.md
│   └── README.md
└── README.md                     ← this file
```

## Contributing

Solo dev for now. Issue tracker is the main contact channel during dogfood.

## Licence

UNLICENSED during dogfood. Licensing model lands with V2.
