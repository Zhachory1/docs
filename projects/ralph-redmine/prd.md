---
id: prd-ralph-redmine
title: Ralph keeps work requests in a private tracker only the owner and Ralph can read, and replies in the requester's own chat thread — asking first when the ask is unclear, once when work starts, and once with the outcome itself
status: in review
owner: Zhachory1
card: t_8e5a6fa4
revision: v6
date: 2026-11-06
prs:
  - Zhachory1/docs#6   # v1, v2, v3, v4, v5
  - Zhachory1/docs#8   # v6, this reconciliation
supersedes: none
---

# Ralph keeps work requests in a private tracker only the owner and Ralph can read, and replies in the requester's own chat thread — asking first when the ask is unclear, once when work starts, and once with the outcome itself

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker" (a self-hosted Redmine), and the chat system is Google Chat, which is
> a vendor name and is named. Host paths are the public Ralph source tree and the
> local profile install, cited as found.

Status: **in review** — v5 was merged at `Zhachory1/docs#6` on 2026-11-03, and
that merge is the approval, and the one grant R15 asks for. **v6 asks for
nothing new.** It is a reconciliation: four statements below described behaviour
that the owner's own later decisions overruled, and this revision makes them say
what was decided. Nothing is built. No chat message has been sent, no scope
changed, no REST API enabled, no tracker touched.

Source: Kanban cards `t_8e5a6fa4` (v1-v5) and `t_73bf3c94` (v6). The earlier
combined document at `Zhachory1/docs#5` was closed without merge and is **not**
an approved specification. Its assumptions are not carried forward.

---

## Review header

### Decisions needed from you: one, and it is about this revision's standing

**v6 contains no new product decision.** Every change is one of your own
decisions written down: **D2** and **D6** on design `dd-ralph-redmine`, both
2026-11-06. The four edits are listed under "What changed since v5".

**The one thing only you can settle: is this reconciliation still what you
want?** `dd-ralph-redmine`, the design that recorded D2 and D6, was **closed
without merging** on 2026-11-06 at your direction, and is being split into two
successor designs. So these amendments rest on your own quoted words and on
shipped code at a named commit — not on a merged design. Your closing note says
the decisions "carry over", and v6 takes that at face value. If you would rather
leave the PRD at v5 until the successor designs land, close this pull request;
nothing is lost, because every edit is recorded here and on the design's branch.

**A fifth divergence exists and v6 deliberately does not touch it.** R15 and the
"Current state, verified" section still say no chat sender exists on this path
and that `SKILL.md:44` is unedited. `dd-ralph-redmine` found both already
shipped. Card `t_73bf3c94` scoped v6 to four items and said not to reopen R15,
so this is reported under observations and not fixed. One sentence from you puts
it in a v7.

#### What the v5 merge already granted

**May Ralph reply automatically, and may those replies carry text it wrote?**
Four messages, named in R15:

| Reply | When | What it carries |
|---|---|---|
| Clarification (R20) | The ask is not understood well enough to work it | A short receipt and **one** specific question. Never "I am working on it". |
| Acceptance (R6) | The work is understood, accepted and recorded | Ralph's own words: what it understood, and that it is working on it (your D2). |
| No-fit (R7) | A message Ralph understands and will not act on | A plain refusal: the reason, and where to go instead (your D2). |
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

Four things I chose rather than asking you again. **You have since answered two
of them** — the first two rows — and the other two still stand, each one comment
away from changing:

| I chose | Instead of | Say this to change it |
|---|---|---|
| A12 — ~~in a space with other people, only a message that @mentions you or Ralph is eligible~~ **Retired by your D6:** eligibility is the approved space list and nothing else | Acting on every message in those spaces | Settled — you chose "act on everything in the rooms". See R18 |
| A14 — ~~a request that ends **without** succeeding is silent toward the requester~~ **Amended by your D2:** silence when the work never finished, one honest reply when it finished badly | An automatic "this did not work out" | Settled — you chose "tell them when it fails too" for the finished-badly case. See A14 |
| A17 — one open clarification question at a time, and one follow-up at most; then Ralph acts on what it has or tells you | An unbounded back-and-forth with the requester | "keep asking until it is clear" |
| A18 — a clarification question that is never answered is never re-sent; the request stays pending and silent | A reminder after some time | "nudge them if they go quiet" |

### Council dissent

An independent agent review of v4 found **no blocking issues** and two
non-blocking notes.

| Finding | Resolution |
|---|---|
| Non-blocking. Unknown 3 called `run-notifier:18,35` "the digest's webhook". Line 18 gates the board alert and line 35 the daily digest; both use the same fixed destination, so the substance held and only the label overclaimed. | Fixed: unknown 3 now says "the host notifier's webhook". |
| Non-blocking. R18's "this feature adds no allowlist of its own" nudges unknown 11 away from one option it lists — a sender list local to this feature. | Named rather than reworked in v5. **Overtaken by your D6 (2026-11-06):** an approved list of spaces is now the surface itself, so unknown 11 is closed rather than constrained, and R18 no longer forbids a list. The reviewer was right that the clause was doing work; it stopped being true rather than being argued away. |

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

### Scope: the feature stays medium; this revision is small

**The feature, unchanged.** One tracker, one thread, and at most three messages
to a requester: a question if the ask is unclear, an acknowledgement when work
starts, the outcome at the close. The design decides the implementation
breakdown, and v6 changes nothing about it.

**This revision.** One file, prose only, no new requirement: `prd.md` from v5 to
v6, reconciling four statements to decisions you already made. **Breakdown: one
pull request, and it is this one.** There is no second part and nothing follows
it — the implementation pull requests belong to the successor designs, not here.

### What changed since v5

Four statements, and nothing else. Each one described behaviour your own later
decision overruled; each now says what you chose. No requirement gained or lost
scope, and no ceiling moved.

| # | What v5 said | What you decided | Where |
|---|---|---|---|
| 1 | **R6** — the acceptance reply is "one short sentence saying Ralph has it and is working it" | **D2**, 2026-11-06: Ralph writes the acknowledgement in its own words. The bound moved from the sentence to the sender's refusal checks | R6, and the replies table above |
| 2 | **R7** — the no-fit reply is the fixed sentence "I am not sure how to help with that" | **D2**, 2026-11-06: a plain refusal carrying the reason and where to go instead | R7, E3 |
| 3 | **A14** — a request that ends without succeeding is silent toward the requester, in every case | **D2**, 2026-11-06: silence when the work **never finished**; one honest reply when it **finished badly** | A14, R11, E11, non-goals |
| 4 | **R18** — the surface is a direct message or an @mention of the owner, and "this feature adds no allowlist of its own" | **D6**, 2026-11-06: *"Keep it only to approved chat rooms please"* / *"@ mentions and direct messages can come later"* — every human message in an approved space, no mention needed, no direct messages in v1 | R18, A12, E14, unknown 11, assumptions 7 and 12 |

**Why these moved at all.** Items 1-3 are places where v5 specified a *sentence*
and the code that shipped writes *text*; you chose the shipped behaviour. Item 4
is the one that most needed fixing, because v5's R18 and your D6 contradict each
other in **both** directions — v5 was narrower inside a room and wider outside
one — so a reader could not tell which was current.

**What else this revision touched, and why.** The four edits are restated in
other places, and leaving those would make the document contradict itself. So
also updated: the two review-header tables, "For the requester", R11's failure
sentence, E3, E11, E14, the non-goals for failure and for deferred surfaces,
design unknown 11, assumptions 7, 12 and 14, and the decisions table — plus four
new assumptions (19-22) for the inferences v6 itself makes. Every one is a
consequence of the four, not a new requirement. **Deliberately not touched:**
R15, R19, A17, the per-thread ceilings, and the fifth divergence named above.

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

**A sender set narrower than v5's R18 — since settled the other way.**
`sender_trusted` comes from the owner's allowlist (`SOUL.md:9`) and `route` is
refused without it (`:21`). v5's R18 — the owner, or anyone who @mentions the
owner — was wider than an allowlist and was not expressible here, which v5
recorded as a conflict and as unknown 11. **Your D6 of 2026-11-06 settled it the
other way:** an approved list of spaces is the surface, so the gate and the
requirement now agree and unknown 11 is closed. The finding stands as a record
of what was true on 2026-11-02; it is no longer a conflict.

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
understood, accepted and recorded, one short acknowledgement in Ralph's own words
(R6). When it has actually been worked to a result, one closing reply that is
the outcome itself — the answer, the finding, or the pull requests (R11). An
update gets the acknowledgement and nothing after it. A human message Ralph
understands and will not act on gets one plain refusal with the reason and where
to go instead (R7). If the work finished and finished badly, one honest reply
saying so. If it never finished at all, or the question is never answered,
nothing more (A14, A18).

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
the ask is understood and the work is accepted and recorded.** Ralph writes it
in its own words: what it understood the ask to be, and that it is working on it
— not a fixed sentence. Your **D2** of 2026-11-06 on design `dd-ralph-redmine`,
"let's go with the recommendation", chose the behaviour that shipped at
`Zhachory1/ralph@95bd4d9`, where `profiles/chat-maintainer/SOUL.md:17` says the
acknowledgement is "your short reply to the requester, in your own words, saying
what you understood and that you are working on it". Short, and never a promise
of a date or of a result the work has not produced. What bounds its content is
no longer the sentence but the sender's refusal checks, which `dd-ralph-redmine`
defines. **Never before the item exists, and never while a clarification
question is still open (R20). Never a tracker link, identifier, account or
email.** For an **update**, this is the whole of the conversation: the
acknowledgement you described, and nothing after it (R11).

**R7 — A human message Ralph understands and will not act on gets one short
no-fit reply.** A plain refusal in Ralph's own words, carrying the reason and
where the requester could go instead — not a fixed sentence. Your **D2** of
2026-11-06 on design `dd-ralph-redmine` chose the shipped behaviour, where
`SOUL.md:18` at `Zhachory1/ralph@95bd4d9` calls for "a plain 'I can't help with
this' with the reason and where they could go instead". It stays short, and it
stays plain. This covers a message classified "other" inside the eligible
surface (R18). A message that is merely unclear is **not** this case: it gets
R20's question first, and reaches R7 only if it is understood afterwards and is
still not work. **No item is created to justify the reply,** and the reply never
says anything is underway. At most one no-fit reply per thread; a thread that
already got one stays silent after that (A15).

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

Succeeded is not the same as stopped, and since your **D2** of 2026-11-06 the
two failing cases differ. A request that **never finished** — blocked,
abandoned, refused before work, or folded into other work — gets **no message at
all**, and the owner decides what to say, by hand. A request that **did finish
and finished badly** gets one honest reply saying so plainly and what is
missing, never a claimed result (E11, A14). An intermediate Hermes card
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

**R18 — The eligible surface is every human message in a chat space the owner
has approved.** Your **D6** of 2026-11-06 on design `dd-ralph-redmine`: *"Keep
it only to approved chat rooms please"*, and *"@ mentions and direct messages can
come later"*. So for v1 a message is eligible because of **where** it was sent,
not because of who it names: every message from a human sender, in one of the
spaces on the owner's approved list, inside the recent-message window the design
sets (five minutes, your D4). Nothing is asked about who the human is, and
nothing is asked about what the message says. This is consistent with your
earlier inline comment on `prd.md:62` — "it's all chats for a user and rooms that
they can define themselves".

**Both halves of what this replaced, stated plainly.** v5 said the surface was a
direct message or an @mention of the owner, and that "this feature adds no
allowlist of its own". v1 as you decided it is the opposite on both counts: an
approved list of spaces **is** the surface, and no mention is needed inside one.
So R18 both **widens** — an unmentioned message in an approved space is now
eligible — and **narrows**: a direct message is not a surface in v1 at all,
because a direct message is not one of the approved spaces. The @mention test
and direct messages are **deferred to a later design, not refused**; "later" is
your word, and your inline comment on `prd.md:64` is superseded rather than
contradicted.

**The consequence, stated:** every human message in an approved space becomes a
tracked operation and one unit of Ralph's attention, including a conversation
between two other people that has nothing to do with Ralph. The non-actionable
ones end at one no-fit reply or at nothing (R7, R10), and the per-thread
ceilings still cap what any one thread can be told, but nothing bounds the
volume. **So an approved space should be one the owner is willing to have Ralph
read in full.** One sentence puts the @mention test back.

This no longer conflicts with the host gate. `sender_trusted` (`SOUL.md:9,11`)
is set by the host "from the approved intake policy", and the approved space
list **is** that policy — so the gate and the requirement now say the same
thing, and design unknown 11 is closed. The list itself stays the owner's: it
lives in an instance file on the owner's machine, not in the code.

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
thanks, an FYI. **Then** one short, plain refusal in that thread — the reason,
and where to go instead — **and no item**. A second such message in the same
thread gets nothing. A message that is merely unclear is E15, not this one.

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

**E11 — the request ends without succeeding.** Two cases since your D2. If the
work **never finished** — blocked, abandoned, refused before work, or folded
into other work — **then** no message reaches the requester at all; the owner is
told and decides whether to say anything. If the work **finished badly** — it
ran to an end, and the end was a failure or is incomplete — **then** exactly one
honest reply in that thread, saying so plainly and saying what is missing, and
never a claimed result.

**E12 — the owner audits.** The owner opens the tracker cold, with no chat open.
**Then** every open item reads: who asked, what, who owns it, what state, what
next, and where it came from.

**E13 — an ambiguous match.** A message might be the same work as an open item,
and might not. **Then** Ralph proceeds on its own judgement and the request is
not left waiting on the owner.

**E14 — a human message in a space that is not on the approved list.** **Then**
nothing: it is outside the eligible surface, so there is no item, no question
and no reply of any kind. A direct message is the same case in v1, because a
direct message is not an approved space (R18, your D6). The converse is now also
an example: an unmentioned human message **inside** an approved space is
eligible, and is E1 or E3 by its content.

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
`Zhachory1/docs#6` and your four inline comments on it, all 2026-11-02, and from
your decisions on design `dd-ralph-redmine`, 2026-11-06.

| Where you said it | What you said | Where it lives now |
|---|---|---|
| Comment, answer 1 | Classify human messages as question, update, feature request, bug investigation, or other; track the first four; reply once in the same thread; "other" gets one no-fit reply and no item; ignore Ralph's own messages, bots and duplicates | R4, R5, R7, R10 |
| Comment, answer 2 | Yes, bounded preauthorization for the acknowledgement and the fallback reply. Ralph operates without per-message permission. The skill rule must not be treated as already changed; the design must show the bound | R15, first two replies |
| Comment, answer 3 | A second message on real completion, at most one, prepared and delivered through the chat-maintainer path; an intermediate card finishing is not completion | R11 |
| Comment, answer 4 | Ralph links follow-ups and runs the lifecycle. No routine questions. If reuse is not confident, record a new item and reconcile or link later | R2, R12, E13 |
| Inline, `prd.md:62` | "it's all chats for a user and rooms that they can define themselves" | R18, first paragraph |
| Inline, `prd.md:64` | "It will just be the user or anyone who mentions the user in google chat. This is fine for an initial v1." | **Superseded by your D6 below.** It is why v5's R18 had no allowlist of its own. The @mention surface it describes is deferred, not refused |
| `dd-ralph-redmine`, D2, 2026-11-06 | "Let's go with the recommendation" — where the shipped sender and this document disagreed, the shipped behaviour wins and the bound moves from the sentence to the sender | R6, R7, A14, R11, E3, E11 |
| `dd-ralph-redmine`, D6, 2026-11-06 | "Keep it only to approved chat rooms please" / "@ mentions and direct messages can come later" | R18, A12, E14, unknown 11, assumptions 7 and 12 |
| `dd-ralph-redmine`, D7, 2026-11-06 | "Approved. Update the PRD" | This revision exists because of it; card `t_73bf3c94` |
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

### A12 — retired by your D6: the approved space is the whole test

v5 assumed that in a shared space an @mention is what makes a message eligible,
and that in a direct message every human message is. **Your D6 of 2026-11-06
replaced both halves:** eligibility is the approved space list and nothing else,
and direct messages are not a v1 surface. Nothing reads a mention now, so the
cost v5 named here — "a request written in a shared space that mentions neither
is not seen" — is gone, and the cost that replaces it is stated in R18: Ralph
reads every human message in an approved space, including conversations that are
not for it. The concern behind the old assumption was that R7 would answer
ordinary conversation in a shared room; what answers it now is that an approved
space is a space the owner chose, and that most such messages end in no reply at
all. A12 is kept rather than deleted, because R18 and the assumptions list
referred to it and a reader of v5 needs to find out what became of it.

### A14 — amended by your D2: silence when it never finished, one honest reply when it finished badly

v5 kept silence toward the requester in every failing case, because your line-66
comment listed outcomes that exist and not the absence of one. **Your D2 of
2026-11-06 took the other reading,** so A14 splits in two rather than being
deleted:

- **The work never finished** — blocked, abandoned, refused before work, or
  folded into other work. The requester hears nothing. This is less a rule Ralph
  enforces than a consequence of how the work is sequenced: the step that would
  send a closing reply never runs. You are told, and you decide what to say.
- **The work finished badly** — it ran to an end, and the end was a failure or
  is incomplete. The requester gets **one** honest reply saying so plainly and
  saying what is missing, as `SOUL.md:23` at `Zhachory1/ralph@95bd4d9` already
  requires of the shipped path. It sits inside R11's one-reply-per-request
  ceiling and inside R19's content bound, and it never claims a result the work
  did not produce.

**Cost of the second case, which is new:** a requester can be told that Ralph
tried and failed, in text Ralph composed, without you reading it first. That is
the same exposure R15 already grants for the other replies, under the same
bound. **Cost of the first case, unchanged:** a requester whose request died
quietly is left waiting, and only you can close that loop. Say "tell them when
nothing finished either" and the first case gains a reply too.

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
- **Telling a requester that work never finished** (A14). A request that is
  blocked, abandoned or folded into other work is silent toward the requester
  and the owner handles it. A request that *did* finish and finished badly is
  **not** this case: it gets one honest reply (your D2).
- **Direct messages, and the @mention test** (R18, your D6). Neither is a v1
  surface. Both are deferred to a later design, not refused: "@ mentions and
  direct messages can come later" is your sentence. v1's surface is the approved
  space list.
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
11. **Closed by your D6 (2026-11-06), not by a design.** While R18's surface was
    an @mention, the gate at `SOUL.md:9,21` was narrower than the requirement and
    would have refused an eligible requester. That is no longer so: the approved
    space list is the surface, `SOUL.md:11` already has the host set
    `sender_trusted` from "the approved intake policy", and that list is the
    policy. No sender list local to this feature is needed and none is added.
    Kept in place, numbered, so the unknowns below keep their numbers.
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
line each, referenced elsewhere as A1-A22 in list order. Numbers are never
reused: an assumption your decision retired keeps its number and says so.

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
7. **Retired by your D6 (2026-11-06).** v5 read "mentions the user" as an
   @mention in the message rather than any textual reference to the owner by
   name. Nothing reads a mention now, so the reading no longer bears on any
   requirement (R18, A12).
8. **Your line-66 comment is an instruction about content, not a question I
   must answer back.** I read "What other closing message is required?" as "the
   outcome is the message" and wrote R11 accordingly, rather than replying with
   a fourth question.
9. **The tracker is self-hosted Redmine** on the owner's machine, per the card.
10. **Nothing is time-critical.** No latency target is stated and none is
    assumed.
11. **`/ralph` as mounted here matches the deployed tree.** It is not a git
    checkout, so I cannot prove it. The design must re-verify against a commit.
12. **Retired by your D6 (2026-11-06).** v5 assumed that in a shared space only
    an @mention is eligible, and that a mention of Ralph counts as a mention of
    the owner. Eligibility is now the approved space list alone (R18, A12).
13. **"Context" in your line-66 comment is the "update" kind of R4,** so an
    update's acknowledgement is R6's acceptance reply and nothing follows it
    (R11).
14. **Amended by your D2 (2026-11-06).** v5 assumed silence toward the requester
    in every failing case, on the narrow reading of comments that named outcomes
    which exist and not the absence of one. Silence now covers only work that
    never finished; work that finished badly gets one honest reply (R11, E11,
    A14).
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

New in v6. Four inferences this reconciliation itself makes:

19. **Your decisions on `dd-ralph-redmine` still bind this document, even though
    that design was closed without merging.** Your closing note says the
    decisions "carry over", and your D7 — "Approved. Update the PRD" — is itself
    one of them. If you meant the PRD to wait for the two successor designs
    instead, close the v6 pull request and `prd.md` stays at v5.
20. **"No other requirement is touched" permits editing the places where these
    four rules are *restated*.** R11, E3, E11, E14, the two review-header
    tables, the non-goals, unknown 11 and three assumptions all repeat a rule
    that changed. Leaving them would make the document contradict itself, which
    is the defect v6 exists to remove. No requirement gained or lost scope.
21. **The five-minute recent-message window in R18 is the design's to set, not a
    requirement.** Your D4 fixed the number; R18 names it so the surface is
    legible, and the design owns it. A different window would not change R18.
22. **This document's status stays `in review`.** The v5 merge approved v5. v6 is
    not approved until you merge it, so the field is correct as it stands rather
    than being advanced to `approved`.

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
- **New in v6, and the largest one.** R15 and "Current state, verified" both say
  no chat sender exists on this path and that `SKILL.md:44` is unedited. Neither
  is true at `Zhachory1/ralph@95bd4d9`: `bootstrap/chat-reply` ships and
  `SKILL.md:44` already names it as the approved automatic sender. Card
  `t_73bf3c94` scoped v6 to four items and said not to reopen R15, so this is
  reported and not fixed. It is the fifth divergence, and it is a bigger one
  than any of the four.
- **New in v6.** `dd-ralph-redmine` was closed without merging on 2026-11-06 and
  is being split into two successor designs. Until those exist, this document is
  the only merged text in the project, and the four amendments above are the only
  record in it of what was decided on that design.
