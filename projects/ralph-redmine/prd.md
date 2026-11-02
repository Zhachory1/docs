---
id: prd-ralph-redmine
title: Ralph keeps work requests in a private tracker only the owner and Ralph can read, and answers the requester in their own chat thread — once when it starts, once when it ends
status: in review
owner: Zhachory1
card: t_8e5a6fa4
revision: v2
date: 2026-11-02
prs:
  - Zhachory1/docs#6   # v1, v2
supersedes: none
---

# Ralph keeps work requests in a private tracker only the owner and Ralph can read, and answers the requester in their own chat thread — once when it starts, once when it ends

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker" (a self-hosted Redmine), and the chat system is Google Chat, which is
> a vendor name and is named. Host paths are the public Ralph source tree and the
> local profile install, cited as found.

Status: **in review** — a requirements document, awaiting the owner's approval.
Nothing is built. No chat message has been sent, no scope changed, no REST API
enabled, no tracker touched. Merging this pull request is the approval.

Source: Kanban card `t_8e5a6fa4`. The earlier combined document at
`Zhachory1/docs#5` was closed without merge and is **not** an approved
specification. Its assumptions are not carried forward.

---

## Review header

### What changed since v1

The owner answered all four of v1's open decisions in a comment on this pull
request. v2 turns those answers into requirements and asks a smaller set of
questions that the answers raised.

- **The feature grew a second message.** v1 ended at the acknowledgement and said
  a closing message was out (old A8). The owner wants one, when the work is
  genuinely done (R11).
- **The feature grew a reply to messages it does not act on.** A human message
  that is not work now gets one short "I am not sure how to help with that"
  (R7), where v1 said silence.
- **Ralph decides; the owner is not in the loop.** Classification, issue reuse,
  linking and the tracker lifecycle are Ralph's, not escalations (R2, R12).
- **The acknowledgement is preauthorized** as a product decision (R15), which
  the design must bound. The skill rule that forbids auto-send is unchanged by
  this document and stays in force until a design changes it explicitly.
- **Three new findings** about the prepared chat profile, listed under
  [What does not exist](#what-does-not-exist): no decision in its vocabulary can
  send an automatic inbound reply, and outbound destinations are fixed by the
  host, not chosen by the agent.
- Reviewer's off-by-one on `run-notifier` corrected and re-verified; the
  indicative work table is cut to one sentence.

### Decisions needed from you: three

Full statements under [Open decisions](#open-decisions). In short:

1. **Which messages does automation see at all?** Your answer classifies
   messages; it does not say where they arrive from.
2. **Does the existing trusted-sender gate still apply,** and what does an
   untrusted human get — silence, or the "not sure how to help" reply?
3. **What may the closing message say,** and does a close that is not a success
   get one? This also decides whether an accepted question is ever answered in
   chat.

Each carries a default I will carry into the design if you merge without
answering. The default is named in each one.

### Council dissent

An independent agent review found no blocking issues and four non-blocking
notes. Resolution:

| Note | Resolution |
|---|---|
| `run-notifier` cited as `:20,26,37`; actual lines are 19, 26, 37 | Fixed. Re-read: `poller.py` at `:19`, `approvals.py` at `:26`, `digest.py` at `:37`. |
| The five-row "planned breakdown" table reads too close to design | Cut. One sentence replaces it. |
| D1 ended with "my reading, not a decision: (b)" — a nudge | Removed. Each open decision now carries a stated default and says what merging silently buys, which is a commitment, not a hint. |
| Frontmatter complete, status valid | No change. |

The reviewer flagged assumption 6 — that the sender meant by "reuse the existing
host sender" is the chat skill's token, not the daily digest's webhook — as the
one most likely to be wrong and most expensive if it is. It remains an
assumption (A6) and is now also design unknown 3: the digest path sends to a
fixed destination and cannot reply into a thread at all, so if it is the only
sender available, this feature needs a different one.

### Scope: medium, larger than v1

Still one tracker and one thread. Risk grew: there are now three kinds of
outbound message instead of one, and one of them may carry composed text.
**Breakdown:** the design document decides it, and nothing starts before this
document is merged.

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
prohibits. Your preauthorization (R15) is a product decision that this document
records and does **not** apply: the line stands as written until a design change
edits it and you approve that change.

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
reply and a closing message are each a new path, not a setting.

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
| The existing work digest uses a separate read-only chat token (`chat.messages.readonly`, `chat.spaces.readonly`) | The digest configuration is not readable from this profile. What is adjacent and confirmed: `/ralph/bootstrap/run-notifier:18-27,35-41` sends board alerts and a daily digest through `GCHAT_WEBHOOK_URL` — a fixed destination, never a thread reply — and skips them when it is unset. The token's scopes I could not check. |
| `/ralph/notifier/approvals.py` and the docs-PR gate | `/ralph/notifier/` is not present in this container. `run-notifier:19,26,37` invokes `poller.py`, `approvals.py` and `digest.py`, so they exist on the host; their contents I did not read. |
| `/ralph/profiles/author/SOUL.md` | `/ralph/profiles/` is not present. A local installed copy of the author profile states the gate this document follows: write to the docs repository as a pull request, never merge, approval is explicit. The deployed file I did not read. |
| The work tracker's state: owner-and-Ralph-only access, one private Work Requests project, one manual migration issue, REST API off, no integration running | Supplied by the owner's session. This profile must not and did not touch the work instance, its credentials, chat messages or any private document. |

---

## The outcome

**For the owner.** Every actionable request that arrives in chat exists as one
item in the work tracker, readable without opening chat. Ralph runs it from
arrival to close without asking the owner to choose a project, pick an issue, or
settle an ambiguous match. The owner hears about genuine failures only.

**For the requester.** At most two short messages, in the thread they wrote in:
one meaning received and being worked, one when it is over. A human message
Ralph cannot act on gets one honest "I am not sure how to help with that" and
nothing else.

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
the work is accepted and recorded.** Short, saying Ralph has it and is working
it. Never before the item exists. Never a tracker link, identifier, account or
email.

**R7 — An "other" human message gets one short no-fit reply.** "I am not sure
how to help with that", or equivalent. **No item is created to justify the
reply.** At most one no-fit reply per thread; a thread that already got one
stays silent after that.

**R8 — No new-thread fallback, ever.** If a reply cannot land in the thread it
belongs to, nothing is sent anywhere. The failure surfaces to the owner. A new
thread is a wrong answer, not a degraded one. See `gchat.py:249`.

**R9 — No false acknowledgement.** Ralph says work is underway only when it has
actually been accepted and recorded. A receipt for work that does not exist is
worse than silence, because the requester stops chasing.

**R10 — Ralph never answers itself, a bot, or the same message twice.**
Ralph's own messages, bot messages and duplicate deliveries of one message
produce nothing — no item, no reply. This is what keeps a reply loop impossible.

**R11 — One closing message, when the request is actually finished.** Sent into
the originating thread, at most once per request. An intermediate Hermes card
finishing is **not** the request finishing. A request that was never
acknowledged never gets a closing message either.

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

**R15 — The automatic reply is preauthorized at the product level, and bounded.**
The owner grants that these three message kinds may be sent without per-message
human permission. The grant does not extend to arbitrary outbound chat: anything
outside the acceptance reply (R6), the no-fit reply (R7) and the closing message
(R11) still needs the owner. The design must show how that bound is enforced,
and must state the edit it makes to `SKILL.md:44` rather than assuming the rule
already changed.

**R16 — Chat content stays data.** A message asking for a reply is not an
instruction to reply (`SKILL.md:46-48`, `SOUL.md:23`). This feature may not
become a path by which message text causes a send it did not independently earn,
or changes a destination.

**R17 — Nothing in this document authorizes a change.** No scope enlargement, no
REST enablement, no policy edit, no message sent. Those follow an approved
design, one at a time.

---

## Acceptance examples

Public-safe and deliberately generic.

**A1 — the ordinary case.** An eligible human sends an actionable request.
**Then** one item exists with requester, ask, owner, state and next action; and
exactly one short reply appears in that same thread. No link, no identifier, no
account invitation.

**A2 — the same message, twice.** One message is delivered to the pipeline
twice. **Then** one item, one reply.

**A3 — a human message that is not work.** Small talk, thanks, an FYI. **Then**
one short "I am not sure how to help with that" in that thread, **and no item**.
A second such message in the same thread gets nothing.

**A4 — Ralph's own message, or a bot.** **Then** nothing. No item, no reply,
and in particular no reply to Ralph's own acceptance reply.

**A5 — a follow-up from a different thread.** The same requester writes again
elsewhere about the same work. **Then** no second disconnected item: it joins
the existing item, or it becomes a new item linked to it. Either is acceptable;
two unconnected items is not.

**A6 — the thread reply fails.** The thread cannot be replied to. **Then** no
message is sent anywhere, the item still exists, and the owner is told which
reply failed and for which item. A reply in a new thread is a defect.

**A7 — an intermediate card finishes.** One Hermes card in a multi-card request
completes. **Then** the requester hears nothing.

**A8 — the request finishes.** The item reaches a closed state. **Then** exactly
one closing message in the originating thread, and never a second one.

**A9 — the owner audits.** The owner opens the tracker cold, with no chat open.
**Then** every open item reads: who asked, what, who owns it, what state, what
next, and where it came from.

**A10 — an ambiguous match.** A message might be the same work as an open item,
and might not. **Then** Ralph proceeds on its own judgement and the request is
not left waiting on the owner.

---

## Decisions you have already made

Recorded so the design cannot quietly re-open them. From your comment on this
pull request, 2026-11-02.

| v1 asked | You answered | Now |
|---|---|---|
| D1: which messages are eligible | Classify human messages as question, update, feature request, bug investigation, or other; track the first four; reply once in the same thread; "other" gets one no-fit reply and no item; ignore Ralph's own messages, bots and duplicates | R4, R7, R10; open decision O1 covers the part your answer did not reach |
| D2: do you preauthorize the acknowledgement | Yes, bounded. Ralph operates without per-message permission. The skill rule must not be treated as already changed; the design must show the bound | R15 |
| D3: what may the reply say, and is there a second message | Yes to a second message, on real completion, at most one; an intermediate card finishing is not completion | R11; open decision O3 covers its wording |
| D4: who links a follow-up | Ralph. No routine questions to the owner. If reuse is not confident, record a new item and reconcile or link later | R2, R12, A10 |

---

## Open decisions

Three. Each changes the shape of the design, and each carries a default I will
carry if you merge this document without answering it.

### O1 — Which messages does the automation see at all?

Your answer classifies messages by kind. It does not say which messages reach
the classifier. The two constraints in the brief only fit together if that set
is narrower than everything Ralph can read: the brief forbids "automatic
response to every message", while R7 requires a reply even to messages Ralph
will not act on. If the set were every message in every space, Ralph would
answer every human remark in every space it can see.

- **(a) Direct messages only.** Smallest. An @mention in a shared space is
  ignored, which will feel broken the first time a request arrives that way.
- **(b) Direct messages, plus messages that @mention Ralph or the owner.**
  Covers how requests actually arrive in a shared space, and keeps bystander
  conversation invisible to the automation. **This is the default I will carry.**
- **(c) Every human message in every space Ralph can read.** Contradicts the
  brief's "not every message" and produces a no-fit reply to ordinary
  conversation. I do not recommend it.

### O2 — Does the trusted-sender gate still apply, and what does an untrusted human get?

`SOUL.md:9` sets `sender_trusted` from your allowlist, never from the message,
and `:21` makes `route` unavailable for untrusted senders whatever the message
says. Your answer did not mention the allowlist. The gate is still in force in
the source, so the design must either keep it or be told to drop it.

- **(a) Keep the gate. An untrusted sender gets silence** — no item, no reply of
  either kind. The automation is invisible to anyone not on your list.
  **This is the default I will carry.**
- **(b) Keep the gate for work, drop it for the reply.** An untrusted human gets
  the no-fit reply. More courteous, and it tells anyone who messages Ralph that
  something automatic is listening.
- **(c) Drop the gate for this path.** Anyone who can message Ralph can create
  tracked work. Not recommended; it makes your private tracker writable by
  strangers through a side door.

### O3 — What may the closing message say, and does every close get one?

R11 says one message when the request finishes. Two things are undecided: what
happens when an item closes **without** success, and whether any Ralph message
may carry composed text rather than a fixed sentence. The second matters most,
because an accepted **question** has an answer, and a fixed sentence cannot
deliver it.

- **(a) Fixed sentence, successful close only.** Any other close is silent — the
  requester who was told "I'm on it" hears nothing again. An accepted question
  is never answered in chat.
- **(b) One message on any close, composed to fit the outcome,** including the
  answer to a question or the reason it is not being done. The widest grant
  here, still bounded by R1 (no link, identifier, account or email), R11 (one
  message) and R16 (no content lifted from the work instance or a private
  document). **This is the default I will carry**, because (a) leaves questions
  unanswered and leaves abandoned requests hanging.
- **(c) One message on any close, fixed-form per outcome** — done, not doing it,
  folded into other work. No free text anywhere in the feature. Safest, and a
  question still goes unanswered.

Pick (c) if you want no composed outbound text at all. Say nothing and the
design will assume (b).

---

## Non-goals

- **A multi-user portal.** No requester accounts, no self-service, no queue view.
  If a requirement would make a requester log in, it is out.
- **Email.** No notifications, no ingestion, no digests to requesters.
- **A general chat auto-responder.** Three message kinds, bounded by R15. Not
  conversation, not status-on-demand, not a chatbot.
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
5. **How R15's bound is enforced** so that the three permitted messages are
   sendable and nothing else is — a writer that can only emit those shapes, a
   host-fixed destination (`SOUL.md:11`), an eval like
   `inbox-writer`'s (`:7-8`), or some combination. The exact edit to
   `SKILL.md:44` is part of this.
6. **Who classifies, and into what vocabulary.** Today the orchestrator
   classifies (`SOUL.md:15`) and the five kinds in R4 do not exist anywhere.
   Whether the chat profile gains decisions, or a classifier sits in front of
   it, is open.
7. **What "the request is finished" means mechanically,** and which component
   notices (R11, A7, A8).
8. **What chat-derived data may be stored in an item,** and for how long. R3
   wants a legible requester; R1 and public safety constrain what that means.
9. **How the owner sees a failed reply** (A6) and a genuine failure generally.
10. **Whether `inbox-writer`'s single-write eval has a chat equivalent,** and
    what it asserts.

---

## Assumptions

Every inference this card, the owner's answers and the brief did not state. One
line each.

1. **"Fallback reply" in your answer 2 means the no-fit reply of R7,** not a
   fallback to a new thread. R8 forbids the second absolutely, and I read the
   two as unrelated despite the shared word.
2. **"Accepted" means the item exists and Ralph intends to work it** — not
   merely that the message was understood. R6 and R9 rest on that reading.
3. **The closing message goes to the thread where the request was first
   accepted,** when the work was discussed in more than one.
4. **A follow-up in a new thread that joins existing work gets its own
   acceptance reply** in that new thread, so the requester is not ignored there;
   it does not get its own closing message (R11 is per request, not per thread).
5. **One item per request, not per thread.**
6. **The chat skill at `/ralph/skills/ops/google-chat/` is the sender meant by
   "reuse the existing host sender".** The card also names a read-only digest
   token I could not read. Flagged by review as the most load-bearing assumption
   here; it is also design unknown 3.
7. **The requester allowlist, if O2 keeps it, is the owner's existing
   trusted-sender list** (`SOUL.md:9`), not a new list built for this feature.
8. **The acceptance reply and the no-fit reply are fixed form,** not composed
   per message. Only the closing message may carry composed text, and only if
   you pick O3(b).
9. **The tracker is self-hosted Redmine** on the owner's machine, per the card.
10. **Nothing is time-critical.** No latency target is stated and none is
    assumed.
11. **`/ralph` as mounted here matches the deployed tree.** It is not a git
    checkout, so I cannot prove it. The design must re-verify against a commit.

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
