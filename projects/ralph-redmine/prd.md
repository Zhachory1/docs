---
id: prd-ralph-redmine
title: Ralph keeps work requests in a private tracker only the owner and Ralph can read, and answers the requester in their own chat thread — once when it starts, once when it has actually succeeded
status: in review
owner: Zhachory1
card: t_8e5a6fa4
revision: v3
date: 2026-11-02
prs:
  - Zhachory1/docs#6   # v1, v2, v3
supersedes: none
---

# Ralph keeps work requests in a private tracker only the owner and Ralph can read, and answers the requester in their own chat thread — once when it starts, once when it has actually succeeded

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

**May Ralph send three fixed messages without asking you each time?** R15 names
them: an acceptance reply (R6), a no-fit reply (R7), and one success sentence
(R11). Each is fixed wording, each at most once, each into the thread the
request arrived in. Nothing else may be sent automatically.

Your comment on this pull request preauthorized the first two. The third does
not come from this pull request: it comes from your follow-up instruction —
always preauthorize a reply, Ralph is meant to be automatic — relayed through
card `t_b8869aff`. So this document **proposes** that grant instead of
recording it as already given. Merging is how you grant it. A comment saying
"acknowledgement only" is how you refuse it, and R11 then becomes a message you
send by hand.

**No sending permission is live today, and merging does not create one.**
`SKILL.md:44` still forbids automatic send and this document does not edit it.
There is no chat producer, no writer and no credential on this path. The grant
is a product requirement; the design must define the authorization boundary and
test it before anything can send (R15, R17, unknown 5).

Three smaller choices I made rather than asking you a third time. Each is an
assumption, and each is one comment away from changing:

| I chose | Instead of | Say this to change it |
|---|---|---|
| A12 — surface: your direct messages, plus the spaces you put Ralph in; in a space with other people, only messages that @mention Ralph | Acting on every message in those spaces | "act on everything in the rooms" |
| A13 — the trusted-sender gate stays for work; an untrusted human in that surface still gets the no-fit reply, which starts nothing | Silence for anyone not on your allowlist | "untrusted senders get silence" |
| A14 — one fixed success sentence, on genuine success only, with no composed text anywhere in the feature | A message written per outcome, able to carry a question's answer or a reason for stopping | "let the second message be written per outcome" |

A14 has a cost worth naming: an accepted **question** is tracked and worked,
and its answer never reaches the requester in chat. You read it in the tracker
and answer by hand. That is the price of having no generated outbound text.

### Council dissent

An independent agent review of v2 found one blocking issue and two non-blocking
notes.

| Finding | Resolution |
|---|---|
| **Blocking.** R15 claimed you preauthorized three message kinds. Your comment names only the acknowledgement and the no-fit reply; answer 3 never says the second message is sent without permission. A silent merge would have invented a grant. | Accepted. R15 now separates what you said, where you said it, and what this document proposes. The success message is proposed, and the header says merging is the grant. |
| **Blocking, second half.** O3's default — one message on any close, composed to fit the outcome — was the widest grant in the document and was set to bind by silence. | Accepted, and narrowed rather than asked again. One fixed sentence, on success only (R11, R19). No composed outbound text exists anywhere in the feature. |
| R7's "at most one no-fit reply per thread, ever" is an inference from "reply once", written as a requirement rather than an assumption. | Fixed. It is A15, and R7 points at it. |
| The Unverified table cited `run-notifier:18-27,35-41`; lines 25-27 are the unrelated `GH_TOKEN` branch. | Fixed: `:18-23,35-41`. |

The reviewer's load-bearing assumption is unchanged: A6, that "the existing host
sender" means the chat skill's OAuth token and not the digest webhook. The
webhook sends to one fixed destination and cannot reply into a thread at all,
so if it is the only sender available, all three messages need a sender that
does not exist yet. It is design unknown 3, and the first thing to confirm.

### Scope: medium

One tracker, one thread, three fixed sentences. Smaller than v2: composed text
is gone, which removes the only path by which Ralph could have written
something unpredictable to a requester. **Breakdown:** the design document
decides it, and nothing starts before this document is merged.

### What changed since v2

- **R15 no longer claims a grant you did not give.** It separates your public
  comment, your follow-up instruction, and what this document proposes.
- **The second message is one fixed sentence, on real success only** (R11).
  A request that failed, was refused, was abandoned, or was folded into other
  work gets nothing (E11). No composed text anywhere (R19).
- **The three open decisions are gone,** replaced by three narrow choices with
  their costs named (A12-A14), editable by one comment here.
- **The eligible surface is a requirement now** (R18), from your inline comment
  that it is "all chats for a user and rooms that they can define themselves".
- Two reviewer notes folded in: A15, and the `run-notifier` citation range.
- Acceptance examples are relabelled **E1-E13**, so they stop colliding with
  the assumption ids A1-A16 they shared in v1 and v2.

---

## Who this is for

**The owner, and only the owner.** Work requests arrive in chat and get lost,
because a chat thread is not a list.

**The requester is not a user of this system.** They never see the tracker,
never get a link to it, never get an account, never get an email. The only thing
that reaches a requester is short text in the thread they already wrote in. That
is the main constraint below: anything that would make a requester go somewhere
or sign in is out of scope.

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
empty body (`:19`). So an automatic acceptance reply, a "not sure how to help"
reply and a success message are each a new path, not a setting.

**No threaded-reply guarantee.** `gchat.py:249` sets
`messageReplyOption = "REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD"`. That is the exact
behaviour R8 forbids: when the thread cannot be replied to, the helper starts a
new one instead of failing. The API's `REPLY_MESSAGE_OR_FAIL` is not used
anywhere I found.

**No classification into the kinds you named.** The profile routes to the
orchestrator, which "classifies, picks the workflow, and writes the cards"
(`SOUL.md:15`). Nothing classifies a message as question, update, feature
request or bug investigation today.

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
arrival to close without asking the owner to choose a project, pick an issue, or
settle an ambiguous match. The owner hears about genuine failures only.

**For the requester.** At most two short fixed sentences, in the thread they
wrote in: one meaning received and being worked, one when the work has actually
succeeded. A human message Ralph will not act on gets one honest "I am not sure
how to help with that". Nothing else, ever — and if the work fails or is
abandoned, nothing at all.

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
tracked. "Other" is not.

**R5 — An actionable message becomes exactly one tracked item,** or joins one
that already exists (R12).

**R6 — One acceptance reply, in the thread the message arrived in, only after
the work is accepted and recorded.** One short fixed sentence, saying Ralph has
it and is working it (R19). Never before the item exists. Never a tracker link,
identifier, account or email.

**R7 — A human message Ralph will not act on gets one short no-fit reply.**
"I am not sure how to help with that", fixed wording (R19). This covers both a
message classified "other" and a message from a sender the allowlist does not
trust (R18). **No item is created to justify the reply,** and the reply never
says anything is underway. At most one no-fit reply per thread; a thread that
already got one stays silent after that (A15).

**R8 — No new-thread fallback, ever.** If a reply cannot land in the thread it
belongs to, nothing is sent anywhere. The failure surfaces to the owner. A new
thread is a wrong answer, not a degraded one. See `gchat.py:249`.

**R9 — No false acknowledgement.** Ralph says work is underway only when it has
actually been accepted and recorded. A receipt for work that does not exist is
worse than silence, because the requester stops chasing.

**R10 — Ralph never answers itself, a bot, or the same message twice.**
Ralph's own messages, bot messages and duplicate deliveries of one message
produce nothing — no item, no reply. This is what keeps a reply loop impossible.

**R11 — One success message, one fixed sentence, only when the request has
actually succeeded.** Sent into the originating thread, at most once per
request. Succeeded is not the same as stopped: a request that failed, was
refused, was abandoned or was folded into other work gets **no message at all**,
and the owner decides what to say, by hand (E11, A14). An intermediate Hermes
card finishing is **not** the request finishing. A request that was never
acknowledged never gets one either.

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

**R15 — Automatic sending is bounded to three fixed messages, and merging this
document is the grant.** What was said, and where, because the difference is
the whole of this requirement:

- The owner's comment on this pull request, answer 2, preauthorizes "the
  bounded automatic acknowledgement/fallback reply" — R6 and R7 — and says the
  manual-send rule must not be treated as already changed.
- The same comment's answer 3 asks for a second message on real completion,
  delivered through the chat-maintainer path. It does **not** say that message
  is sent without per-message permission, and it does not authorize generated
  text.
- The owner's follow-up instruction, relayed through card `t_b8869aff` and not
  posted here: always preauthorize a reply, Ralph is meant to be automatic.

**What this document proposes, and a merge approves:** R6, R7 and R11 may be
sent without per-message human permission. Each is fixed wording (R19), each is
at most once, each goes into the thread the request arrived in. Any other
outbound chat still needs the owner, per message.

**What a merge does not do.** It creates no sending permission. `SKILL.md:44`
is unedited and in force; no chat producer, writer or credential exists on this
path. The design must state exactly how the bound is enforced — a writer able
to emit only these three shapes, a destination fixed by the host
(`SOUL.md:11`), a test that fails the build if anything else can be sent, or
some combination — and must demonstrate it before anything sends (unknown 5,
R17).

**R16 — Chat content stays data.** A message asking for a reply is not an
instruction to reply (`SKILL.md:46-48`, `SOUL.md:23`). This feature may not
become a path by which message text causes a send it did not independently earn,
or changes a destination.

**R17 — Nothing in this document authorizes a change.** No scope enlargement, no
REST enablement, no policy edit, no message sent. Those follow an approved
design, one at a time.

**R18 — The eligible surface is the owner's own chats and the spaces the owner
put Ralph in.** The automation sees nothing else. In a direct message every
human message is eligible; in a space with other people, only a message that
@mentions Ralph is (A12). Membership is the owner's control, and this feature
adds no second list. The existing trusted-sender gate (`SOUL.md:9,21`) still
decides whose message becomes work; it does not decide who gets the no-fit
reply (A13).

**R19 — No composed outbound text.** All three messages are fixed sentences
carrying no detail from the request, the item, the tracker, or any private
document. Nothing Ralph writes reaches a requester. A consequence to accept
with this: an accepted question is tracked and worked, and its answer does not
reach the requester in chat (A14).

---

## Acceptance examples

Public-safe and deliberately generic. Referenced elsewhere as E1-E13.

**E1 — the ordinary case.** A trusted human sends an actionable request inside
the eligible surface. **Then** one item exists with requester, ask, owner, state
and next action; and exactly one short fixed reply appears in that same thread.
No link, no identifier, no account invitation.

**E2 — the same message, twice.** One message is delivered to the pipeline
twice. **Then** one item, one reply.

**E3 — a human message that is not work.** Small talk, thanks, an FYI. **Then**
one short "I am not sure how to help with that" in that thread, **and no item**.
A second such message in the same thread gets nothing.

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

**E8 — the request succeeds.** The item reaches a successful closed state.
**Then** exactly one fixed success sentence in the originating thread, and never
a second one.

**E9 — the owner audits.** The owner opens the tracker cold, with no chat open.
**Then** every open item reads: who asked, what, who owns it, what state, what
next, and where it came from.

**E10 — an ambiguous match.** A message might be the same work as an open item,
and might not. **Then** Ralph proceeds on its own judgement and the request is
not left waiting on the owner.

**E11 — the request ends without succeeding.** The item closes as failed,
refused, abandoned, or folded into other work. **Then** no message reaches the
requester at all. The owner is told, and the owner decides whether to say
anything.

**E12 — an untrusted human in the eligible surface.** Someone not on the
allowlist writes in a space Ralph is in and @mentions it. **Then** no item and
no routing, and one no-fit reply that claims nothing is underway.

**E13 — a human message in a space, with no @mention.** **Then** nothing: it is
outside the eligible surface, so there is no item and no reply of either kind.

---

## Decisions you have already made

Recorded so the design cannot quietly re-open them. From your comment on this
pull request, 2026-11-02.

| v1 asked | You answered | Now |
|---|---|---|
| D1: which messages are eligible | Classify human messages as question, update, feature request, bug investigation, or other; track the first four; reply once in the same thread; "other" gets one no-fit reply and no item; ignore Ralph's own messages, bots and duplicates | R4, R7, R10. The surface your answer did not reach is settled by your inline comment on this pull request — "all chats for a user and rooms that they can define themselves" — and is now R18 |
| D2: do you preauthorize the acknowledgement | Yes, bounded, for the acknowledgement and the fallback reply. Ralph operates without per-message permission. The skill rule must not be treated as already changed; the design must show the bound | R15, first two message kinds |
| D3: what may the reply say, and is there a second message | Yes to a second message, on real completion, at most one, prepared and delivered through the chat-maintainer path; an intermediate card finishing is not completion | R11, R19. Whether it is sent without per-message permission is the one thing merging decides (R15) |
| D4: who links a follow-up | Ralph. No routine questions to the owner. If reuse is not confident, record a new item and reconcile or link later | R2, R12, E10 |

Two instructions reached me outside this pull request, through card
`t_b8869aff`: always preauthorize a reply because Ralph is meant to be
automatic, and stop asking routine questions. They shaped R15 and the three
choices below. Neither is on the public record here, which is why R15 asks your
merge to confirm the first rather than citing it as already given.

---

## Choices I made for you

You said to stop asking routine questions. Three things were left over. Each is
narrow, each costs something I name here, and each is one comment away from
being changed.

### A12 — surface: your own chats, plus the spaces you put Ralph in

Your inline comment on this pull request settles it: "all chats for a user and
rooms that they can define themselves". So membership is the control, and this
feature adds no list of its own (R18). One narrowing is mine, not yours: in a
space with other people, Ralph acts only on a message that @mentions it.
Without that, R7 would answer ordinary conversation, which the brief forbids.
**Cost:** a request written in a shared space with no @mention is not seen. In
a direct message no @mention is needed.

### A13 — the trusted-sender gate stays for work, not for courtesy

`SOUL.md:9,21` gates on your allowlist and your answers did not mention it, so
it stays: an untrusted sender creates no item and routes nothing. Inside the
eligible surface they still get the no-fit reply, which starts nothing and
claims nothing (R7). **Cost:** it tells a stranger in one of your spaces that
something automatic is listening. Say "silence" if you would rather they got
nothing at all.

### A14 — one fixed success sentence, on success only

No composed outbound text anywhere (R19), and nothing at all when a request
ends without succeeding (R11, E11). This is the narrowest form of the second
message you asked for, and the narrowest is the only form that is safe to bind
by a merge, because a merge is silent about wording. **Cost, stated plainly:**
an accepted question is tracked and worked, and its answer never reaches the
requester in chat. You answer it by hand or not at all.

---

## Non-goals

- **A multi-user portal.** No requester accounts, no self-service, no queue view.
  If a requirement would make a requester log in, it is out.
- **Email.** No notifications, no ingestion, no digests to requesters.
- **A general chat auto-responder.** Three fixed sentences, bounded by R15. Not
  conversation, not status-on-demand, not a chatbot.
- **Composed or generated text to a requester** (R19). Including the answer to
  an accepted question.
- **Telling a requester that work stopped.** A request that does not succeed is
  silent towards the requester (R11, E11); the owner handles it.
- **Progress updates.** The requester hears twice: start and end. Nothing in
  between, however long it runs.
- **Replacing Kanban.** Cards still do the work (R13).
- **A new database or sidecar store** (R14).
- **Enlarging OAuth scopes.** The scopes at `gchat.py:54-60` already include
  send. This feature needs no more and asks for none.
- **Enabling the tracker's REST API, or any other runtime change.** A design
  decision, after approval.
- **Deciding the tool shape, the API calls, the credential split, the storage of
  the thread-to-item mapping, the classifier, or how the producer runs.** All
  design. Any of them stated as a requirement here would be an invention.
- **Reviving the closed document at `Zhachory1/docs#5`.**

---

## Unknowns to settle at design time

Listed so they are not quietly decided by whoever implements first.

1. **How Ralph writes to the tracker at all,** given its REST API is off.
   Turning it on is one option, not the assumption.
2. **Where the thread-to-item mapping lives,** given R14 forbids a new store.
   The item, the card, and a staging directory are all candidates.
3. **Which sender this feature uses.** Confirm before anything else: the skill's
   OAuth token (`gchat.py:54-60`) can thread-reply; the digest's webhook
   (`run-notifier:18,35`) sends to a fixed destination and cannot. If the
   skill's token is not available to this path, R6, R7 and R11 all need a sender
   that does not exist yet.
4. **Whether `REPLY_MESSAGE_OR_FAIL` behaves as R8 needs** against a thread
   identifier captured earlier, and how stale an identifier may be.
5. **How R15's bound is enforced and proven,** so that the three permitted
   fixed messages are sendable and nothing else is — a writer that can only
   emit those shapes, a host-fixed destination (`SOUL.md:11`), an eval like
   `inbox-writer`'s (`:7-8`), or some combination. The exact edit to
   `SKILL.md:44` is part of this, and so is the test that fails if the bound
   leaks. Nothing sends before this is settled and demonstrated.
6. **Who classifies, and into what vocabulary.** Today the orchestrator
   classifies (`SOUL.md:15`) and the five kinds in R4 do not exist anywhere.
   Whether the chat profile gains decisions, or a classifier sits in front of
   it, is open.
7. **Which tracker states count as success,** what "the request is finished"
   means mechanically, and which component notices (R11, E7, E8, E11). Getting
   this wrong is how a failed request gets told it succeeded.
8. **What chat-derived data may be stored in an item,** and for how long. R3
   wants a legible requester; R1 and public safety constrain what that means.
9. **How the owner sees a failed reply** (E6) and a genuine failure generally.
10. **Whether `inbox-writer`'s single-write eval has a chat equivalent,** and
    what it asserts.

---

## Assumptions

Every inference this card, the owner's answers and the brief did not state. One
line each, referenced elsewhere as A1-A16 in list order.

1. **"Fallback reply" in your answer 2 means the no-fit reply of R7,** not a
   fallback to a new thread. R8 forbids the second absolutely, and I read the
   two as unrelated despite the shared word.
2. **"Accepted" means the item exists and Ralph intends to work it** — not
   merely that the message was understood. R6 and R9 rest on that reading.
3. **The success message goes to the thread where the request was first
   accepted,** when the work was discussed in more than one.
4. **A follow-up in a new thread that joins existing work gets its own
   acceptance reply** in that new thread, so the requester is not ignored there;
   it does not get its own success message (R11 is per request, not per thread).
5. **One item per request, not per thread.**
6. **The chat skill at `/ralph/skills/ops/google-chat/` is the sender meant by
   "reuse the existing host sender".** The card also names a read-only digest
   token I could not read. Flagged by review as the most load-bearing assumption
   here; it is also design unknown 3.
7. **The requester allowlist is the owner's existing trusted-sender list**
   (`SOUL.md:9`), not a new list built for this feature (A13).
8. **All three messages are fixed form** (R19). Nothing in this feature is
   composed per message, per requester or per outcome.
9. **The tracker is self-hosted Redmine** on the owner's machine, per the card.
10. **Nothing is time-critical.** No latency target is stated and none is
    assumed.
11. **`/ralph` as mounted here matches the deployed tree.** It is not a git
    checkout, so I cannot prove it. The design must re-verify against a commit.
12. **In a space with other people, only an @mention is eligible** (R18). Your
    inline comment names the spaces, not which messages inside them; without
    this narrowing R7 would answer ordinary conversation.
13. **The trusted-sender gate stays for work and not for the no-fit reply**
    (R7, R18). Your answers did not mention the allowlist, so I kept it where
    it decides work and did not extend it to a reply that starts nothing.
14. **The success message is one fixed sentence, sent only on success** (R11,
    R19). Answer 3 asked for a second message and did not say what it may
    contain; this is the narrowest reading that still delivers it.
15. **"Reply once" to a non-actionable message means once per thread, ever,**
    not once per message (R7). A thread that already got the no-fit reply stays
    silent. The wider reading would let a chatty thread collect many of them.
16. **The follow-up instruction behind R15 is yours** — always preauthorize a
    reply, Ralph is meant to be automatic. It reached me through card
    `t_b8869aff`, not through this pull request, so R15 proposes rather than
    records it.

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
