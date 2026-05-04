---
name: draft-reply
description: Admin Helper — drafts a reply to a Gmail thread in the user's voice and pushes it to Gmail as a reply-in-thread draft. The user identifies the thread (by natural-language description like "Sarah's pricing email", a Gmail URL, or a thread ID), the skill reads the thread via the HHQ Gmail MCP, drafts a reply that references the conversation specifically, and pushes the draft to Gmail. User reviews + sends in Gmail's own UI — Helper HQ never sends. Triggers on "draft a reply to <thread>", "reply to Sarah's email", "draft response to the pricing thread", "let's reply to that", "/hhq:draft-reply <reference>". Requires extended Gmail access (HHQ OAuth) — routes the user to /hhq:connect-gmail if not connected. Reuses the user-level voice profile (same one Sales Helper uses).
---

# Draft Reply — Admin Helper

You are drafting a reply to one specific Gmail thread the user wants to action. Identify the thread, read it, write a short reply in the user's voice that references what was actually said, push it to Gmail as a draft, and tell the user to review + send in Gmail.

This is a **single-thread, single-draft** skill. For batch follow-up workflows on Sales Helper conversations, the user wants `surface-followups`. For inbox triage / cleanup, they want `triage-inbox`. This skill is for "I have one specific email I want to reply to right now."

## When this skill runs

Trigger on:

- "draft a reply to `<thread reference>`"
- "reply to `<sender>`'s email"
- "draft a response to the `<topic>` thread"
- "let's reply to that" (when the user has just been looking at an inbox item)
- "write back to `<sender>`"
- "/hhq:draft-reply `<reference>`"

Don't trigger on:
- "send X to Y" — Helper HQ doesn't send. Tell the user to open Gmail directly.
- Generic "what should I say" — that's a coaching question, not a draft request.

## Phase 0 — Auth + Gmail connection check

### Step 0a — Get the project folder

Use `mcp__ccd_directory__request_directory`. Save as `<project-dir>`. Fallback `~/.hhq/sales-helper/`.

### Step 0b — Resolve auth

Read `<project-dir>/.hhq-session.json`. Refresh-on-near-expiry pattern. Halt with `/hhq:connect` if missing.

### Step 0c — Confirm extended Gmail access

`GET <backend_url>/api/me/gmail/connection`.

- `connected: true` → continue.
- `connected: false` → halt:
  > "Drafting uses Helper HQ's direct Gmail integration (extended Gmail access — beta). You're not connected yet. Run `/hhq:onboard` to opt in, or `/hhq:connect-gmail` if you already have approval."

This skill **does not work** with Cowork's generic Gmail connector — that connector exposes `create_draft` but not `get_thread` (full bodies) in the shape we need. Extended Gmail access is the only path.

## Phase 1 — Identify the thread

Three ways the user can refer to a thread, in priority order:

### 1. Direct thread ID

If the user pasted something that looks like a Gmail thread ID (a lowercase alphanumeric string like `198d3a2b9f4c7e5a` or with prefix like `t-abc123`), use it directly. Skip search.

### 2. Gmail URL

If the user pasted a Gmail URL like `https://mail.google.com/mail/u/0/#inbox/198d3a2b9f4c7e5a` or `https://mail.google.com/mail/.../FMfcgz...`, extract the trailing alphanumeric segment as the thread ID. Use it directly.

### 3. Natural-language description

The most common case. The user says "Sarah's pricing email" or "the thread about the proposal" or "Greg's last message". Search the inbox for matches:

`POST <backend_url>/api/mcp/gmail/list_inbox` with `{"max_results": 50, "query": "<the user's description, transformed>"}`.

Transform the description to a Gmail search query:
- Sender names → `from:<name or email>` (if you can resolve the name to an email via `/api/me/contacts`, prefer the email)
- Topic keywords → bare terms
- "last week's" → add `after:<date>` if appropriate
- "unread" → `is:unread`

Examples:
- "Sarah's pricing email" → if Sarah is a contact → `from:sarah@acme.com pricing`; otherwise `from:Sarah pricing`
- "the proposal thread" → `proposal`
- "Greg's last message" → `from:greg@magnetorquer.com`

If the search returns:
- **Zero threads** → "No matching thread in your recent inbox. Could you give me a more specific reference, or paste the Gmail URL?"
- **One thread** → use it. Tell the user briefly: "Found it — Sarah Khan's email about pricing from Tuesday."
- **Multiple threads** → render the top 3-5 with sender + subject + date and ask: "Which one do you want to reply to? (1 / 2 / 3 / cancel)"

## Phase 2 — Read the thread

`POST <backend_url>/api/mcp/gmail/get_thread` with `{"thread_id": "<the id>"}`.

Response has the full thread including message bodies. The body data is base64url-encoded in `messages[].payload.body.data` — decode it. For multipart messages, walk `payload.parts` and pick the `text/plain` or `text/html` body.

**Privacy contract:** the body data goes into your context for drafting. Do NOT echo full message bodies back to the user verbatim, do NOT save bodies anywhere — they live only in your draft prompt for this single invocation. Same rule as `surface-followups`.

## Phase 3 — Read the user's voice + sender context

### Step 3a — Voice profile

`GET <backend_url>/api/me/config` — pull `config.voice_profile`. Same shape Sales Helper uses:

```json
{
  "summary": "Direct, plain, no jargon. Single-line questions. Curious not selling.",
  "tone": ["direct", "warm", "curious"],
  "do": ["Use first names", "Ask one short question"],
  "dont": ["Use the word leverage", "Exclamation marks"],
  "phrases": ["Hey <Name> — saw <thing>. <observation>?"]
}
```

If `voice_profile` is null or missing, draft in a generic professional tone but tell the user once at the end:
> "Heads up — your voice profile isn't set up yet. The draft above is in a generic tone. Run `/hhq:tune-voice` to teach me how you actually write."

### Step 3b — Sender context (optional but valuable)

Parse the sender email from the latest inbound message's `From` header. Look up via `/api/me/contacts?per_page=500` (or a more targeted endpoint if available) — find the contact whose `email` matches.

If found, surface lightweight context to the drafting step:
- The contact's `first_name`, `company`, `position`
- Whether they're a `prospect` / `customer` / `partner` / etc. (`relationship_type`)
- Their pipeline stage (if prospect)
- Their `contact_profile` (rolling dossier from v0.13) — this is the most valuable signal, gives history

If not found, just use the display name from the email header. Continue without enriched context.

## Phase 4 — Draft the reply

You are writing a short, specific reply in the user's voice that references what the other person actually said.

### Hard rules for the draft

- **Reference the conversation specifically.** No generic "thanks for reaching out" if they asked a specific question — answer it. Quote or paraphrase the exact thing they said you're responding to.
- **Match the user's voice.** Use the `do` / `dont` / `phrases` from the voice profile. If the profile says no exclamation marks, no exclamation marks.
- **Match the conversation's register.** A long detailed thread gets a substantive reply; a one-line "thanks" gets a one-line reply back. Don't write a wall of text in response to "ok sounds good."
- **Default short.** 2-4 sentences for typical replies. Longer only if the other person asked a specific multi-part question that warrants it.
- **Sign off the way the user signs off in their voice profile.** If unspecified, use a single-name signoff: "Brad" — match what their existing sent messages would look like.
- **NO subject prefix in the body** — the backend's `push_draft` handles `Re:` automatically. Don't write "Subject: ..." in the body.

### What you have access to

- The full thread (all messages, all bodies) — via Phase 2.
- The user's voice profile — via Phase 3a.
- The sender's contact context — via Phase 3b (if known).

### What you DON'T have access to

- The user's calendar (no scheduling).
- The user's pricing / contracts / specific commitments (don't invent figures or dates).
- Anything that isn't in the thread or the contact's dossier.

If the reply naturally needs information you don't have ("can you confirm the price?"), insert a placeholder in `<<>>` brackets and flag it to the user:

> "Hi Sarah,
>
> Thanks — Tuesday at <<TIME>> works for me. Let's go with the <<PROPOSAL VERSION>> draft we discussed.
>
> Brad"
>
> *Heads up: I left two placeholders for you to fill in (time, proposal version). Replace those before sending.*

## Phase 5 — Show the draft

Render the draft to the user with light context:

```
**Draft reply to Sarah Khan (Re: Pricing for 50 seats):**

Hi Sarah,

The 50-seat tier lands at $X/year — same per-seat rate as the
20-seat tier, no discount jump until 100. Happy to walk through
the rollout sequence on a call if it helps; my Calendly is in
my signature.

Brad

---
**Approve?** (yes — push to Gmail / edit / cancel)
```

Notes:

- Show the draft inside a clear block. The block ends with `---` and the action prompt.
- Mention the recipient + subject so the user knows context.
- If you used placeholders, surface them above the action prompt: "*Placeholders: <<TIME>>, <<PROPOSAL VERSION>> — replace before sending.*"

## Phase 6 — Handle the user's response

### Yes / approve / "push it"

`POST <backend_url>/api/mcp/gmail/push_draft` with:

```json
{
  "thread_id": "<the id>",
  "body": "<the draft body verbatim — no leading/trailing whitespace>"
}
```

Response includes the draft ID and the message it was attached to.

#### Auto-relabel: To Do → Awaiting Reply

Once the draft is pushed, check whether the thread carries the user's `to_do` label. The thread's `label_ids` came back in `list_inbox` (if you searched there in Phase 1) or you can read them via `get_thread`'s `messages[].labelIds` (top-level on the thread itself isn't always populated — fall back to the latest message's labels).

If the thread is currently labelled `to_do` (i.e. its label_ids include the user's `to_do` `gmail_label_id` from `/api/me/gmail/labels/config`), then:

1. `POST /api/mcp/gmail/unlabel_thread` with the `to_do` `gmail_label_id`.
2. `POST /api/mcp/gmail/label_thread` with the `awaiting_reply` `gmail_label_id`.
3. If the `awaiting_reply` entry has `archive_on_apply: true`, also `POST /api/mcp/gmail/archive_thread` so it leaves the inbox.

Skip the relabel if either label isn't set up (entry missing from `/api/me/gmail/labels/config`), the thread isn't currently `to_do`, or the user explicitly said "leave the labels alone" during edit. Don't surface label setup errors to the user — the draft was the primary action and it succeeded.

#### Confirm to user

> "Draft pushed to Gmail — open the thread in Gmail to review and send.
>
> *Moved from To Do to Awaiting Reply.* (only if the relabel actually ran)
>
> Reminder: Helper HQ never sends on your behalf. The draft is sitting in your Drafts folder, threaded with the original conversation. Click send when you're ready."

### Edit / "change X"

Conversational. Accept:
- "Make it shorter" → re-draft, render, ask again.
- "Drop the second paragraph" → re-draft.
- "Tone is too casual / formal" → adjust voice and re-draft.
- "Add Y" → add and re-draft.

Re-render and ask "Approve? (yes / edit / cancel)" again. No limit on iterations — the user might iterate 2-3 times.

### Cancel / no

Don't push anything.
> "No draft pushed. Nothing changed in Gmail. Run again any time."

## Things you must NOT do

- Do NOT send the email. Helper HQ never sends. Always pushes to drafts. The user opens Gmail and clicks send.
- Do NOT echo the full message body back to the user verbatim. They can see it in Gmail. Render only the DRAFT in Phase 5, not the source thread.
- Do NOT persist the message body anywhere. It's in your context for this skill invocation only. Don't write it to the project folder, don't POST it to the backend, don't store it. Same v0.13 privacy contract as `surface-followups`.
- Do NOT invent facts the user hasn't given you. Prices, dates, commitments, names of people not in the thread — placeholder them with `<<>>` brackets and flag.
- Do NOT add a subject line to the body. The backend's `push_draft` constructs the subject (auto-prefixes `Re:` if needed). Body is body only.
- Do NOT run on the Cowork generic Gmail connector. Phase 0c gates on extended Gmail access. Fail honestly if missing.
- Do NOT modify `<project-dir>/.hhq-session.json` except for JWT refresh.
- Do NOT call `push_draft` more than once per skill invocation. If the user wants a different version, edit and re-draft until they approve, THEN push once.

## Edge cases to handle gracefully

- **Thread is from the user themselves only (sent folder, no inbound)** → "This thread is your outbound only — no one to reply to. If you meant to follow up on someone who hasn't replied, run `/hhq:followups` instead."
- **Thread has only one message, from someone else, but they didn't ask anything** → Draft a brief acknowledgement. Surface to the user: "*Their message didn't ask anything specific — drafted a short acknowledgement. Edit if you want something more substantive.*"
- **Voice profile is missing** → draft in a generic tone but tell the user at the end (per Phase 3a).
- **Sender context lookup fails** → drop the enrichment and draft from the thread alone. Don't block the draft on contact backend issues.
- **Push_draft returns an error** → surface the error code + message. Common: 502 `gmail_api_error` (Gmail rate limit / network) → suggest retry in a minute. 409 `gmail_revoked` → tell the user to re-run `/hhq:connect-gmail`.
- **Thread is enormous (50+ messages, e.g. mailing-list digest)** → focus the draft on the most recent inbound message that's clearly addressed to the user. Don't try to summarise the whole digest.
- **The user typed an obviously wrong thread ID (random characters)** → `get_thread` returns 502 with a 404 from Gmail. Surface honestly: "Gmail can't find a thread with that ID. Could you double-check the URL or describe the thread instead?"
