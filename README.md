# Helper HQ — Claude Plugin

A Claude Code / Cowork plugin family for solo founders, coaches, and consultants. The plugin prepares, you review and send.

This repo is both the **plugin** (`hhq`) and the **`helper-hq` marketplace** that distributes it.

## Status

**V1 MVP — public dogfood.** Currently shipping **Sales Helper Lite**: signal-driven outbound prospecting from your LinkedIn connections.

## Install

In Claude Code or Cowork:

```
/plugin marketplace add https://github.com/Helper-HQ/claude-plugin
/plugin install hhq@helper-hq
```

After install, run `/hhq:help` to see what the plugin does and what skills are available on your tier.

## Updating

Whenever this repo gets new commits, pull the latest in your client:

```
/plugin marketplace update helper-hq
```

## Roadmap

- **Lite** *(V1, in build)* — LinkedIn warm-network only, on-demand, 5 prospects at a time
- **Pro** *(V2)* — LinkedIn + email + phone, deeper research, voice profile learning, follow-up drafting
- **Elite** *(V2+)* — new prospect discovery via LinkedIn / Apollo enrichment
- **Marketing Helper, Ops Helper, …** — additional helpers in the same `hhq` plugin

Tier and helper entitlements are gated by a server-issued signed token (V2). V1 dogfood ships with the auth check stubbed.

## Architecture

- Single `hhq` plugin contains all helpers and tiers, gated by a signed token (V2)
- Hybrid skill model — low-IP skills full-local, high-IP skills structured to move to server-side MCP calls in V2
- Marketplace pulls from `main` branch on every `/plugin marketplace update`

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
