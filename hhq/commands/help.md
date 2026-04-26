---
description: Lists every Helper HQ skill the user's licence entitles them to, grouped by helper, plus locked skills available on higher tiers. Reads tier from the local config (V1) or the signed token (V2).
---

# /hhq:help — Helper HQ skill directory

When invoked, render a clean directory of every skill the user's licence currently entitles them to, plus a brief list of locked skills they could unlock by upgrading.

## Auth check (V1 stub)

Same auth-check stub as the helper skills: in V1 dogfood, assume `auth_ok = true` and proceed. In Stage 2 this becomes a signed-token check; if the token is missing or invalid, output the activation prompt instead of the help directory.

## Determine tier and helper entitlements

V1 dogfood:
- Read `~/.hhq/sales-helper/config.json` (Windows: `%USERPROFILE%\.hhq\sales-helper\config.json`).
- Pull `tier` field. If missing or unreadable, default to `"lite"`.
- Treat `helpers` as `["sales-helper"]` (only helper available in V1).

Stage 2 (when wired up): verify the locally stored signed token, pull `tier` and `helpers[]` from the verified payload.

## Determine onboarding state

Briefly check whether `config.json` actually exists and looks complete (`offer`, `icp`, `signals.weighted` populated). If not, prepend a one-line note at the top of the output:

> "⚠ You haven't finished onboarding yet — say 'set me up' to get started."

Don't block the help listing — still show it. The note is just informational.

## Render the directory

Use this exact shape. Adapt the active/locked split to the user's tier.

```
**Helper HQ** • Tier: <tier title-cased>

═══ Sales Helper (<tier title-cased>) ═══

Active skills:

• **onboard-user** — One-time setup. Captures your offer, ICP, and 5 weighted signals; kicks off your LinkedIn export. Triggers when you're new or say "set me up", "re-onboard", "start over".

• **ingest-contacts** — Imports your LinkedIn export into the master contact list. Triggers when you drop a CSV in chat or say "I've got my LinkedIn export", "import my contacts".

• **surface-next-5** — Filters your contacts by your weighted signals and surfaces the top 5 with one-line reasoning. Triggers on "get me the next 5 prospects", "who should I reach out to", "next 5".

• **research-and-draft** — Researches each surfaced prospect via their LinkedIn profile + recent posts, drafts a Greg-style opener for each, saves to per-prospect folders. Triggers on "let's go", "draft them" after surfacing.

Locked — upgrade to unlock:

• **voice-profile** (Pro) — Learns your writing style from past messages so drafted openers sound like you, not like an SDR.
• **follow-up-drafter** (Pro) — Drafts timed follow-ups for prospects who haven't replied, aware of the original opener.
• **deep-research** (Pro) — Pulls company news, funding rounds, recent press alongside the LinkedIn profile read.
• **email-ingest** (Pro) — Ingests Gmail / Outlook contacts in addition to LinkedIn.
• **apollo-enrich** (Elite) — Finds NEW prospects from Apollo enrichment, not limited to your existing LinkedIn connections.
• **linkedin-discover** (Elite) — Discovers prospects matching your ICP from outside your warm network.

═══ Marketing Helper ═══

Coming in V2 — not yet available.

═══ Storage ═══

Your data lives at `~/.hhq/sales-helper/` (Windows: `%USERPROFILE%\.hhq\sales-helper\`). Open any file in that folder any time — it's all plain text and yours.

═══ Need a hand? ═══

If you're stuck on what to say, just describe what you want to do in plain language. The skills above auto-trigger from natural phrasing — you almost never need to invoke them by name.
```

## Tier-based locking rules

| Tier | Active | Locked (shown as upgrade) |
|---|---|---|
| `lite` | onboard-user, ingest-contacts, surface-next-5, research-and-draft | All Pro and Elite skills above |
| `pro` | All Lite skills + voice-profile, follow-up-drafter, deep-research, email-ingest | Elite skills above |
| `elite` | All Lite + Pro skills + apollo-enrich, linkedin-discover | (no locked skills) |

When rendering, only show the **Locked** section if there are actually locked skills for the user's tier. For an Elite user, omit the Locked section entirely.

## What this command does NOT do

- Does NOT modify any files.
- Does NOT call any external API.
- Does NOT trigger any other skill — it's pure rendering.
- Does NOT show roadmap items beyond the Pro/Elite skills listed above. Don't speculate about V3 features.
- Does NOT pretend a locked skill is active. If the user's tier doesn't include a skill, list it as locked.
- Does NOT show internal config values (token contents, machine_id, raw signal weights).

## Edge cases

- **No config.json** → still render the help block, but with the onboarding warning at the top. Use default tier `lite`.
- **Malformed tier value** → default to `lite`, render help, show a discreet note: "_(tier read defaulted to lite — check config.json)_".
- **`helpers[]` does not include `sales-helper`** (future state) → omit the Sales Helper block. If no helpers at all, render an empty-state message: "No helpers active on your licence — visit helperhq.co to add one."
