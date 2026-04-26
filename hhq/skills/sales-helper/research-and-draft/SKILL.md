---
name: research-and-draft
description: Researches each prospect in the current batch and drafts a Greg-style opener for each. Triggers when the user says "let's go", "draft them", "start drafting", "go for it", or any equivalent confirmation right after surface-next-5 has presented a batch. Reads `~/.hhq/sales-helper/current-batch.json` (the 5 confirmed prospects from surface-next-5), uses the Chrome connector to read each prospect's LinkedIn profile + recent posts (narrow scope only), saves findings to `~/.hhq/sales-helper/contacts/<slug>/research.md`, drafts a short signal-referenced opener, saves it to `~/.hhq/sales-helper/contacts/<slug>/messages.md` with timestamp, updates each prospect's status in the master to `drafted`, then presents all openers cleanly in chat for the user to copy and send. Run AFTER surface-next-5 has produced a current-batch.json. If no current-batch.json exists, route the user to surface-next-5 first.
---

# Research and Draft — Sales Helper Lite

You are doing pass-2 enrichment and opener drafting for the 5 prospects the user just confirmed in `surface-next-5`. For each prospect: read their public LinkedIn profile + recent posts via the Chrome connector, capture findings, draft a short signal-referenced opener in the Greg style, save artifacts to per-prospect folders, update the master, present the openers for copying.

This is the heaviest V1 skill and it's where the actual product value lives. Work through it carefully — quality of research and quality of drafts is the whole product.

## When this skill runs

Trigger when the user says any variant of:
- "let's go"
- "draft them"
- "start drafting"
- "go for it"
- "do it"
- "yes draft"
- "give me the openers"

…AND `~/.hhq/sales-helper/current-batch.json` exists with at least one prospect.

If a user says one of these phrases but `current-batch.json` doesn't exist, route them to surface-next-5 first ("There's no current batch — let's surface 5 prospects first. Say 'get me the next 5'.").

## Auth check (V1 stub)

Before doing anything else, run the auth check.

For V1 dogfood this is a stub that always returns `true`. When the licence backend ships (Stage 2), this becomes a signed-token verification against the locally stored token. Skill code should call the check, branch on the result, and never run skill logic if it returns `false`.

For V1: assume `auth_ok = true` and proceed. Leave a code-comment-style note in your reasoning that this is the auth seam, so future-you knows where to wire the real check.

## Hollow-skill seams (V1 stubs)

This skill has **two** hollow-skill seams that move to server-side MCP calls in V2:

**Seam A — Research analysis** (`hhq__analyze_research`):
Takes the raw page-scrape data (profile blob + recent post blobs) and returns a structured `research.md`-shaped finding with signals matched and a "hook for opener" suggestion. The IP is the analysis prompt — *what to look for, what counts as a signal, how to weight a recent post against a role change*, etc.

**Seam B — Opener drafting** (`hhq__draft_opener`):
Takes the research findings + offer + ICP + voice notes and returns a Greg-style opener. The IP is the drafting prompt — the style guide, the structural rules, the anti-patterns to avoid.

For V1 dogfood you do both yourself in-context. The Greg-style methodology and the analysis heuristics are written below. Treat them as the high-IP content that will live behind the MCP API in V2 — they ship in plaintext today only because we accept that compromise for the dogfood phase.

## Pre-flight checks

Determine the user-level config directory:
- Windows: `%USERPROFILE%\.hhq\sales-helper\`
- macOS / Linux: `~/.hhq/sales-helper/`

Check:
1. **`config.json` exists** with offer + ICP populated. If not → route to onboard-user.
2. **`current-batch.json` exists** with at least one prospect. If not → route to surface-next-5.
3. **`contacts-master.csv` exists.** If not → route to ingest-contacts (very unlikely to hit this branch — surface-next-5 would have caught it — but be defensive).

If checks pass, briefly tell the user what's about to happen so they're not staring at a silent screen for 5 minutes:

> "Got it — researching 5 prospects and drafting openers. This takes a few minutes; I'll work through them one at a time and show you everything when I'm done."

Then start the per-prospect loop.

## Phase 1 — Per-prospect loop (sequential, one at a time)

For each prospect in `current-batch.json`, do Phases 2-4. Sequential is fine for V1 — 5 prospects × ~60-90s each is acceptable. Don't try to parallelise.

After each prospect, give a brief progress note in chat ("✓ 1/5 — Greg Coleman done. Next: Marina Park."). No emoji except the check mark for progress. Keeps the user oriented during the wait.

If a single prospect's research fails (page gone, rate-limit, network error), DO NOT crash the whole batch. Mark that prospect's research.md with `Research failed: <reason>`, attempt the opener using only CSV data (it'll be more generic — flag this in the message log), and move on to the next. Report failures in the final summary.

## Phase 2 — Research a single prospect (Chrome connector, narrow scope)

For the current prospect:

### Step 2a — Navigate to LinkedIn profile

Use the Chrome connector tools (`mcp__Claude_in_Chrome__navigate`, then `mcp__Claude_in_Chrome__read_page` or `mcp__Claude_in_Chrome__get_page_text`).

URL: prospect's `url` field from `current-batch.json`. If empty/missing, fall back to a LinkedIn search by name+company — but accept that this is less reliable.

What you read from the profile:
- Current role + tenure (when did they start the current role?)
- Previous role (recency of any change)
- Location
- "About" section — only if it gives offer-relevant context
- Featured / pinned content — if any

Do NOT read: skills section, endorsements, recommendations, education history beyond current/latest, profile photo, banner. None are useful for the opener and they bloat context.

### Step 2b — Navigate to their recent activity / posts

URL pattern: `<profile-url>/recent-activity/all/` (LinkedIn's standard activity feed).

Read the most recent 5-10 posts (not all of them). For each:
- Date posted
- Post body (full text — these are usually short)
- Post type (their own / repost / comment)

If they have no public activity → note that ("No recent posts visible publicly").

### Step 2c — Analyse and write research.md (hollow seam A)

This is **seam A**. In V2 this becomes a server-side MCP call returning a structured analysis. In V1 you do the analysis yourself.

Heuristics for analysis:

- **Recency cliff**: a post in the last 7 days is hot. 7-30 days is warm. 30-90 days is lukewarm. >90 days is cold.
- **Topical hits**: does any recent post touch on the user's offer space? Be specific — "post about scaling sales teams" matches a sales-coach offer, "post about a recent funding round" matches a B2B SaaS offer. Generic "leadership lessons" posts are weak signals.
- **Role-change signals**: did they start their current role recently (last 3 months)? New roles = high openness to vendor conversations. Long tenure (>3 years) = harder to dislodge from their current setup.
- **Shipping signals**: did they announce something they built / shipped / launched? Strong hook.
- **Vulnerability signals**: did they post about a problem they're trying to solve? Strongest possible hook — they've publicly named the pain.

Write `~/.hhq/sales-helper/contacts/<slug>/research.md` using this exact structure:

```markdown
# Research: <Full Name>
*Researched <ISO timestamp>*

## Profile snapshot
- **Role:** <current title> at <current company> (since <start date or "tenure unclear">)
- **Location:** <city/country if visible, else "not visible">
- **Previous role:** <if visible, one line>

## Recent activity
- **Most recent post:** <date> — <one-sentence summary>
- **Recent themes:** <one or two themes across the last 5-10 posts>
- **Activity level:** <hot / warm / lukewarm / cold / silent>

## Signals matched
- <signal name from user's weighted signals>: <one-line evidence, or "not present">
- <signal name>: <evidence or "not present">
- <signal name>: <evidence or "not present">

## Hook for opener
<one sentence — the specific thing the opener should reference. This is the most important field. Pick the strongest, most specific signal. Avoid generic hooks.>

## Notes for the draft
<anything that should shape the opener — sensitivities, caveats, why this person is a softer/harder ask than usual>
```

If you couldn't get a profile read at all, the file body is just:

```markdown
# Research: <Full Name>
*Researched <ISO timestamp>*

**Research failed:** <reason>

Falling back to CSV-only data for the opener. Expect a more generic draft.

## CSV-known facts
- Role: <position from CSV>
- Company: <company from CSV>
- Connected on: <date>
```

Create the directory `~/.hhq/sales-helper/contacts/<slug>/` if it doesn't exist. If `research.md` already exists from a prior surface (re-surfaced after the 30-day cooldown), **append** a new dated section rather than overwriting — preserve the historical research as context.

### Step 2d — Create or preserve notes.md

If `~/.hhq/sales-helper/contacts/<slug>/notes.md` does NOT exist, create it as a placeholder for the user:

```markdown
# Notes: <Full Name>

*Your own notes go here. Examples:*
*- "Greg's away until Jan, come back then."*
*- "Met at the SmallSat conference 2025."*
*- "Don't pitch testing — they have an in-house lab."*
```

If notes.md already exists, leave it completely untouched. The user owns this file.

## Phase 3 — Draft a Greg-style opener (hollow seam B)

This is **seam B**. In V2 this becomes a server-side MCP call returning the opener text. In V1 you do the drafting yourself using the methodology below.

### What a Greg-style opener IS

- **3-5 sentences max.** Often shorter is better.
- **Opens with a specific reference** — a post, a milestone, a role change, something they shipped. NOT "I noticed you..." or "I came across your profile..." or "Hope you're well!"
- **States offer relevance in one line** — what you do, why it connects to *their specific situation*. Not a pitch, not a value prop deck.
- **Soft ask** — "happy to chat", "lmk if useful", "no pressure". Never "let's set up a call" or "15 minutes of your time".
- **Reads like a smart peer noticed something** — not like an SDR running a sequence.

### What a Greg-style opener IS NOT

- ❌ "Hi Greg, hope you're well!" — generic greeting filler
- ❌ "I help small satellite companies with microgravity testing." — generic value prop, not personal
- ❌ "I came across your profile and was impressed by..." — fake personalisation
- ❌ "Would love to set up 15 minutes to discuss how we can help you." — pushy ask
- ❌ Any emoji
- ❌ Any "!" exclamation (unless replicating user's voice and they use them)
- ❌ Buzzword stack ("synergy", "leverage", "scale", "transform")
- ❌ Long preamble before the actual point

### Structure (3 lines, often less)

1. **Specific opener** — refers to the hook from the research
2. **Bridge to relevance** — one sentence connecting their thing to your offer
3. **Soft ask** — leave the door open, don't push

### Examples

**Good** (illustrative, made up):
> Hey Greg — saw your post about the attitude control work you're shipping next quarter. We do microgravity component validation at Sunburnt Space; if you're after a final shake-down before launch, happy to chat. No pressure either way.

**Good** (a "they shipped something" hook):
> Hey Marina — congrats on the launch of FlightDeck. The demo video looks slick. We work with founders on outbound right around your stage — if you're thinking about a more deliberate go-to-market motion, lmk and I'll send something useful.

**Good** (a "recent role change" hook):
> Hey Tim — saw you joined Gilmour Space as Senior Propulsion Engineer last month, congrats. We work with propulsion teams on flight-readiness testing in microgravity; if it's relevant for your roadmap I'd love to compare notes. No pitch unless asked.

**Bad** (avoid this):
> Hi Greg, hope you're well! I saw you're a Founder at Magnetorquer. I help small satellite companies with microgravity testing solutions. Would love to set up a quick 15-minute call to discuss how Sunburnt Space can help you scale your operations. Let me know what works in your calendar!

### Drafting process

1. Re-read the **Hook for opener** field from research.md.
2. Re-read the user's `offer` from config.json.
3. Draft using the structure above.
4. **Self-edit pass:** check it against the IS / IS NOT lists. Cut anything weak. If it could have been written about anyone, rewrite — it must reference *this specific prospect's situation*.

### If the research failed

Use the CSV-known facts. Be honest in the draft — don't fabricate posts or activity. The opener will be more generic. Example fallback:

> Hey Tim — noticed we connected a while back. We work with propulsion teams on flight-readiness testing — if it's relevant for your work at Gilmour, happy to share more. Otherwise no pressure.

### Append to messages.md

Write or append to `~/.hhq/sales-helper/contacts/<slug>/messages.md`:

If creating new:

```markdown
# Messages: <Full Name>

## <ISO date> — Opener (LinkedIn)

<the drafted opener text>

---
```

If file already exists, append a new `## <ISO date> — Opener (LinkedIn)` section + body + `---` separator at the bottom. Preserve all prior entries.

## Phase 4 — Update master after each prospect

For the prospect just drafted, update `contacts-master.csv`:
- `status = drafted`
- `last_surfaced_date` stays as set by surface-next-5 (don't re-stamp)

Atomic write via temp-file-rename, same pattern as ingest-contacts.

Do NOT mark them `contacted` — V1 has no automated send-tracking. The user will manually update statuses (or we add that skill in V2).

## Phase 5 — After all 5 are done — clear the batch and present

Once the loop completes:

### Delete or empty current-batch.json

The batch is processed. Delete `~/.hhq/sales-helper/current-batch.json` (or overwrite with `{"version": 1, "prospects": []}`). This way the next `surface-next-5` call doesn't see a stale batch.

### Present all openers in chat

Show all 5 openers cleanly so the user can copy them. Format:

```
All done. Here are your 5 openers — copy, tweak if you want, send from LinkedIn:

═══ 1. Greg Coleman — Founder, Magnetorquer Pty Ltd ═══

<opener text>

LinkedIn: <url>
Files: ~/.hhq/sales-helper/contacts/coleman-greg-magnetorquer/

═══ 2. Marina Park — CEO, FlightDeck ═══

<opener text>

LinkedIn: <url>
Files: ~/.hhq/sales-helper/contacts/park-marina-flightdeck/

... etc for all 5
```

If any prospect failed research, mark it clearly in their block:

```
═══ 4. Tim Reyes — Senior Propulsion Engineer, Gilmour Space ═══

⚠ Research failed (rate-limited). Opener uses CSV data only — more generic than usual.

<fallback opener text>

LinkedIn: <url>
Files: ~/.hhq/sales-helper/contacts/reyes-tim-gilmour/
```

### Close with a short status nudge

> "When you've sent any of these, you can manually mark them `contacted` in `~/.hhq/sales-helper/contacts-master.csv` — V1 doesn't track sends automatically. They're at `drafted` for now and will stay out of the next surface for 30 days regardless.
>
> When you're ready for the next 5, just say 'get me the next 5'."

## Things you must NOT do

- Do NOT do anything beyond the prospect's profile + recent posts. No company page deep-dive, no news search, no LinkedIn graph traversal. Narrow.
- Do NOT log into LinkedIn or attempt any authenticated actions. Public profile reads only.
- Do NOT fabricate posts, role history, or anything that wasn't actually visible. If research is thin, say so honestly.
- Do NOT overwrite `notes.md`. Ever. The user owns it.
- Do NOT overwrite prior entries in `messages.md`. Always append.
- Do NOT overwrite prior `research.md` content on re-surface. Append a new dated section.
- Do NOT mark prospects `contacted` — V1 has no send tracking.
- Do NOT use emoji in opener drafts.
- Do NOT use exclamation marks in opener drafts (unless mirroring the user's voice in a future tier).
- Do NOT pitch in the opener. Soft ask only.
- Do NOT promise to "send a follow-up" automatically — V1 has no follow-up automation.
- Do NOT call any external API beyond the Chrome connector tools.
- Do NOT write outside `~/.hhq/sales-helper/`.
- Do NOT touch `config.json`.

## Edge cases to handle gracefully

- **Profile is private / "Out of network"** → research fails, fall back to CSV data, flag in the opener block.
- **No recent posts** → activity level = silent, hook from role/company match instead. Be honest in the opener — no fake "saw your recent post" reference.
- **Profile URL is wrong / 404** → mark research failed, fall back to CSV data, suggest the user verify the LinkedIn URL.
- **Prospect's company in CSV differs from current LinkedIn profile** (they changed jobs since the export) → research finds a new company. Update the master with new company + position. Use the new company in the opener.
- **Rate-limited mid-batch** → mark remaining prospects as research-failed, fall back, finish the batch. Don't retry mid-flight (it'll just re-rate-limit).
- **Single-name profile** (no last name) → slug uses what's available, e.g. `cher--cher-records`. Don't crash.
- **Two prospects in batch with the same name** (unlikely but possible) → slugs are already different from ingest-contacts (collision suffix), so files don't collide. Just process both normally.
- **`current-batch.json` is malformed** → tell the user and route to surface-next-5 to regenerate.
- **User interrupts mid-batch** ("stop", "pause", "wait") → finish the current prospect cleanly (don't leave half-written files), then stop. The batch.json still reflects the original 5; the user can re-trigger to resume processing the unfinished ones (you can detect which prospects already have a recent research.md and skip those).
