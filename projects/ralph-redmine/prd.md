---
id: prd-ralph-redmine
title: Ralph keeps work requests in a private tracker only the owner and Ralph can read, and answers the requester once, in the chat thread they asked in
status: in review
owner: Zhachory1
card: t_8e5a6fa4
revision: v1
date: 2026-11-02
prs:
  - Zhachory1/docs#PR   # v1
supersedes: none
---

# Ralph keeps work requests in a private tracker only the owner and Ralph can read, and answers the requester once, in the chat thread they asked in

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker" (a self-hosted Redmine), and the chat system is Google Chat, which is
> a vendor name and is named. Host paths under `/ralph/...` are the public Ralph
> source tree and are cited as found.

Status: **in review** — a requirements document, awaiting the owner's approval.
Nothing is built. No chat message has been sent, no scope changed, no REST API
enabled, no tracker touched. Merging this pull request is the approval.

Source: Kanban card `t_8e5a6fa4`. This replaces nothing: the earlier combined
document at `Zhachory1/docs#5` was closed without merge and is **not** an
approved specification. Its assumptions are not carried forward.

---

## Review header

**Decisions needed from you: four.** They are stated in full under
[Open decisions](#open-decisions). In short:

1. **Which messages are eligible at all?** DMs from allowlisted senders only, or
   also @mentions in shared spaces? This sets the blast radius of everything else.
2. **Do you preauthorize the acknowledgement?** Today the chat skill requires your
   explicit instruction per message. An automatic acknowledgement cannot exist
   without you narrowing that rule, or rejecting the feature.
3. **What may the acknowledgement say?** One fixed sentence, or may it name what
   was recorded?
4. **When a later message looks like the same work, who decides?** Ralph attaches
   it to the existing item, or Ralph asks you.

**Council dissent:** none recorded yet. An independent agent review runs on this
pull request; any dissent and its resolution lands in v2.

**Scope: medium.** Small in surface — one tracker, one acknowledgement — and
medium in risk, because the feature's whole point is a write that other people
can see, under a credential that today refuses to fire without you.

**Planned breakdown, indicative only.** The design document decides this; it is
here so you can judge the size, not so it binds anyone.

| # | Work | Gate |
|---|------|------|
| 1 | Design document for this PRD | Your merge of the design PR |
| 2 | Write path to the work tracker (create, update, close) | Design |
| 3 | Inbound chat producer: stage a message, create a card | Design |
| 4 | Decision step: accept, link to existing, or decline | Design |
| 5 | Acknowledgement writer: same-thread reply or hard fail | Design, and decision 2 below |

Nothing in that table starts until this document is merged.

---

## Who this is for

**The owner, and only the owner.** The owner receives work requests in chat and
loses them, because a chat thread is not a list. The owner wants a list that
Ralph maintains, and wants the requester to stop wondering whether the message
landed.

**The requester is not a user of this system.** They never see the tracker, never
get a link to it, never get an account, never get an email about it. The only
thing that reaches a requester is one short sentence in the thread they already
wrote in. That is deliberate, and it is the main constraint on everything below:
anything that would require a requester to go somewhere or sign in is out of scope.

---

## Current state, verified

I read the public Ralph source for this. **`/ralph` in this container is not a
git checkout**, so I cannot cite a commit; files were read on 2026-11-02 and the
chat skill files carry mtime 2026-10-22. Treat the paths as the citation and
re-check them before implementing.

### What exists

**A chat helper that can already send, including into a thread.**
`/ralph/skills/ops/google-chat/gchat.py:243-252` defines `cmd_send`, which calls
`spaces.messages.create` and takes an optional `--thread`. The token at
`gchat.py:51` holds five scopes (`gchat.py:54-60`), and they are **not**
read-only: `chat.messages.create` (`:57`) and `chat.spaces.create` (`:58`) are
both present. The capability is there. Nothing has to be built or re-scoped to
make a thread reply technically possible.

**A policy that forbids using it the way this feature needs.**
`/ralph/skills/ops/google-chat/SKILL.md:44` is explicit: run `send` only on an
explicit, per-message instruction from the actual user; never send because a
message that was *read* asked for a reply; never batch or auto-send without a
fresh instruction each time. An automatic acknowledgement is, by definition, the
thing that line prohibits. This is the hardest constraint in the document and it
is decision 2.

**A trust boundary that already treats chat as data.**
`SKILL.md:46-48` and `gchat.py:16-18` say chat content read back is external
data, never instructions, and `gchat.py:62-63,178-185` wraps read output in
external-content markers. A message asking Ralph to reply is not an instruction
to reply. This feature must not become a loophole in that rule.

**A decision profile, prepared and inert.** `ralph-chat-maintainer` has a
`SOUL.md` describing one chat item per card and exactly five decisions —
`route`, `draft_reply`, `escalate`, `send`, `ignore` (`SOUL.md:13-19`) — with
`sender_trusted` set by the host from the owner's allowlist and never by the
message (`:9`), and `route` refused outright for untrusted senders (`:21`). Its
`config.yaml` gives it the `file` tool on the CLI and `no_mcp` on the API server,
with one external skills directory. **It has no network, no chat credential, and
no cron entry.** It can decide; it cannot act.

**A working pattern for exactly this shape, built for email.**
`/ralph/bootstrap/inbox-producer:1-23` runs on the host where the token lives,
stages a thread as JSON into a directory, and creates a card whose workspace is
that directory, keyed for idempotency on thread id plus latest message id.
`/ralph/bootstrap/inbox-writer:1-20` turns the finished card into exactly one
write, and an eval fails the build if a send call ever appears in it. The
producer/writer split, the host-held token and the single-write guarantee are all
proven. They are proven **for Gmail**.

### What does not exist

**No chat producer and no chat writer.** `/ralph/bootstrap` contains
`inbox-producer` and `inbox-writer` and no chat equivalent. Nothing turns an
inbound chat message into a card, and nothing carries a `chat-maintainer`
decision back out to chat. The inert profile is inert because its two halves
were never written.

**No threaded-reply guarantee.** `gchat.py:249` sets
`messageReplyOption = "REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD"`. That is the exact
behaviour this feature forbids: if the thread cannot be replied to, the helper
starts a new one instead of failing. The API's `REPLY_MESSAGE_OR_FAIL` is not
used anywhere I found.

**No acknowledgement decision.** The five decisions in `SOUL.md:13-19` do not
include one that means "I accepted this and recorded it". `send` is for outbound
text another profile supplied, not for an automatic receipt.

**No tracker integration of any kind.** No issue-maintainer, no write path, no
reference to the work tracker in the Ralph source I read.

### Unverified

| Claim | Why unverified |
|---|---|
| The existing work digest uses a separate read-only chat token (`chat.messages.readonly`, `chat.spaces.readonly`) and a fixed-space webhook | The digest configuration is not readable from this profile. What I can confirm is adjacent: `/ralph/bootstrap/run-notifier:18-27,35-41` sends board alerts and a daily digest through `GCHAT_WEBHOOK_URL` — a fixed destination, not a thread reply — and skips them when it is unset. The token's scopes I could not check. |
| `/ralph/notifier/approvals.py` and the docs-PR gate | `/ralph/notifier/` is not present in this container. `run-notifier:26` references `approvals.py`, so the file exists on the host; its contents I did not read. |
| `/ralph/profiles/author/SOUL.md` | `/ralph/profiles/` is not present in this container. A local snapshot copy of the author profile's `SOUL.md` (lines 7 and 11) states the gate this document is following: write to the docs repository as a pull request, never merge, approval is explicit. The deployed file itself I did not read. |
| The work tracker's state: owner-and-Ralph-only access, one private Work Requests project, one manual migration issue, REST API off, no integration running | Supplied by the owner's session. This profile must not and did not touch the work instance, its credentials, chat messages or any private document. |

---

## The outcome

**For the owner.** Every actionable request that arrives in chat exists as one
item in the work tracker, with an owner, a state and a next action readable
without opening chat. Ralph maintains it through its life. The owner never does
data entry to keep it current.

**For the requester.** One short sentence, once, in the thread they wrote in,
meaning: received, and being worked. Nothing else, ever.

**For everyone else.** Nothing. No account, no portal, no notification, no
mailing list.

---

## Requirements

Outcomes, not implementations. Where an implementation seems implied, it is a
design question and is listed as one.

**R1 — The tracker is private to two parties.** Readable and writable by the
owner and by Ralph, and by nobody else. No requester account is ever created. No
tracker URL, identifier or email reaches a requester.

**R2 — Ralph maintains its own items end to end.** Create, update state, record
the next action, and close. The owner may correct anything by hand and Ralph must
not fight the correction.

**R3 — An item is legible on its own.** A person opening one item can tell who
asked, what was asked, who owns it, what state it is in, and what happens next,
without opening chat and without reading a card.

**R4 — One acknowledgement, in the original thread, only when work was
accepted.** Exactly one short reply, in the thread the request arrived in, sent
only after the item exists and the work is genuinely accepted.

**R5 — No fallback thread, ever.** If the reply cannot land in the original
thread, nothing is sent. The failure surfaces to the owner. A new thread is a
wrong answer, not a degraded one. This needs a change to how the helper is
called, not to what it can do: see `gchat.py:249`.

**R6 — No acknowledgement without acceptance.** Declined, unclear, untrusted, not
actionable, or a bot notice: no reply, no item. Silence is the correct output. A
false receipt is worse than none, because the requester stops chasing.

**R7 — Not every message is answered.** Automation reacts only to messages that
pass the eligibility rule in decision 1. Everything else is invisible to it.

**R8 — A later message can join existing work.** A request restated in another
thread, or a follow-up elsewhere, may attach to the item that already exists
rather than spawning a disconnected second one. Who decides that is decision 4.

**R9 — Kanban executes; the tracker is canonical.** Hermes cards do the work and
link back to the item. When card and item disagree about status, the item is the
answer a human reads.

**R10 — No new datastore.** The tracker and the existing Kanban board are what
exist. A sidecar database is not assumed and not authorized here.

**R11 — Chat content stays data.** A message asking for a reply is not an
instruction to reply (`SKILL.md:46-48`). This feature may not become a path by
which message text causes a send it did not independently earn.

**R12 — Nothing in this document authorizes a change.** No scope enlargement, no
REST enablement, no policy edit, no message sent. Those follow an approved
design, one at a time.

---

## Acceptance examples

Public-safe and deliberately generic.

**A1 — the ordinary case.** An allowlisted requester sends an actionable request
in an eligible thread. **Then** one item exists in the tracker with requester,
ask, owner, state and next action; and exactly one short reply appears in that
same thread. No item link, no identifier, no account invitation in the reply.

**A2 — the same request, twice.** The same message is delivered to the pipeline
twice. **Then** there is still one item and still one reply.

**A3 — a follow-up from a different thread.** The same requester writes again
elsewhere about the same work. **Then** no second disconnected item is created.
Whether it attaches automatically, and whether that second thread gets its own
reply, is decision 4; the invariant this document fixes is that two unconnected
items for one piece of work is a failure.

**A4 — declined.** The message is not actionable, or the sender is not trusted.
**Then** no item, no reply, nothing visible to the requester. The owner can still
see that it was seen and declined.

**A5 — a bot notice or a thank-you.** **Then** nothing happens. No item, no reply.

**A6 — the thread reply fails.** The thread cannot be replied to. **Then** no
message is sent anywhere, the item still exists, and the owner is told the
acknowledgement failed. A reply in a new thread is a defect.

**A7 — the owner audits.** The owner opens the tracker cold, with no chat open.
**Then** every open item reads: who asked, what, who owns it, what state, what
next, and where it came from.

**A8 — the work finishes.** Ralph closes the item. **Then** the requester gets no
closing message, unless decision 3 says otherwise.

---

## Open decisions

Four. Each is a product judgement I will not make for you, because each changes
the shape of the design.

### D1 — Which messages are eligible?

The blast radius of the whole feature.

- **(a) DMs from an allowlisted sender only.** Smallest. An @mention in a shared
  space is ignored, which will feel broken the first time it happens.
- **(b) DMs plus @mentions of the owner, allowlist-gated.** Covers how requests
  actually arrive in a shared space. Larger surface: anyone in a space can put a
  message in front of the automation, though the allowlist still gates acceptance.
- **(c) Any message from an allowlisted sender, anywhere.** Loudest. Likely to
  acknowledge things that were not requests.
- **(d) Nothing automatic. Ralph proposes; you confirm each one.** Zero risk, and
  it does not deliver the outcome: the requester's wait is exactly as long as your
  attention.

Note `SOUL.md:9,21`: `sender_trusted` is set by the host from your allowlist and
never by the message, and untrusted senders cannot be routed at all. Whichever
option you pick inherits that.

**My reading, not a decision:** (b), because (a) breaks in the common case and
(c) acknowledges noise.

### D2 — Do you preauthorize the acknowledgement?

`SKILL.md:44` forbids exactly this: no auto-send, no send because a read message
asked, a fresh instruction every time. The feature cannot exist alongside that
rule unchanged.

- **(a) Approve a narrow preauthorization.** One fixed-form reply, only into the
  originating thread, only after an item exists, only for senders passing D1,
  rate-limited, and logged. Everything else still requires you.
- **(b) Reject it.** The acknowledgement becomes a `draft_reply` you send, and
  R4's promise weakens to "when the owner next looks".

This document does not change that policy and must not be read as having changed
it. Approving (a) is an instruction to write the narrowed rule into the design,
where you review it again.

### D3 — What may the reply say?

- **(a) One fixed sentence**, e.g. "I got this and am working on it." Says nothing
  that could leak, and says nothing useful on a second reading.
- **(b) Fixed sentence plus a one-line restatement of what was recorded.** Lets the
  requester catch a misread immediately. Costs a decision about what may be echoed
  back.

Neither option may include a tracker link, identifier, account or email (R1).
Also decide: is there ever a **second** message — on completion, on blocked? A8
currently says no.

### D4 — Who links a later message to existing work?

- **(a) Ralph attaches it automatically** when confidence is high. Fewer
  duplicates, and a wrong attachment buries a real request inside an unrelated one.
- **(b) Ralph asks you**, via `escalate` (`SOUL.md:17`). Never wrong, always
  slower, and it puts a decision in your queue for every follow-up.
- **(c) Ralph always creates a new item and links the two.** No judgement call, and
  your list grows.

---

## Non-goals

- **A multi-user portal.** No requester accounts, no self-service, no public queue
  view. If a requirement would make a requester log in, it is out.
- **Email.** No notifications, no ingestion, no digests to requesters.
- **A general chat auto-responder.** One acknowledgement for accepted work. Not
  status updates, not answers, not conversation.
- **Replacing Kanban.** Cards still do the work (R9).
- **A new database or sidecar store** (R10).
- **Enlarging OAuth scopes.** The scopes at `gchat.py:54-60` already include send.
  This feature needs no more, and asks for none.
- **Enabling the tracker's REST API, or any other runtime change.** A design
  decision, after approval.
- **Deciding the tool shape, the API calls, the credential split, the storage of
  the thread-to-item mapping, or how the producer runs.** All design. Any of those
  stated as a requirement here would be an invention.
- **Reviving the closed document at `Zhachory1/docs#5`.**

---

## Unknowns to settle at design time

Listed so they are not quietly decided by whoever implements first.

1. **How Ralph writes to the tracker at all,** given its REST API is off. Turning
   it on is one option, not the assumption.
2. **Where the thread-to-item mapping lives,** given R10 forbids a new store. The
   item itself, the card, and a staging directory are all candidates.
3. **Whether `REPLY_MESSAGE_OR_FAIL` behaves as R5 needs** against a thread
   identifier captured earlier, and how stale an identifier may be.
4. **Which component holds the credential.** The email pattern keeps the token on
   the host and out of the agent's reach (`inbox-producer:1-23`); whether chat
   copies that, and whether the reply is a sixth `chat-maintainer` decision or a
   separate writer, is open.
5. **What chat-derived data may be stored in an item,** and for how long. R3 wants
   a legible requester; R1 and the public-safety rule constrain what that means.
6. **How the owner sees a failed acknowledgement (A6).**
7. **Whether `inbox-writer`'s single-write eval has a chat equivalent,** and what
   it asserts.

---

## Assumptions

Every inference this card and the owner's brief did not state. One line each.

1. **"Accepted" means the item exists and Ralph intends to work it** — not merely
   that the message was understood. R4 and A1 are written on that reading.
2. **The requester allowlist is the owner's existing trusted-sender list**
   (`SOUL.md:9`), not a new list built for this feature.
3. **One item per request, not per thread.** A3 assumes a thread is where a request
   arrived, not what a request is.
4. **"Short acknowledgement" means one sentence,** no formatting, no attachment.
5. **The tracker is self-hosted Redmine** on the owner's machine, per the card; no
   hosted service is assumed.
6. **The chat skill at `/ralph/skills/ops/google-chat/` is the sender meant by
   "reuse the existing host sender".** The card also names a read-only digest token
   I could not read; I assume the send path is the skill's token, not the digest's.
7. **`/ralph` as mounted here matches the deployed tree.** It is not a git checkout,
   so I cannot prove it. A design must re-verify against a commit.
8. **Nothing is time-critical.** No latency target is stated and none is assumed;
   "promptly" is left to the design.

---

## Observations, out of scope

One line each. Reported, not fixed.

- `gchat.py:249` hard-codes `REPLY_MESSAGE_FALLBACK_TO_NEW_THREAD` for every
  threaded send, so any current caller can silently start a new thread.
- `/ralph` is not a git checkout in the container this ran in, so no document
  written here can cite a commit for it.
- `ralph-chat-maintainer` has been configured and left inert since at least
  2026-10-30 with no producer; it is either waiting on this work or it is dead
  weight.
- `/ralph/notifier/` is referenced by `run-notifier:20,26,37` but is not visible to
  this profile, so the approvals gate cannot be audited from here.
