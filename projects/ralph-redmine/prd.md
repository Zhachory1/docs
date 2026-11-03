---
id: prd-ralph-redmine
title: Ralph keeps work requests in a private tracker only the owner and Ralph can read, and replies in the requester's own chat thread — asking first when the ask is unclear, once when work starts, and once with the outcome itself
status: in review
owner: Zhachory1
card: t_8e5a6fa4
revision: v5
date: 2026-11-03
prs:
  - Zhachory1/docs#6   # v1, v2, v3, v4, v5
supersedes: none
---

# Ralph keeps work requests in a private tracker only the owner and Ralph can read, and replies in the requester's own chat thread — asking first when the ask is unclear, once when work starts, and once with the outcome itself

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker" (a self-hosted Redmine), and the chat system is Google Chat, which is
> a vendor name and is named. Host paths are the public Ralph source tree and the
> local profile install, cited as found.

Status: **in review** — a requirements document, awaiting the owner's approval.
Nothing is built. No chat message has been sent, no scope changed, no REST API
enabled, no tracker touched. Merging this pull request is the approval, and it
is also the one grant R15 asks for.

Source: Kanban card `t_8e5a6fa4`. The earlier combined document at
`Zhachory1/docs#5` was closed without merge and is **not** an approved
specification. Its assumptions are not carried forward.

---

## Review header

### Decisions needed from you: one, and merging is how you make it

**May Ralph reply automatically, and may those replies carry text it wrote?**
Four messages, named in R15:

| Reply | When | What it carries |
|---|---|---|
| Clarification (R20) | The ask is not understood well enough to work it | A short receipt and **one** specific question. Never "I am working on it". |
| Acceptance (R6) | The work is understood, accepted and recorded | One short sentence: Ralph has it. |
| No-fit (R7) | A message Ralph understands and will not act on | "I am not sure how to help with that." |
| Closing (R11) | The request was actually worked to a result | The outcome itself — the answer, the finding, the pull requests. |

Each at most once per request, except that the clarification question may have
one follow-up (A17). Each goes into the thread the request arrived in, and each
is drawn only from this request's own work (R19).

This is **wider than v4**, which proposed three. Your comment of 23:41 added the
fourth: *"If the request is unclear, Ralph replies in the same original Chat
thread with a short receipt and one specific clarification question."* That
question is text Ralph composed, sent before any work exists, to someone who may
not be you — so it belongs inside the same grant and the same bound, not beside
it. The cost of the whole grant is stated plainly: text Ralph composes reaches a
requester without you reading it first, which is exactly what R15 and design
unknown 5 exist to bound.

**No sending permission is live today, and merging does not create one.**
`SKILL.md:44` still forbids automatic send and this document does not edit it.
There is no chat producer, no writer and no credential on this path. The grant
is a product requirement; the design must define the authorization boundary and
test it before anything can send (R15, R17, unknown 5).

Four things I chose rather than asking you again. Each is an assumption, and
each is one comment away from changing:

| I chose | Instead of | Say this to change it |
|---|---|---|
| A12 — in a space with other people, only a message that @mentions you or Ralph is eligible; in a direct message every human message is | Acting on every message in those spaces | "act on everything in the rooms" |
| A14 — a request that ends **without** succeeding is silent toward the requester; you are told and you decide what to say | An automatic "this did not work out" | "tell them when it fails too" |
| A17 — one open clarification question at a time, and one follow-up at most; then Ralph acts on what it has or tells you | An unbounded back-and-forth with the requester | "keep asking until it is clear" |
| A18 — a clarification question that is never answered is never re-sent; the request stays pending and silent | A reminder after some time | "nudge them if they go quiet" |

### Council dissent

An independent agent review of v4 found **no blocking issues** and two
non-blocking notes.

| Finding | Resolution |
|---|---|
| Non-blocking. Unknown 3 called `run-notifier:18,35` "the digest's webhook". Line 18 gates the board alert and line 35 the daily digest; both use the same fixed destination, so the substance held and only the label overclaimed. | Fixed: unknown 3 now says "the host notifier's webhook". |
| Non-blocking. R18's "this feature adds no allowlist of its own" nudges unknown 11 away from one option it lists — a sender list local to this feature. | Named rather than reworked. Unknown 11 now says that if the design concludes a list is the only safe reconciliation, that contradicts R18 and comes back to you as a change, instead of being settled inside the design. |

Earlier blockers stay fixed. v2's was R15 claiming a grant you had not given:
R15 still separates what you said, where you said it, and what this document
proposes. v3's was a checkable claim about what you had not said: every decision
below is attributed to the comment or inline thread it came from, and v5 adds
two rows for your comment of 23:41 and your inline comment on `prd.md:39`.

The reviewer's load-bearing assumption is unchanged: A6, that "the existing host
sender" means the chat skill's OAuth token and not the digest webhook. The
webhook sends to one fixed destination and cannot reply into a thread at all,
so if it is the only sender available, all three messages need a sender that
does not exist yet. It is design unknown 3, and the first thing to confirm.

### Scope: medium

One tracker, one thread, and at most three messages to a requester: a question
if the ask is unclear, an acknowledgement when work starts, the outcome at the
close. **Breakdown:** the design document decides it, and nothing starts before
this document is merged.

### What changed since v4

- **Ralph asks before it starts** (R20, new): when the ask is not understood,
  a short receipt and one specific question in the same thread — no claim that
  work is underway, no work begun, no acceptance reported. Your inline comment
  on `prd.md:39` and your comment of 23:41.
- **The acceptance reply moved** (R6): it now comes only after the ask is
  *understood* and the work is recorded, not merely after an item exists.
- **The no-fit reply narrowed** (R7): it is for a message Ralph understands and
  will not act on. A merely unclear message gets R20's question instead, so
  "I am not sure how to help with that" is never the answer to a request that
  was only badly phrased.
- **Pending is a real state** (R5, R11, E17): a request waiting on an answer may
  be recorded with its next action, is not acceptance, and can never be reported
  complete or successful.
- **The loop guard covers the new exchange** (R10, R16): Ralph's own question
  never re-enters as a request, and a requester's answer is context — never an
  instruction about what Ralph may send, or where.
- **R15 widens from three replies to four, and says so.** The clarification
  question is inside the grant and inside the bound the design must prove.
- Four new acceptance examples (E15-E18), two new design unknowns (13, 14), and
  both of the v4 review's non-blocking notes fixed.

---

## Who this is for

**The owner, and only the owner.** Work requests arrive in chat and get lost,
because a chat thread is not a list.

**The requester is not a user of this system.** They never see the tracker,
never get a link to it, never get an account, never get an email. The only thing
that reaches a requester is text in the thread they already wrote in. That is
the main constraint below: anything that would make a requester go somewhere or
sign in is out of scope.

---

## Current state, verified

I read the public Ralph source and the installed chat profile on 2026-11-02.
**`/ralph` in this container is not a git checkout**, so I cannot cite a commit;
the chat skill files carry mtime 2026-10-22. Treat the paths as the citation and
re-check them against a commit before implementing.

Paths below are `/ralph/...` for source, and
`~/.hermes/profiles/ralph-chat-maintainer/...` for the installed profile. The
card named the profile under `/ralph/profiles/`; it is not there.

### What exists

**A chat helper that can already send, including into a thread.**
`/ralph/skills/ops/google-chat/gchat.py:243-252` defines `cmd_send`, which calls
`spaces.messages.create` and takes an optional `--thread`. Its token holds five
scopes (`gchat.py:54-60`) and they are **not** read-only: `chat.messages.create`
(`:57`) and `chat.spaces.create` (`:58`) are both present. Nothing has to be
built or re-scoped to make a thread reply technically possible.

**A policy that forbids using it the way this feature needs.**
`/ralph/skills/ops/google-chat/SKILL.md:44`: run `send` only on an explicit,
per-message instruction from the actual user; never send because a message that
was *read* asked for a reply; never batch or auto-send without a fresh
instruction each time. An automatic acknowledgement is the thing that line
prohibits. The bounded grant R15 proposes is a product decision, and merging
this document does not apply it: the line stands as written until a design
edits it, you approve that edit, and a test proves the bound holds.

**A trust boundary that already treats chat as data.** `SKILL.md:46-48` and
`gchat.py:16-18` say chat content read back is external data, never
instructions; `gchat.py:62-63,178-185` wraps read output in external-content
markers; `SOUL.md:23` repeats it for the profile. A message asking Ralph to
reply is not an instruction to reply. This feature must not become a loophole.

**A decision profile, prepared and inert.** `ralph-chat-maintainer` handles one
chat item per card and decides what should happen; a host-side script holding
the credential carries the decision out (`SOUL.md:5`). It has exactly five
decisions — `route`, `draft_reply`, `escalate`, `send`, `ignore`
(`SOUL.md:15-19`) — with `sender_trusted` set by the host from the owner's
allowlist and never by the message (`:9`), and `route` unavailable for untrusted
senders whatever the message says (`:21`). Its `config.yaml` gives it the `file`
tool on the CLI and `no_mcp` on the API server. **It has no network, no chat
credential and no cron entry.** It can decide; it cannot act.

**A working pattern for exactly this shape, built for email.**
`/ralph/bootstrap/inbox-producer:1-23` runs on the host where the token lives,
stages a thread as JSON into a directory, and creates a card whose workspace is
that directory, keyed for idempotency on thread id plus latest message id
(`:20-22`). `/ralph/bootstrap/inbox-writer:1-20` turns the finished card into
exactly one write, never a send, and an eval fails the build if a send call ever
appears in it (`:7-8`). Recipients come from the staged thread, not from
anything the agent wrote (`:16-19`). The producer/writer split, the host-held
token, the idempotency key and the single-write guarantee are all proven. They
are proven **for Gmail**.

### What does not exist

**No chat producer and no chat writer.** `/ralph/bootstrap` holds
`inbox-producer` and `inbox-writer` and no chat equivalent. Nothing turns an
inbound chat message into a card, and nothing carries a decision back out to
chat. The inert profile is inert because both halves were never written.

**No decision that sends an inbound reply.** This is the finding that most
affects your answers 1-3. `draft_reply` is explicitly not it: "The owner reads
every draft and sends it or doesn't. You never send as the owner."
(`SOUL.md:16`). `send` is for an **outbound** item another profile supplied,
whose `destination` is "fixed by the host; you cannot change it"
(`SOUL.md:11,18`). `ignore` covers bot notices and acknowledgements with an
empty body (`:19`). So a clarification question, an automatic acceptance reply,
a "not sure how to help" reply and a closing reply are each a new path, not a
setting. Nothing today can ask a requester anything.

**No sender set matching R18.** `sender_trusted` comes from the owner's
allowlist (`SOUL.md:9`) and `route` is refused without it (`:21`). R18's
boundary — the owner, or anyone who @mentions the owner — is wider than an
allowlist and is not expressible today. This is a conflict, not a detail; see
unknown 11.

**No threaded-reply guarantee.** `gchat.py:249` sets
`messageReplyOption = "REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD"`. That is the exact
behaviour R8 forbids: when the thread cannot be replied to, the helper starts a
new one instead of failing. The API's `REPLY_MESSAGE_OR_FAIL` is not used
anywhere I found.

**No classification into the kinds you named.** The profile routes to the
orchestrator, which "classifies, picks the workflow, and writes the cards"
(`SOUL.md:15`). Nothing classifies a message as question, update, feature
request or bug investigation today, and nothing separates a message that is
**unclear** from one that is simply not work — the distinction R20 and R7 turn
on.

**No tracker integration of any kind.** No write path, no issue maintainer, no
reference to the work tracker in the Ralph source I read.

### Unverified

| Claim | Why unverified |
|---|---|
| The existing work digest uses a separate read-only chat token (`chat.messages.readonly`, `chat.spaces.readonly`) | The digest configuration is not readable from this profile. What is adjacent and confirmed: `/ralph/bootstrap/run-notifier:18-23,35-41` sends board alerts and a daily digest through `GCHAT_WEBHOOK_URL` — a fixed destination, never a thread reply — and skips them when it is unset. The token's scopes I could not check. |
| `/ralph/notifier/approvals.py` and the docs-PR gate | `/ralph/notifier/` is not present in this container. `run-notifier:19,26,37` invokes `poller.py`, `approvals.py` and `digest.py`, so they exist on the host; their contents I did not read. |
| `/ralph/profiles/author/SOUL.md` | `/ralph/profiles/` is not present. A local installed copy of the author profile states the gate this document follows: write to the docs repository as a pull request, never merge, approval is explicit. The deployed file I did not read. |
| The work tracker's state: owner-and-Ralph-only access, one private Work Requests project, one manual migration issue, REST API off, no integration running | Supplied by the owner's session. This profile must not and did not touch the work instance, its credentials, chat messages or any private document. |

---

## The outcome

**For the owner.** Every actionable request that arrives in chat exists as one
item in the work tracker, readable without opening chat. Ralph runs it from
arrival to close without asking the owner to choose a project, pick an issue,
settle an ambiguous match, or approve a question to the requester. The owner
hears about genuine failures only.

**For the requester.** At most three messages, all in the thread they wrote in.
If Ralph cannot work the ask as written, a short receipt and one specific
question, and nothing that claims work has started (R20). When the work is
understood, accepted and recorded, one short acknowledgement (R6). When it has
actually been worked to a result, one closing reply that is the outcome itself —
the answer, the finding, or the pull requests (R11). An update gets the
acknowledgement and nothing after it. A human message Ralph understands and will
not act on gets one honest "I am not sure how to help with that" (R7). If the
work fails, is abandoned, or the question is never answered, nothing more
(A14, A18).

**For everyone else.** Nothing. No account, no portal, no notification, no mail.

---

## Requirements

Outcomes, not implementations. Where an implementation seems implied, it is a
design question and is listed as one.

**R1 — The tracker is private to two parties.** Readable and writable by the
owner and by Ralph, and by nobody else. No requester account is ever created. No
tracker URL, identifier or email reaches a requester.

**R2 — Ralph runs the tracker without asking.** Create, classify, update state,
record the next action, link, reuse and close. Routine project and issue choices
are Ralph's. An ambiguous match is Ralph's. The owner may correct anything by
hand, and Ralph must not fight the correction.

**R3 — An item is legible on its own.** A person opening one item can tell who
asked, what was asked, who owns it, what state it is in, and what happens next,
without opening chat and without reading a card.

**R4 — Human messages are classified into five kinds:** question, update,
feature request, bug investigation, other. The first four are actionable and are
tracked. "Other" is not. **"Unclear" is not a sixth kind:** classification
happens once the ask is understood, and a message Ralph cannot place
confidently gets R20's question first, never a guess and never R7's no-fit
reply.

**R5 — An understood, accepted message becomes exactly one tracked item,** or
joins one that already exists (R12). A request still waiting on a clarification
answer may be recorded as pending, with its next action, so it is not lost
(R20). **A pending record is not acceptance:** it does not trigger R6, it is
never described to the requester, and it can never close as a success (R11).

**R6 — One acceptance reply, in the thread the message arrived in, only after
the ask is understood and the work is accepted and recorded.** One short
sentence saying Ralph has it and is working it. Never before the item exists,
and never while a clarification question is still open (R20). Never a tracker
link, identifier, account or email. For an **update**, this is the whole of the
conversation: the acknowledgement you described, and nothing after it (R11).

**R7 — A human message Ralph understands and will not act on gets one short
no-fit reply.** "I am not sure how to help with that", short and plain. This
covers a message classified "other" inside the eligible surface (R18). A message
that is merely unclear is **not** this case: it gets R20's question first, and
reaches R7 only if it is understood afterwards and is still not work. **No item
is created to justify the reply,** and the reply never says anything is
underway. At most one no-fit reply per thread; a thread that already got one
stays silent after that (A15).

**R8 — No new-thread fallback, ever.** If a reply cannot land in the thread it
belongs to, nothing is sent anywhere. The failure surfaces to the owner. A new
thread is a wrong answer, not a degraded one. See `gchat.py:249`.

**R9 — No false acknowledgement.** Ralph says work is underway only when it has
actually been understood, accepted and recorded. A receipt for work that does
not exist is worse than silence, because the requester stops chasing. A
clarification receipt (R20) is not an acknowledgement: it says Ralph has the
message and needs one thing, never that work has started.

**R10 — Ralph never answers itself, a bot, or the same message twice.**
Ralph's own messages — the clarification question, the acceptance reply, the
no-fit reply and the closing reply alike — bot messages, and duplicate
deliveries of one message produce nothing: no item, no reply. This is what keeps
a reply loop impossible, and it is what makes the clarification exchange in R20
safe to run automatically.

**R11 — One closing reply, and it carries the outcome.** Sent into the
originating thread, at most once per request, only when the request has actually
been worked to a result. What it says is decided by what was asked — your words
on `prd.md:66`:

| Kind | What closes it in the thread |
|---|---|
| Question | The answer. |
| Bug investigation | The finding: what was wrong, and what was done or is recommended. |
| Feature request | The pull requests it produced — named, and linked only where the requester can already reach them (R19). |
| Update | Nothing further. The acceptance reply (R6) was the acknowledgement. |

Succeeded is not the same as stopped: a request that failed, was refused, was
abandoned or was folded into other work gets **no message at all**, and the
owner decides what to say, by hand (E11, A14). An intermediate Hermes card
finishing is **not** the request finishing. A request that was never
acknowledged never gets a closing reply either, and a request still waiting on a
clarification answer has not been worked to a result: it stays pending and is
never reported complete (R20, E17).

**R12 — A later message can join existing work.** A request restated in another
thread, or a follow-up elsewhere, attaches to the item that already exists.
Ralph decides. When Ralph cannot confidently reuse an item, it records a new
trackable item and reconciles or links it later, rather than stopping to ask.
Two unconnected items for one piece of work is a failure; a linked pair is not.

**R13 — Kanban executes; the tracker is canonical.** Hermes cards do the work
and link back to the item. When card and item disagree about status, the item is
the answer a human reads.

**R14 — No new datastore.** The tracker and the existing Kanban board are what
exist. A sidecar database is not assumed and not authorized here.

**R15 — Automatic sending is bounded to four replies, and merging this
document is the grant.** What was said, and where, because the difference is
the whole of this requirement:

- The owner's first comment on this pull request, answer 2, preauthorizes "the
  bounded automatic acknowledgement/fallback reply" — R6 and R7 — and says the
  manual-send rule must not be treated as already changed.
- The same comment's answer 3 asks for a second message on real completion,
  delivered through the chat-maintainer path. It does **not** say that message
  is sent without per-message permission.
- The owner's inline comment on `prd.md:66` says what that message is for: a
  question gets answered, a bug investigation gets a report, a feature request
  gets its pull requests. That is content, not a fixed sentence.
- The owner's comment of 23:41 adds a reply that comes **before** any work: a
  receipt and one specific clarification question, in the same original thread,
  with the requester answering there. It describes the behaviour and leaves the
  mechanism to the design. It does not use the word "preauthorize" either.
- The owner's follow-up instruction, relayed through card `t_b8869aff` and not
  posted here: always preauthorize a reply, Ralph is meant to be automatic.

**What this document proposes, and a merge approves:** R20, R6, R7 and R11 may
be sent without per-message human permission, each carrying text Ralph composed
— a question about this request, or this request's own work output — bounded by
R19. Each is at most once per request, except R20's one allowed follow-up
(A17); each goes into the thread the request arrived in. Any other outbound chat
still needs the owner, per message.

**What a merge does not do.** It creates no sending permission. `SKILL.md:44`
is unedited and in force; no chat producer, writer or credential exists on this
path. The design must state exactly how the bound is enforced — a writer that
can emit only a clarification question, an acceptance reply, a no-fit reply, or
one closing reply per request; a destination fixed by the host (`SOUL.md:11`);
a count that stops a second clarification follow-up; a check that the closing
reply's content came from this request's own artifacts; a test that fails the
build if anything else can be sent — and must demonstrate it before anything
sends (unknown 5, R17).

**R16 — Chat content stays data.** A message asking for a reply is not an
instruction to reply (`SKILL.md:46-48`, `SOUL.md:23`). This feature may not
become a path by which message text causes a send it did not independently earn,
changes a destination, or decides what the closing reply says. A requester's
answer to a clarification question is data on the same terms: it may change what
Ralph understands the ask to be, and never what Ralph is allowed to send, where,
or to whom (R20).

**R17 — Nothing in this document authorizes a change.** No scope enlargement, no
REST enablement, no policy edit, no message sent. Those follow an approved
design, one at a time.

**R18 — The eligible surface is the owner, and anyone who @mentions the owner,
in Google Chat.** Your inline comment on `prd.md:64`: "It will just be the user
or anyone who mentions the user in google chat. This is fine for an initial v1."
So: the owner's own chats, and the spaces the owner put Ralph in; in a direct
message every human message is eligible, and in a space with other people only a
message that @mentions the owner or Ralph is (A12). Membership and the @mention
are the control, and **this feature adds no allowlist of its own**.

That is wider than what exists. `sender_trusted` (`SOUL.md:9`) comes from the
owner's allowlist and `route` is refused without it (`:21`), so a stranger who
@mentions the owner is eligible under R18 and refused by the current gate. The
owner's boundary is the requirement; reconciling the gate with it is design
unknown 11. **The consequence, stated:** anyone who can @mention the owner in a
space Ralph is in can create a tracked item. The owner accepted that for a first
version in the same comment.

**R19 — Outbound text is the work's own output, and nothing else.** The closing
reply may carry the answer, the finding, or the pull requests produced *for this
request*. It may not carry: anything from the work tracker — no link, no
identifier, no field (R1); anything from another request, another thread or
another requester; anything from a private document the requester has no access
to; or a reference the requester cannot already reach. When the artifact is not
reachable by the requester, Ralph names the outcome in words and omits the link.
The same bound holds for a clarification question (R20): it may refer to what
this requester wrote, and to nothing else.

**R20 — An unclear request gets a question, not a guess and not a start.** Your
inline comment on `prd.md:39` — "I don't want ralph to start without clear
understanding of what people are asking for" — and your comment of 23:41, which
separates receipt, clarification and acceptance into different replies:

- When Ralph does not understand the ask well enough to work it, it replies in
  the **same original thread** with a short receipt and **one** specific
  question. It does not say work is underway, does not begin the work, and does
  not report the work accepted (R9).
- It may record the pending request and its next action while it waits (R5).
  That record is not acceptance, and the requester is never told it exists.
- The requester answers in that same thread. Ralph reads that answer as context
  for the pending request — not as a new request, not as a second item, and
  never as an instruction (R12, R16). A follow-up question is allowed only when
  the answer narrowed the ask without settling it (A17).
- Once Ralph understands and records accepted work, the acceptance reply (R6)
  is sent, once. If the clarified message turns out to be something Ralph
  understands and will not act on, R7's no-fit reply closes it instead.
- If no answer arrives, the request stays pending. It is never reported
  complete, never gets a closing reply, and never gets a success message of any
  kind (A18, E17).
- None of this reaches the owner. What to ask, and what the answer means, are
  Ralph's to decide (R2); only a genuine failure is escalated.

This applies to every kind in R4, including a message that would otherwise be
classified "other".

---

## Acceptance examples

Public-safe and deliberately generic. Referenced elsewhere as E1-E18.

**E1 — the ordinary case.** A human sends an actionable request inside the
eligible surface, and it is clear enough to work. **Then** one item exists with
requester, ask, owner, state and next action; and exactly one short acceptance
reply appears in that same thread. No link, no identifier, no account
invitation, and no clarification question.

**E2 — the same message, twice.** One message is delivered to the pipeline
twice. **Then** one item, one reply.

**E3 — a human message that is understood and is not work.** Small talk,
thanks, an FYI. **Then** one short "I am not sure how to help with that" in that
thread, **and no item**. A second such message in the same thread gets nothing.
A message that is merely unclear is E15, not this one.

**E4 — Ralph's own message, or a bot.** **Then** nothing. No item, no reply,
and in particular no reply to Ralph's own acceptance reply.

**E5 — a follow-up from a different thread.** The same requester writes again
elsewhere about the same work. **Then** no second disconnected item: it joins
the existing item, or it becomes a new item linked to it. Either is acceptable;
two unconnected items is not.

**E6 — the thread reply fails.** The thread cannot be replied to. **Then** no
message is sent anywhere, the item still exists, and the owner is told which
reply failed and for which item. A reply in a new thread is a defect.

**E7 — an intermediate card finishes.** One Hermes card in a multi-card request
completes. **Then** the requester hears nothing.

**E8 — a question is answered.** The item closes with an answer. **Then** one
closing reply in the originating thread carrying that answer, and never a second
one. The answer is what was worked out for this request — not a tracker field
and not a quote from a private document (R19).

**E9 — a feature request ships.** The work produced pull requests. **Then** one
closing reply naming them, with links only where the requester can already open
them; otherwise the outcome in words and no link.

**E10 — an update arrives.** Someone sends context, not a request for work.
**Then** one acceptance reply, an item recorded, and **no closing reply**.

**E11 — the request ends without succeeding.** The item closes as failed,
refused, abandoned, or folded into other work. **Then** no message reaches the
requester at all. The owner is told, and the owner decides whether to say
anything.

**E12 — the owner audits.** The owner opens the tracker cold, with no chat open.
**Then** every open item reads: who asked, what, who owns it, what state, what
next, and where it came from.

**E13 — an ambiguous match.** A message might be the same work as an open item,
and might not. **Then** Ralph proceeds on its own judgement and the request is
not left waiting on the owner.

**E14 — a human message in a space, with no @mention.** **Then** nothing: it is
outside the eligible surface, so there is no item, no question and no reply of
any kind.

**E15 — the ask is unclear.** A human sends a request inside the eligible
surface that Ralph cannot work as written. **Then** one short receipt and
**one** specific question in that same thread; no sentence claiming work has
started; no acceptance reply; nothing presented as accepted work. Ralph may hold
a pending record with its next action, and the requester is not told about it.

**E16 — the requester answers.** The answer arrives in that same thread.
**Then** Ralph uses it as context, records the accepted work, and sends exactly
one acceptance reply there. The answer produces no second item, no no-fit reply,
and no second acknowledgement.

**E17 — the question is never answered.** **Then** the request stays pending:
no reminder, no second question, no closing reply, and nothing that says it
completed or succeeded. The owner can see it waiting in the tracker.

**E18 — clarification does not settle it.** After the one allowed follow-up the
ask is still not workable. **Then** Ralph stops asking the requester. If the
message is understood by then and is simply not work, E3's no-fit reply closes
it; otherwise the request stays pending, the owner is told, and the requester
hears nothing further (A17).

---

## Decisions you have already made

Recorded so the design cannot quietly re-open them. From your two comments on
this pull request and your four inline comments on it, all 2026-11-02.

| Where you said it | What you said | Where it lives now |
|---|---|---|
| Comment, answer 1 | Classify human messages as question, update, feature request, bug investigation, or other; track the first four; reply once in the same thread; "other" gets one no-fit reply and no item; ignore Ralph's own messages, bots and duplicates | R4, R5, R7, R10 |
| Comment, answer 2 | Yes, bounded preauthorization for the acknowledgement and the fallback reply. Ralph operates without per-message permission. The skill rule must not be treated as already changed; the design must show the bound | R15, first two replies |
| Comment, answer 3 | A second message on real completion, at most one, prepared and delivered through the chat-maintainer path; an intermediate card finishing is not completion | R11 |
| Comment, answer 4 | Ralph links follow-ups and runs the lifecycle. No routine questions. If reuse is not confident, record a new item and reconcile or link later | R2, R12, E13 |
| Inline, `prd.md:62` | "it's all chats for a user and rooms that they can define themselves" | R18, first paragraph |
| Inline, `prd.md:64` | "It will just be the user or anyone who mentions the user in google chat. This is fine for an initial v1." | R18. This is why there is no allowlist in this feature, and why the existing gate's conflict is named rather than carried quietly |
| Inline, `prd.md:66` | "A question gets answered. A bug investigation gets a report. A feature request gets a slew of PRs. Context gets an acknowledgement. What other closing message is required?" | R11's table, and R19's content bound. It is also why v3's fixed success sentence is gone |
| Inline, `prd.md:39` | "What about any clarification questions?I don't want ralph to start without clear understanding of what people are asking for." (quoted as written) | R20 |
| Comment, 23:41 | Receipt, clarification and acceptance are different replies. An unclear request gets a short receipt and one specific question in the same original thread, with no claim of work underway and no work started; Ralph may track the pending request and its next action; the requester answers in that thread and the answer is context; a follow-up only if needed; the acknowledgement follows once the work is understood and recorded; with no answer the request stays pending and is never reported complete; own messages, bots and duplicates are ignored so this cannot loop; the polling and send mechanism belongs in the design | R20, and R5, R6, R9, R10, R11; unknown 13 |

Two instructions reached me outside this pull request, through card
`t_b8869aff`: always preauthorize a reply because Ralph is meant to be
automatic, and stop asking routine questions. They shaped R15. Neither is on the
public record here, which is why R15 asks your merge to confirm the first rather
than citing it as already given.

---

## Choices I made for you

You said to stop asking routine questions. Four things were left over. Each is
narrow, each costs something I name here, and each is one comment away from
being changed.

### A12 — in a shared space, an @mention is what makes a message eligible

Your line-64 comment says "the user or anyone who mentions the user". In a
direct message there is nobody to mention, so every human message there is
eligible. In a space with other people, requiring the @mention is mine, not
yours: without it, R7 would answer ordinary conversation in a shared room, which
the original brief forbids. I read a mention of Ralph as equivalent to a mention
of you, since Ralph is acting for you. **Cost:** a request written in a shared
space that mentions neither is not seen.

### A14 — a request that ends without succeeding is silent toward the requester

Your line-66 comment lists outcomes that exist — an answer, a report, pull
requests. It does not say what a requester hears when there is no outcome. I
kept silence: Ralph tells you, and you decide. **Cost:** a requester whose
request quietly failed is left waiting, and only you can close that loop. Say
"tell them when it fails too" and R11 gains a failure case.

### A17 — one open question at a time, and one follow-up at most

You allow a follow-up "only if needed" and do not say how many. Unbounded, that
is a conversation with a stranger that Ralph runs by itself, so I put a ceiling
on it: one question, one follow-up, then Ralph works with what it has or tells
you. **Cost:** a genuinely tangled request can use the budget and land on your
desk instead of being sorted out in the thread. Say "keep asking until it is
clear" and the ceiling goes.

### A18 — an unanswered question is never re-sent

You said the request stays pending if no clarification arrives. You did not say
whether Ralph should ask again later. I kept silence: no reminder, no second
question, no expiry. **Cost:** a requester who simply missed the message is
never nudged, and only the tracker shows the request is waiting. Say "nudge them
if they go quiet" and R20 gains a reminder.

---

## Non-goals

- **A multi-user portal.** No requester accounts, no self-service, no queue view.
  If a requirement would make a requester log in, it is out.
- **Email.** No notifications, no ingestion, no digests to requesters.
- **A general chat auto-responder.** Four bounded replies, named in R15. Not
  status-on-demand, not a chatbot.
- **An open-ended dialogue.** The clarification exchange exists to understand
  one ask: one question, one follow-up at most, then Ralph decides or stops
  (R20, A17). It is not a chat session and not a support channel.
- **Chasing a requester.** An unanswered question is never re-sent (A18).
- **A conversation after the closing reply.** A requester who writes back starts
  a new request under R4 and R12; it is not a continuing dialogue.
- **Any tracker content reaching a requester** (R1, R19). The outcome travels;
  the record does not.
- **Telling a requester that work stopped** (A14). The owner handles it.
- **Progress updates.** The requester hears when Ralph needs one thing to
  start, when work starts, and when it ends. Nothing in between, however long it
  runs.
- **Replacing Kanban.** Cards still do the work (R13).
- **A new database or sidecar store** (R14).
- **Enlarging OAuth scopes.** The scopes at `gchat.py:54-60` already include
  send. This feature needs no more and asks for none.
- **Enabling the tracker's REST API, or any other runtime change.** A design
  decision, after approval.
- **Deciding the tool shape, the API calls, the credential split, the storage of
  the thread-to-item mapping, the classifier, how the producer runs, or how chat
  is polled for a requester's answer.** All design — you said so for the polling
  and send mechanism yourself. Any of them stated as a requirement here would be
  an invention.
- **Reviving the closed document at `Zhachory1/docs#5`.**

---

## Unknowns to settle at design time

Listed so they are not quietly decided by whoever implements first.

1. **How Ralph writes to the tracker at all,** given its REST API is off.
   Turning it on is one option, not the assumption.
2. **Where the thread-to-item mapping lives,** given R14 forbids a new store.
   The item, the card, and a staging directory are all candidates.
3. **Which sender this feature uses.** Confirm before anything else: the skill's
   OAuth token (`gchat.py:54-60`) can thread-reply; the host notifier's webhook
   (`run-notifier:18,35` — the board alert and the daily digest) sends to one
   fixed destination and cannot. If the skill's token is not available to this
   path, R6, R7, R11 and R20 all need a sender that does not exist yet.
4. **Whether `REPLY_MESSAGE_OR_FAIL` behaves as R8 needs** against a thread
   identifier captured earlier, and how stale an identifier may be.
5. **How R15's bound is enforced and proven,** so that exactly these four
   replies are sendable and nothing else is — a writer that can emit only those
   shapes, a host-fixed destination (`SOUL.md:11`), a per-request count that
   stops a second clarification follow-up (A17), a check that the closing
   reply's content came from this request's own artifacts, an eval like
   `inbox-writer`'s (`:7-8`), or some combination. The exact edit to
   `SKILL.md:44` is part of this, and so is the test that fails if the bound
   leaks. Nothing sends before this is settled and demonstrated.
6. **Who classifies, and into what vocabulary,** and who judges that an ask is
   too unclear to work (R20). Today the orchestrator classifies (`SOUL.md:15`)
   and the five kinds in R4 do not exist anywhere. Whether the chat profile
   gains decisions, or a classifier sits in front of it, is open.
7. **Which tracker states count as a worked result,** what "the request is
   finished" means mechanically, and which component notices (R11, E7, E8, E11).
   Getting this wrong is how a failed request gets told it succeeded.
8. **What chat-derived data may be stored in an item,** and for how long. R3
   wants a legible requester; R1 and public safety constrain what that means.
9. **How the owner sees a failed reply** (E6) and a genuine failure generally.
10. **Whether `inbox-writer`'s single-write eval has a chat equivalent,** and
    what it asserts.
11. **How R18's boundary is expressed against `sender_trusted`.** The gate at
    `SOUL.md:9,21` is narrower than R18 and would refuse an eligible requester.
    Widening the host's list, bypassing the gate for this path, or deriving
    eligibility from the @mention are all options; the design picks one and says
    what it costs. If the design concludes that a sender list local to this
    feature is the only safe reconciliation, that contradicts R18 and comes back
    to the owner as a change — it is not settled inside the design.
12. **How the closing reply's content is assembled and checked** against R19 —
    in particular how "the requester can already reach this reference" is
    decided for a pull request or any other artifact.
13. **How Ralph notices the requester's answer.** You left the chat polling and
    send mechanism to the design: how a thread with an open question is watched,
    how an answer is correlated back to the pending request, and what delay is
    acceptable before Ralph reads it (R20, E16).
14. **What a pending request looks like in the tracker, and how long it lives.**
    R5 allows a pending record and R3 wants it legible; nothing says whether a
    never-answered request is ever swept, or how the owner distinguishes
    "waiting on a person" from "waiting on Ralph" (A18, E17).

---

## Assumptions

Every inference this card, the owner's answers and the brief did not state. One
line each, referenced elsewhere as A1-A18 in list order.

1. **"Fallback reply" in your answer 2 means the no-fit reply of R7,** not a
   fallback to a new thread. R8 forbids the second absolutely, and I read the
   two as unrelated despite the shared word.
2. **"Accepted" means Ralph understands the ask, the item exists, and Ralph
   intends to work it** — not merely that the message was received, and not
   merely that it was understood. R6, R9 and R20 rest on that reading.
3. **The closing reply goes to the thread where the request was first
   accepted,** when the work was discussed in more than one.
4. **A follow-up in a new thread that joins existing work gets its own
   acceptance reply** in that new thread, so the requester is not ignored there;
   it does not get its own closing reply (R11 is per request, not per thread).
5. **One item per request, not per thread.**
6. **The chat skill at `/ralph/skills/ops/google-chat/` is the sender meant by
   "reuse the existing host sender".** The card also names a read-only digest
   token I could not read. Flagged by review as the most load-bearing assumption
   here; it is also design unknown 3.
7. **"Mentions the user" means an @mention in the message,** not any textual
   reference to the owner by name (R18).
8. **Your line-66 comment is an instruction about content, not a question I
   must answer back.** I read "What other closing message is required?" as "the
   outcome is the message" and wrote R11 accordingly, rather than replying with
   a fourth question.
9. **The tracker is self-hosted Redmine** on the owner's machine, per the card.
10. **Nothing is time-critical.** No latency target is stated and none is
    assumed.
11. **`/ralph` as mounted here matches the deployed tree.** It is not a git
    checkout, so I cannot prove it. The design must re-verify against a commit.
12. **In a space with other people, only an @mention is eligible,** and a
    mention of Ralph counts as a mention of the owner (R18).
13. **"Context" in your line-66 comment is the "update" kind of R4,** so an
    update's acknowledgement is R6's acceptance reply and nothing follows it
    (R11).
14. **A request that ends without succeeding is silent toward the requester**
    (R11, E11). Your comments name outcomes that exist and not the absence of
    one; this is the narrow reading.
15. **"Reply once" to a non-actionable message means once per thread, ever,**
    not once per message (R7). A thread that already got the no-fit reply stays
    silent. The wider reading would let a chatty thread collect many of them.
16. **The follow-up instruction behind R15 is yours** — always preauthorize a
    reply, Ralph is meant to be automatic. It reached me through card
    `t_b8869aff`, not through this pull request, so R15 proposes rather than
    records it.
17. **The clarification exchange is bounded: one open question at a time, and
    one follow-up at most** (R20). Your comment allows a follow-up "only if
    needed" and sets no ceiling; without one, Ralph can hold an unbounded
    conversation with a stranger on its own authority.
18. **A clarification question that is never answered is never re-sent** (R20,
    E17). You said the request stays pending; you did not say whether to ask
    again, so there is no reminder and no expiry.

---

## Observations, out of scope

One line each. Reported, not fixed.

- `gchat.py:249` hard-codes `REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD` for every
  threaded send, so any current caller can silently start a new thread.
- `/ralph` is not a git checkout in the container this ran in, so no document
  written here can cite a commit for it.
- The card places the chat profile under `/ralph/profiles/chat-maintainer/`; the
  only copy I found is the installed profile. Either the source tree is not
  mounted here, or the card's path is stale.
- `ralph-chat-maintainer` has been configured and left inert with no producer
  since at least 2026-10-30; it is either waiting on this work or it is dead
  weight.
- `/ralph/notifier/` is referenced by `run-notifier:19,26,37` but is not visible
  to this profile, so the approvals gate cannot be audited from here.
