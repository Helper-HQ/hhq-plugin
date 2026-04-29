---
name: tune-voice
description: Manage the user's voice profile after onboarding. Two modes — (a) base voice (default) acts on `voice_profile` at user level via `/api/me/config`; (b) campaign-additions mode acts on `voice_additions` for the current campaign via `/api/me/campaigns/{slug}/config`, additive on top of the base. Triggers — base mode on "tune my voice", "update my voice", "show my voice", "regenerate my voice"; campaign-additions mode on "tune this campaign's voice", "add to campaign voice", "tune voice for this campaign", or any phrase mentioning "this campaign" alongside voice. Both modes display the profile, then offer (1) view only; (2) edit do/dont/tone/phrases/summary directly; (3) add a new source (URL, paste, PDF/DOCX) and re-synthesise; (4) regenerate from scratch. Synthesis logic mirrors onboard-user Phase 6.
---

# Tune Voice — Sales Helper Lite

You are managing the user's `voice_profile` outside the onboarding flow. This skill is for **ongoing** voice tuning — adding new samples, removing rules that aren't them, refining tone over time.

The synthesis logic mirrors `onboard-user` Phase 6. Read that skill if you need a reference for the synthesis pass — they share the same shape and rules.

## When this skill runs

Trigger when the user says:
- "tune my voice", "update my voice", "refine my voice"
- "show my voice", "view my voice", "what's my voice"
- "regenerate my voice", "redo my voice"
- "add <thing> to my voice" (URL, article, message, file)
- "remove <thing> from my voice"

Do NOT trigger if `~/.hhq/machine.json` is missing (and no legacy `<project>/.hhq-auth.json` to migrate from) — route to `onboard-user` first ("No auth — say 'set me up' to onboard.").

## Phase 0 — Auth, mode, campaign

### Step 0a — Resolve auth (machine-level)

**Per-machine folder access.** First, call `mcp__ccd_directory__request_directory({"path": "~/.hhq"})` to request Cowork access to the per-machine auth folder. The user sees a one-time prompt per Cowork project; approve once and `~/.hhq/` is readable for the rest of the project's history. If declined, set `home_hhq_unavailable = true` and fall back to reading per-project `<project-dir>/.hhq-auth.json` instead. If the tool is unavailable (CLI), treat as approved.

Then read `~/.hhq/machine.json` (or `<project-dir>/.hhq-auth.json` if `home_hhq_unavailable`).

- **Found** → parse `backend_url`, `license_key`, `machine_id`, `jwt`, `jwt_expires_at`. Continue.
- **Not found, but `<project-dir>/.hhq-auth.json` exists** → legacy file from before v0.10. Migrate inline: `mkdir -p ~/.hhq`, copy → `~/.hhq/machine.json`, delete legacy. Continue.
- **Not found, no legacy file** → tell the user and stop.

If `jwt_expires_at` is past or within 60s of expiry, run the standard refresh / re-activate fallback (see `ingest-contacts` Phase 0 for the exact protocol). Save token updates to `~/.hhq/machine.json`.

All API calls below use `Authorization: Bearer <jwt>` and `curl -sk`. Never log the JWT or licence key in chat.

### Step 0b — Determine mode (user-base vs campaign-additions)

Decide `mode` based on how the skill was triggered:

- Phrases mentioning "this campaign" / "for this campaign" / "campaign voice" / "campaign-specific voice", or explicit `--campaign` argument → `mode = "campaign"`.
- All other voice phrases ("tune my voice", "update my voice", "show my voice", etc.) → `mode = "user"`.

If ambiguous (e.g. just "tune voice" while the project is pinned to a non-default campaign), ask once:

> "Tune your **base voice** (used everywhere) or this campaign's **voice additions** (layered on top of the base, just for this campaign)? (base / campaign)"

### Step 0c — Resolve current campaign (if mode = "campaign")

Use `mcp__ccd_directory__request_directory` to get the project folder. Read `<project-dir>/.hhq-campaign.json` to get `campaign_slug`. If missing or empty → tell the user this project isn't pinned to a campaign yet. Suggest `/hhq:new-campaign` or running base-voice tuning instead. Stop.

### Step 0d — Set the target endpoint and field

- **mode = "user"** → endpoint `/api/me/config`, field `voice_profile`.
- **mode = "campaign"** → endpoint `/api/me/campaigns/<campaign_slug>/config`, field `voice_additions`.

Throughout the rest of the skill, references to "the profile" mean the value at `<endpoint>.<field>`.

## Phase 1 — Load current voice

`GET <endpoint>` (the endpoint chosen in Step 0d).

Read the relevant field — `voice_profile` (user mode) or `voice_additions` (campaign mode).

**If the field is null or missing:**

User mode:
> "You don't have a base voice profile yet. Want to build one now? I'll ask for a brand guide, articles, or LinkedIn message URLs — same as the voice step in onboarding. (yes / no)"

Campaign mode:
> "This campaign doesn't have any voice additions yet. Most campaigns don't need any — your base voice is fine on its own. Want to add some? Examples: 'lean more formal', 'mention investor track record', 'never start with a first name', specific phrases worth echoing. (yes / no)"

If yes → jump to Phase 3 ("Add sources and synthesise") with no existing profile to merge into. In campaign mode, the user can also just paste a few notes and skip source-fetching entirely.
If no → stop politely.

**If the field exists:**

Display it using the format below (Phase 2). Then go to Phase 2.

## Phase 2 — Display + pick action

Show the voice profile cleanly:

```
Your voice profile (last updated: <generated_at>)

Summary
  <summary>

Tone
  · <tone item> · <tone item> · ...

Do
  · <do item>
  · <do item>
  · ...

Don't
  · <dont item>
  · <dont item>
  · ...

Sound check
  <first phrase>

[Sources: N articles, M LinkedIn messages, brand_guide=<text|pdf|docx|url|null>]
```

Then ask:

> "What do you want to do? (edit / add source / regenerate / done)"

- **edit** → Phase 2a (direct list edits)
- **add source** → Phase 3 (gather + synthesise into existing profile)
- **regenerate** → Phase 4 (re-synthesise from existing sources, no new input)
- **done** → save (if anything changed) and close warmly

Accept natural-language equivalents: "let me edit", "add an article", "redo it", "looks good", etc.

### Phase 2a — Direct edits

The user can speak edits in plain language. Apply each one immediately and re-show the profile after each change. Examples:

- *"Remove the 'leverage' rule"* → string-match in `dont` array, remove the matching item.
- *"Add: never end with 'looking forward to hearing from you'"* → append to `dont` array verbatim.
- *"Change the summary to: warm, plain, slightly dry"* → replace `summary` verbatim.
- *"Take 'curious' out of the tone"* → remove from `tone` array.
- *"Add 'use first names' to the dos"* → append to `do` array.
- *"Drop the second sound check phrase"* → remove `phrases[1]`.

After applying, re-show the profile (Phase 2 format) and ask "Anything else? (edit / add source / regenerate / done)".

If a string match fails (the user references something not in the profile), tell them honestly: "I don't see 'X' in your current voice — want me to show it again so you can pick the exact wording?"

After 5 edit rounds without "done", nudge: "We can keep tuning, or save what we've got and refine later. (continue / save and done)"

When user says **done** → go to "Save" step at the bottom.

## Phase 3 — Add a source and re-synthesise

> "What's the new source? Paste text, drop a URL, or attach a PDF or Word file."

Accept any of:
- **Pasted text** → hold as new `brand_guide_text` (or merge with existing if they say "this is brand guide" / "this is an article").
- **URL** — distinguish: article URL, LinkedIn message URL, brand-guide URL. Ask once if ambiguous: "Is this an article, a LinkedIn message, or your brand guide?"
- **Attached file (`.pdf` / `.docx` / `.txt`)** — read inline using the `pdf` / `docx` skill or direct read. Extract text. Treat as brand_guide_text by default unless the user specifies it's an article.

Hold the new source in skill memory.

### Step 3.1 — Synthesise (incorporating existing profile)

Tell the user:

> "Reading and merging into your existing voice — back in 1-2 minutes."

Fetch the new source:
- **URL** → `WebFetch`, fall back to `mcp__Claude_in_Chrome__navigate` + `read_page` if blocked.
- **LinkedIn message URL** → `mcp__Claude_in_Chrome__navigate` + `read_page` (logged-in session).
- **Text / file content** → already in hand.

Now distil. **Important:** don't replace the existing profile — merge into it. The existing profile represents accumulated tuning; new sources should refine, not overwrite.

Merge rules:
- **summary** — only revise if the new source clearly shifts the tone description; otherwise keep existing.
- **tone** — add new tone words if genuinely new; keep existing.
- **do / dont** — add new items the new source surfaces; keep existing items unless they directly contradict the new source.
- **phrases** — add 0–1 new sound-check phrases from the new source if they're better than the existing ones.
- **sources** — append the new URL / mark new brand_guide source.
- **generated_at** — bump to now.

If the fetch fails, tell the user honestly and don't crash. Ask if they want to try a different source or move on.

### Step 3.2 — Show + tune

Display the merged profile (Phase 2 format) and run Phase 2a edit loop.

When user says **done** → Save.

## Phase 4 — Regenerate from existing sources

This rebuilds the profile from scratch using the URLs already listed in `voice_profile.sources` — useful if the user has been editing manually for a while and wants to start fresh, or if synthesis quality has improved since the last run.

> "Regenerating from your existing sources (`<N>` articles, `<M>` LinkedIn messages). Manual edits will be lost. Continue? (yes / no)"

If no → return to Phase 2.

If yes:
- Re-fetch all `sources.articles` and `sources.linkedin_messages`.
- If brand_guide source is `text` and the original text isn't in the profile (it shouldn't be — we don't persist source content), tell the user honestly and ask them to re-paste.
- Re-synthesise from scratch (no merge with existing — fresh profile).
- Show the new profile (Phase 2 format) and run Phase 2a edit loop.

When user says **done** → Save.

## Save

`PUT <endpoint>` (chosen in Step 0d) with the existing config + the updated field (`voice_profile` for user mode, `voice_additions` for campaign mode).

Do NOT touch any other config fields. Read the existing config first, splice in the new value for the target field, PUT the whole config.

```
PUT <endpoint>
Authorization: Bearer <jwt>
Content-Type: application/json

{ "config": <existing config with the target field replaced> }
```

Expected HTTP 200. On 401, run auth fallback once and retry. On 5xx / network error, tell the user honestly and don't lose their edits — keep them in conversation context for retry.

In campaign mode, only the campaign's voice_additions are updated — the user's base voice on `/api/me/config` is untouched.

## Close

> "Saved. Your voice is updated and the next opener I draft will use it. Say 'tune my voice' anytime you want to come back to this."

(In campaign mode, the close line should be: *"Saved. The next opener I draft for this campaign will layer these additions on top of your base voice. Other campaigns are unaffected."*)

## Things you must NOT do

- Do NOT save raw source content (page text, PDF/DOCX content, message bodies) anywhere. Distil and forget — same rule as onboard-user Phase 6.
- Do NOT save uploaded files to disk or backend. PDFs/DOCXs are read inline; the file itself is not persisted.
- Do NOT touch other config fields. This skill only writes `voice_profile` (user mode) or `voice_additions` (campaign mode) — never both, never anything else.
- Do NOT regenerate without confirming — manual edits are user work and shouldn't be silently overwritten.
- Do NOT log the JWT or licence key in chat.
- Do NOT loop forever in edit mode. After 5 rounds, nudge to save.

## Edge cases to handle gracefully

- **Profile is null** → offer to build one (jumps to Phase 3 with no merge).
- **All sources fail to fetch** during Phase 3 or 4 → tell the user honestly, return to Phase 2 with the original profile intact.
- **User asks to remove the last item in `tone`** → allow it (empty array is fine; profile is still useful).
- **User asks to clear the whole profile** → confirm explicitly: "That'll wipe your voice profile entirely. Sure? (yes / no)" If yes, set `voice_profile: null` and save. They can rebuild via this skill or onboarding.
- **Conflicting edits** (e.g. add X to don't, then add X to do) → flag once: "You have 'X' in both do and dont — keep both, or pick one?"
- **Brand guide was originally pasted text** — the original text is gone (we don't persist sources). Regeneration tells the user to re-paste.
