---
id: dd-ralph-redmine
title: Build the work tracker onto the chat path Ralph already shipped — one producer to feed it, one tracker script the sender calls but cannot import, and host-side ceilings on every reply
status: in review
owner: Zhachory1
card: t_b5db2cf0
revision: v6
date: 2026-11-06
implements: prd-ralph-redmine
prs:
  - Zhachory1/docs#7   # v1, v2, v3, v4, v5, v6
supersedes: none
---

# Build the work tracker onto the chat path Ralph already shipped — one producer to feed it, one tracker script the sender calls but cannot import, and host-side ceilings on every reply

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker" (a self-hosted Redmine), and Google Chat is a vendor name and is
> named. The PRD's banner applies unchanged.

Status: **in review** — a design, awaiting your approval. Nothing is built, no
chat message was sent or read, no REST API was enabled, no credential and no
work instance was touched. Merging this pull request is the approval.

Implements `prd-ralph-redmine` v5, merged at `Zhachory1/docs#6`. **You answered v3's
four decisions on 2026-11-06, and v5's remaining two the same day; §0.1 and §0.3
record what each one moved.** One decision is left, D5, and it gates an
installation step rather than the design. **Your D6 answer changes who Ralph
answers** — the @mention test is gone and the approved room list is the whole
gate (§3.3) — and **your D7 answer sends the PRD reconciliation to its own card**,
`t_73bf3c94`.

**Citations.** Every claim about Ralph is from a fresh clone of
`Zhachory1/ralph` at commit `95bd4d960e9178a0ea00c8791afa75b851138e1c`
(2026-11-05, `main`). v2 was written against `b41c99d` (2026-11-02). Thirty-eight
commits landed between them and several of v2's central claims are no longer
true. v4 re-read the same commit to answer your container question (§3.5) and to
design the five-minute read (§3.4). v5 re-read `chat-reply`, `chat-route` and
`install-chat-reply-launchd` in full at that commit to check the split proposal;
three of its premises did not hold, which is why §2's Option C is not the shape
it described. v6 re-read `chat-reply:41-53`, `chat-route:55-64` and
`gchat.py:222-240` to rewrite eligibility; all three hold as cited, and v6 adds
no new claim about the code. No earlier claim moved. I read the code; I did
not run it, send a message, or call any API.
Claims about Redmine are general product knowledge and are marked; your
instance was not inspected and must not be.

---

## 0. What changed in Ralph, and what it does to v2

v2 proposed a new host-side `chat-writer` that sends four fixed-shape replies. Ralph
has since shipped most of that path under another name. v3 builds on it instead
of beside it (Reuse first, `profiles/PRINCIPLES.md`).

| v2 said (at `b41c99d`) | Now (at `95bd4d9`) | Effect on the design |
|---|---|---|
| No chat sender exists; write `bootstrap/chat-writer` | `bootstrap/chat-reply` exists: a host script that scans completed chat-maintainer cards and posts `ack`, `reply` and `final` to the staged thread (`chat-reply:3-8,214-245`) | **`chat-writer` is dropped.** The tracker step and the ceilings go into `chat-reply`. |
| `gchat.py` hard-codes the new-thread fallback (`:249`); the writer must make its own API call | `send` takes `--strict-thread` → `REPLY_MESSAGE_OR_FAIL`, `--request-id`, `--message-id` (`gchat.py:251-263,305-307`); `chat-reply` uses all three (`:129-134`) | R8's send half is **done**. v2 §5.5 shrinks to what is still unverified (§4.5). |
| `SKILL.md:44` needs an owner-approved paragraph (v2's D2) | `SKILL.md:44` already names `bootstrap/chat-reply` as the one automatic exception, "owner-approved", for "acknowledgement, reply and final response" | v2's D2 **no longer exists as a text edit.** What is left is a scope question (D2 below). |
| The chat-maintainer has no network and no credential | It has the `terminal` toolset and a repository token on every instance, and the owner's own token on the personal instance (`profiles/chat-maintainer/config.yaml:6-16`; `docs/decisions/0001`, `0002`) | **v2's R19 argument no longer holds** (§4.4). The agent can read what the owner can read. |
| `sender_trusted` has no host side at all | Still no code *sets* it. But `chat-reply` and `chat-route` now both *require* it `is True`, and `chat-reply` also requires the space to be in `CHAT_ROUTE_SPACES`, at most **eight** spaces (`chat-reply:44-50,78-80`; `chat-route:63-64`) | A real gate exists, and after your D6 it is the **only** gate: inside a listed space `sender_trusted` is true for every human message (§3.3), so the allowlist carries the whole surface. |
| Own-message suppression needs a `SENT:` ledger on the board (v2 §4.2) | Replies carry a `client-ralph-*` message id "Ralph can recognize" (`gchat.py:31-33`; `chat-reply:134`), and `item.json` messages carry `from_ralph` (`SOUL.md:11`) | The ledger is **dropped**; the producer recognises Ralph's own messages by id (§3.2). |
| Request identity is a tracker item; counts live on it | Operation id is `chat-` + sha256 of the latest message name, enforced by both scripts (`chat-reply:80-81`; `chat-route:68`). Sent-once state is a marker file per operation (`chat-reply:27,145-151`). `item.json` lists `prior_operations` for the thread (`chat-route:71-74`) | Counts can live **on the thread's operation directories**, no tracker field needed to enforce a ceiling (§4.2). |
| A closure producer polls the tracker for `Resolved` + outcome | The orchestrator ends every chat graph with one `ralph-chat-maintainer` card that depends on the work cards and writes the final reply (`profiles/orchestrator/SOUL.md:19`; `chat-route:89-92`) | **The closure producer is dropped.** The final card is the trigger (§4.6). |
| Nothing is wired to CI | Still true: the workflows are `scope-check.yml` and `skills-public.yml`, and the latter runs two named evals (`skills-public.yml:22-23,33,36`). `evals/test_chat_reply.py`, `test_chat_route.py` and `test_gchat_send.py` are in neither | The bound R15 asks to prove is **still unenforced between hand runs** (§4.7). |
| Inbound chat is read by a producer (v2 PRs 2-3) | **Still no producer.** `SKILL.md:46` says the host scan "does not start inbound Chat polling", and nothing in the repository writes `chat-inbox/<op>/evidence/item.json` | The producer is **still the largest missing piece**, and it now has a fixed schema to meet (§3). |
| No tracker integration | None. `grep -ri redmine` over the clone finds nothing | Unchanged; the tracker work needed your D1, which you have now approved (§0.1). |

**The three places shipped code and the PRD disagree** (your D2 chose the shipped side; §0.1):

1. **Acknowledgement text.** PRD R6: one short sentence that Ralph has it. v2 made it a writer constant. Shipped: the agent writes `ack` "in your own words, saying what you understood and that you are working on it" (`SOUL.md:17`).
2. **The "cannot help" reply.** PRD R7: "I am not sure how to help with that." Shipped: the agent writes a plain refusal "with the reason and where they could go instead" (`SOUL.md:18`), and one decision, `reply`, covers a refusal, a direct answer and a clarifying question alike.
3. **A request that fails.** PRD A14: silence toward the requester. Shipped: "If the work failed or is incomplete, say so plainly" (`SOUL.md:23`).

**And a fourth, which is this design rather than shipped code.** Your D6 makes the
eligible surface every human message in an approved room, with no mention and no
direct messages (§3.3). PRD R18 says the surface is a direct message or an
@mention. All four go to card `t_73bf3c94`, which amends `prd.md` to v6 after
this document merges — **your D7, approved**.

## 0.1 What you decided, and what each answer moved

Your review comment on `dd.md:72`, 2026-11-06. Quoted exactly, because three of
these change the design rather than confirm it.

| You said | It settles | What moved in this document |
|---|---|---|
| **D1 — Approved** | The tracker's REST API goes on, Ralph gets its own tracker account and a per-user API key. | §5.1's `rails runner` fallback is **dropped**, with it the second trust boundary and the "skips validation" cost. PRs 1, 7 and 8 lose their gate; the only prerequisite left is the key file on the host, which is one line of PR 9's runbook. |
| **D2 — Let's go with the recommendation** | Where shipped `chat-reply` and the PRD disagree, **the shipped behaviour wins and the safety moves from the sentence to the host.** Ralph keeps writing its own acknowledgement and refusal, and tells a requester when work failed. | §4.4's refusal checks are now the **whole** content bound, not one of two options; the conditional branches in §4.4 and §4.6 are gone. No `SOUL.md` line changes, so PR 4 shrinks. And the merged PRD now has three sentences (R6, R7, A14) that describe nothing that runs — **your D7, answered in §0.3, sends them to card `t_73bf3c94`**. |
| **D4 — Don't cap anything. Just check for chats from the last 5 minutes for the account from the room. We will expand to all chats later.** | (b) **No rate limit, ever** — §8 is now a flat non-goal, and the per-sender counter v3 offered is gone. (a) **Taken as approved, by inference**: a thread with an open question stays eligible for that requester without a mention (§3.3). Your sentence does not mention thread continuation; I read "don't cap anything" as covering it, and it is the narrower of the two things D4(a) could mean. Behaviour is identical either way — the wide reading of the *same* sentence is D6, which I did not take. (c) `CHAT_ROUTE_SPACES` **stays** as the v1 boundary; "all chats" is a later change and not this design. And new: the producer reads a **five-minute window** (§3.4), which fixes its scan shape, its tick, and what it loses. | §3.3 drops the conditional, §3.4 is new, §6's PR 2 gains the window, and two of v3's assumptions retire. |

**Superseded the same day, by your D6 (§0.3).** D4(a) was a carve-out *from* the
@mention test. The mention test is gone, so the carve-out has nothing left to
carve: a requester's answer in an open thread is now eligible because the room
is approved, not because a question is open. §3.3 no longer reads markers to
decide eligibility, and E16 passes for a simpler reason.

**What "don't cap anything" is *not* read as.** The per-thread reply ceilings in
§4.2 — at most two clarifications, one decline, one acknowledgement, one final —
stay. They are not a volume cap on requesters; they are the bound that makes R15
grantable at all: the reason "Ralph may send to chat" is not the same sentence as
"Ralph may send to chat as often as it likes". Removing them removes the only
limit on how many times Ralph can message one person about one request. If you
meant those too, say so and §4.2 goes with them — but then R15 has no host-side
enforcement and the eval in §4.7 has nothing to assert.

## 0.2 The split proposal, and what v5 did with it

A proposal arrived on the card on your behalf: move the tracker step out of
`chat-reply` into its own host script, so that no one process holds both the
chat credential and the tracker key. **The concern is right and the split is
taken** (§2, Option C). The *mechanism* is not the one proposed, because three
of its premises do not hold at `95bd4d9`.

| The proposal said | At `95bd4d9` | So v5 |
|---|---|---|
| "After the operation is routed", `tracker-sync` records the item | There is **no routed signal**, and nothing else routes: `chat-reply` runs `chat-route` itself, in the ack branch, through `docker exec` (`chat-reply:194-196`). `chat-route` writes no file; it prints `routed: t_…` (`chat-route:102`) and `chat-reply` reads that line and discards it (`:197`) | drops the rendezvous. `chat-reply` **invokes** `tracker-sync` where it already invokes `chat-route`, and gates the ack on its exit code — the shipped pattern at `:193-199` |
| The two scripts coordinate through "the marker files that already exist" | Partly. A watcher *can* retry a close off `.final-sent`, which is what the proposal assigns to `tracker-sync` and what `--settle` does. What the markers cannot carry is the **payload**: they are empty files created `O_EXCL` (`:145-147`), so "Resolved plus the outcome" needs the card anyway, and a marker that held the outcome would stop being a sentinel and become a record with a schema — the test the proposal priced in | keeps the markers empty and lets `--settle` read the card for the outcome (§5.2). `chat-reply` is not the retrier: its own loop skips an operation once the marker exists (`:239-242`) |
| "Up to one extra 60-second tick before the ack", avoided by "running tracker-sync and then chat-reply from the same LaunchAgent in order" | `ProgramArguments` names **one** script (`install-chat-reply-launchd:18`), and the installer refuses to replace a plist whose settings differ (`:44-46`). Two scripts in one job is a new plist and an ops step on your machine, not a config line. A separately ticked `tracker-sync` would also run **outside** the per-thread `flock` §4.2 puts in `chat-reply` — two processes, one item, no shared lock, which is the bug v2's review caught | adds no job and no tick. One invocation, inside the lock, no latency at all |

**The question it asked me to confirm.** Yes: only the acknowledgement waits on
the tracker. R6 says "Never before the item exists" (`prd.md:305`), so the ack
is gated. R9 says a clarification receipt "is not an acknowledgement"
(`prd.md:326-327`), so a `clarify` sends first and its pending record (R5,
`prd.md:297-301`) lands on the settle pass; a `decline` records nothing at all,
because R7 says "No item is created to justify the reply" (`prd.md:314-315`);
an `answer` records nothing either. Holding a question because a tracker is
down would be the wrong trade.

**One honest subtraction from the proposal's case.** "Neither script holds both
credentials" is true of the *code*, not of the *process*. Both run as you, on
your machine, from the same directory; `tracker-sync` could shell out to
`gchat.py` exactly as `chat-reply` does (`chat-reply:128-134`). The split is a
review and eval boundary — one grep proves which file can send — not an
isolation boundary. It is still worth having, for the reason the standards give:
`chat-reply` stays one sentence without an "and".

## 0.3 What you decided on v5, and what each answer moved

Your two review comments on `Zhachory1/docs#7`, 2026-11-06, anchored at
`dd.md:136` and `:137` of v5 (`3ee3ee0`). Quoted exactly. The first one changes
the design; the second closes it.

| You said | It settles | What moved in this document |
|---|---|---|
| **D6 — "Keep it only to approved chat rooms please" / "@ mentions and direct messages can come later"** | The **wide** reading, bounded by the room list. A message is eligible because of *where* it was sent, not because of who it names: every human message in a `CHAT_ROUTE_SPACES` space, inside the five-minute window, is an operation. The @mention test and the direct-message surface are **deferred, not narrowed** — "later" is a later design, not a setting in this one. | §3.3 is rewritten: the mention test is gone, the direct-message path is gone, and eligibility is `space ∈ CHAT_ROUTE_SPACES` and a human sender who is not Ralph. §3.2's load-bearing assumption **shrinks** — mention annotations are no longer needed, only `clientAssignedMessageId` and `sender.type`. D4(a)'s carve-out and its marker read are gone with the test they carved. E14 and E16 change, §8 names direct messages and the @mention test as non-goals, and PRD R18 joins card `t_73bf3c94`. |
| **D7 — "Approved. Update the PRD"** | The PRD is reconciled to what you chose, in its own pull request, after this one merges. | §8's "amending the PRD" non-goal now names the card, **`t_73bf3c94`**, which covers R6, R7 and A14 (your D2) and R18 (your D6). The decisions table loses D7. Nothing here waits on it. |

**The cost of D6, stated plainly, because it is the one you may want back.**
With no mention test and no volume cap (your D4), *every* human message in a
pilot room becomes a staged operation, an intake card and one model run —
including a conversation between two other people that has nothing to do with
Ralph. The agent's `ignore` decision (`SOUL.md:19`) is what stops those at one
card each; nothing stops the card itself. Three things bound the damage and
none of them bound the volume: the room list is at most eight spaces
(`chat-reply:49-50`), Ralph's own and bot messages are never staged (§3.2), and
the per-thread reply ceilings still cap what any one thread can be told (§4.2).
So the pilot rooms should be rooms you are willing to have Ralph read in full.
If that is not what you meant, one sentence puts the mention test back and §3.3
reverts; nothing else in the design depends on it.

---

## Review header

### Decisions needed from you: one

Numbers never repeat across revisions. v3's **D1-D4 are answered** (§0.1) and
v5's **D6 and D7 are answered** (§0.3). What is open is **D5**, and it gates an
installation step rather than anything in the design: every pull request in §6
is inert on merge either way, and D5 decides only whether PR 3's scan runs on a
timer or when you run it.

| # | Decision | Why it is yours | If you say no |
|---|---|---|---|
| **D5** | **Install a second host LaunchAgent for the producer**, beside the one for `chat-reply` (`bootstrap/install-chat-reply-launchd:19`, a 60-second scan). This is v3's D3, still open: you asked why it cannot run in the container instead. **§3.5 answers that in full** — the short form is that the credential's key is derived from the machine id and the unix user (`gchat.py:106-114`), so the token does not decrypt anywhere else, and the only ways around that put a Chat send credential inside the container where the profile that reads untrusted chat text runs. | Installing a job that runs as you, on your machine, every minute, is yours and not mine. | Run `chat-producer --scan` by hand when you want Ralph to notice new messages. Everything after that is already automatic. Nothing else in the design changes. |

**Two things that are yours and are not decisions.** Run one threaded reply in a
space of your own and confirm it lands in the thread, before the first real
send: `test_gchat_send.py` proves the code asks for `REPLY_MESSAGE_OR_FAIL`; it
cannot prove what the Chat service does with an old thread id. I cannot tell
from the repository whether you have done this. `SKILL.md:46` says "after a
controlled test". And choose the pilot rooms knowing what D6 now means — Ralph
reads every human message in them (§0.3).

### Council dissent

v2's independent review found three blocking issues, all accepted and fixed:
`inbox-writer` blocks nothing; the lock was on the operation rather than the
request; and thread-scoped eligibility was settled in prose rather than
escalated. **v3, v4 and v5 keep all three fixes** — the block precedent is cited at
`maintain-pr` (§4.8), the lock is per thread (§4.2), and the eligibility
widening went to you as D4(a), which you approved. v3 reached you without an
independent review; **v4 was reviewed** and raised one blocking finding — two
new cross-references pointed at a decision number that no longer exists (D3
rather than D7) — fixed in that revision. **v5 was reviewed too**: one blocking
finding — `--record` would deadlock the acknowledgement path if its own marker
were written `O_EXCL` like every other marker in the design — and three notes,
all accepted and fixed below. The reviewer re-checked every citation at
`95bd4d9`, including all of v5's new ones, and found them exact but one, which
is corrected.

**The live dissent is with the split proposal** (§0.2). Its concern is taken and
its mechanism is not: there is no routed signal to key on, the markers it would
coordinate through are skip-sentinels rather than a channel, and one LaunchAgent
cannot run two scripts in order without a replacement plist. The costs it listed
— a tick of latency, a marker schema, a `chat-reply` config switch — all come
from the rendezvous, and all disappear when the sender simply calls the script.
I also disagree with one line of its case: the split is **not** a credential
boundary, because both scripts run as you on one host and either could shell out
to `gchat.py`. It is worth doing anyway, for the smaller and truer reason.

The dissent v4 carried forward, with your own words, still stands (§0.1):
"don't cap anything" is implemented as *no rate limit on requesters*, and the
per-thread reply ceilings stay.

**What v6 closes, and the one disagreement that remains.** The oldest dissent in
this document — v2's third blocking finding, that the eligible surface must not
be widened inside the prose — is now settled by you rather than by me: D6 widens
it further than D4(a) did, deliberately, and §3.3 says so in your words. I have
no objection to the decision and one to its framing: "keep it only to approved
chat rooms" sounds like a restriction and is read here as a widening, because
inside those rooms it removes the only sender test there was. §0.3 states the
cost that follows, so that if the sentence meant the other thing, it is one
sentence to put back.

### Scope: medium — nine pull requests

Unchanged in v6, and one of them gets smaller: PR 2 no longer has to read or
test mention annotations (§3.3). Seven in v4; v5 added the two smallest in the
set, a no-behaviour extraction of the evidence reader (PR 6) and the two call
sites that wire the tracker script in (PR 8). Breakdown in §6.

### What changed since v5

Your two answers, and nothing else. No citation moved and no new claim about
the code was added.

- **§0.3 is new**: your D6 and D7, quoted, with what each moved and what D6 costs.
- **§3.3 is rewritten**: eligibility is the approved room plus a human sender who is not Ralph. The @mention test, the direct-message path and D4(a)'s open-question carve-out are all gone — the first two deferred by you, the third left with nothing to carve.
- **§3.2's load-bearing assumption shrinks**: A3 no longer needs mention annotations, only `clientAssignedMessageId` and `sender.type`. The riskiest unverified field is the one that dropped out.
- **The PRD reconciliation is a card, `t_73bf3c94`**, gated on this one, covering R6, R7, A14 and now R18. §0 and §8 name it; D7 leaves the decisions table.
- **E14 and E16 change** (§7), and §8 gains two non-goals: direct messages, and any @mention test.
- **Decisions: D5 only.** It gates installing a LaunchAgent, not the design.


---

## 1. The answer, in one page

Ralph already runs half of this: *an intake card, an agent that only decides, a
host script that posts what it decided*. The tracker needs three additions and
nothing else.

```
chat ─> [NEW] bootstrap/chat-producer ─> chat-inbox/<op>/evidence/item.json + intake card (queues)
        (host, owner's user)                         │
                                                     ▼
                                            ralph-chat-maintainer   (decides: route | reply | ignore)
                                                     │
                                                     ▼
        bootstrap/chat-reply  (host, exists) ── [NEW] ceilings ── send ──> thread
          chat credential, no tracker key   │
                │                           └─ calls ─> [NEW] bootstrap/tracker-sync
                │                                        tracker key, no send path
                ├─ ack:   chat-route makes the orchestrator card, THEN `tracker-sync --record`, THEN the reply
                ├─ clarify: the question goes out, THEN the pending record on the next `--settle`
                └─ final: the card the orchestrator ends the graph with; THEN `--settle` closes the item
```

1. **The agent still cannot send to chat.** It holds no chat credential: that stays on the host (`config.yaml:3-5`). It *can* now read repositories with a token (§0), which is why §4.4 changes.
2. **The host cannot improvise.** Destination comes from staged evidence, never agent text (`chat-reply:64-84`). That stays. New: the host also decides *whether* a reply may go, from counts it reads itself (§4.2).
3. **The record precedes the acknowledgement.** `chat-reply` already holds the ack until `chat-route` confirms an orchestrator card (`:193-200`). `tracker-sync --record` joins that same step, so "Ralph says it has the work" and "the work is recorded" cannot come apart (R9). Nothing else waits on the tracker (§0.2).
4. **No new datastore.** Sent-once state is the marker files that exist. The thread-to-item map is two fields on the tracker item. Nothing else is added (R14).

---

## 2. Options considered

Three ways to add the tracker. Standards require at least two.

| Option | For | Against |
|---|---|---|
| A. Put the tracker step inside `chat-reply` (v4's choice) | Fewest files. One place decides whether a reply goes and whether it was recorded. No new contract between processes. | `chat-reply` would carry ack, reply, final, ceilings, refusals **and** the tracker. Its one-sentence description grows an "and", which is the standards' test for a file that is too big. The tracker key would be read into the process that sends. |
| B. Write the v2 `chat-writer` beside it | Cleaner single responsibility; v2's eval design applies as written. | Two host senders with one policy line. Two places to forget a ceiling. Two LaunchAgents. It re-implements what shipped three days after v2 was written. |
| **C. One sender, one tracker script, called by the sender (chosen)** | `chat-reply` stays "post the maintainer's replies to the original thread" and gains the ceilings and the refusals, which are *about replies*. `bootstrap/tracker-sync` holds the tracker key, imports nothing from the sender and contains no send call, so one grep over the host scripts still proves there is exactly one way to send (§4.7). It is the shape `chat-reply` already uses for `chat-route` (`:194-199`), so the gating, the retry and the failure path are the ones in the file today. | A second file, a second eval, and one more exit-code contract to get right. `chat-reply` still *drives* the tracker step: the ordering lives in the sender even though the tracker code does not. Two scripts must agree on the workspace layout. |

**Why C over A.** Both are safe; C is the one that keeps the file describable.
The decisive argument is not credential isolation — §0.2 says plainly that the
split is a code boundary and not a process boundary — it is that §4.7's bound is
checked by reading one file, and that file should not also be where tracker
behaviour lives.

**Why C is not a rendezvous.** The two scripts do not meet in a shared tick or a
shared file. `chat-reply` calls `tracker-sync --record <op>` before the
acknowledgement and `tracker-sync --settle` once per scan; `tracker-sync` never
reads the board for a decision and never decides whether a reply may go (§5.2).
That keeps one reader of the decision metadata, one writer of the reply
markers, and one lock.

If you would rather have the split exactly as proposed — its own LaunchAgent, on
its own tick — say so. It costs a replacement plist (`install-chat-reply-launchd:44-46`),
one tick of acknowledgement latency, and a shared lock that neither script has
today.

---

## 3. The producer: the missing half

The producer is **the same job v2 described**, now with a contract it must meet
rather than one it could choose. It runs on the host as the owner (§3.5), calls
`spaces.messages.list` over a five-minute window (§3.4), and for each eligible
message stages the evidence and creates one intake card on the `queues` board.
`bootstrap/inbox-producer` is the shape to copy: the same host-side read, the
same `stage_once` evidence at mode `0600` under a `0700` directory, the same
`hermes kanban … create --idempotency-key` through `docker exec`
(`inbox-producer:29-33,137-154`).

### 3.1 The schema it must write

`chat-reply` and `chat-route` read, and refuse anything that differs from:

- directory `chat-inbox/chat-<sha256 of the latest message's resource name>/`, with `evidence/item.json` mode `0600`, a regular file, one link, under 64 KiB, no symlinks (`chat-reply:64-84`; `chat-route:38-48`);
- `kind: "inbound"`, `sender_trusted`, `space`, `thread` (`spaces/<s>/threads/<t>`), and `messages` oldest first, each `{name, text (list of lines), from_ralph}`;
- `prior_operations`: up to 20 earlier operation ids from the same thread (`chat-route:71-74`);
- an intake card on `queues`, assignee `ralph-chat-maintainer`, body containing exactly one `operation_id:` line, workspace `/workspaces/chat-inbox/<op>` (`chat-reply:171-172`), idempotency key = the operation id.

Two consequences. First, the idempotency key v2 proposed is now **fixed**: it is the hash of the message name, and E2 follows from it. Second, a thread's whole history is staged on every operation — the maintainer reads it and treats a later message as steering (`SOUL.md:13`) — which is how R20's answer reaches the right request without a second item (E16).

### 3.2 R10: Ralph's own messages

Replies are created with `messageId = client-ralph-<kind>-…` (`chat-reply:134`). The producer marks `from_ralph: true` for any message whose `clientAssignedMessageId` starts with `client-ralph-`, and stages **no operation** for such a message. Bot senders (`sender.type = BOT`) are skipped the same way. The agent's `ignore` decision (`SOUL.md:19`) stays as the backstop and costs one card and one model run.

**Unverified, and load-bearing:** that `spaces.messages.list` returns
`clientAssignedMessageId` and `sender.type`. They are documented as part of the
Message resource, which I am recalling, not reading. PR 2's first act is one
read-only call to confirm them. If either is absent the design changes there,
not later. This is v2's A3, and **your D6 made it smaller**: the mention
annotations it also depended on are no longer read by anything (§3.3), so the
part of A3 that was hardest to be sure of is gone. What is left are two scalar
fields, and the failure if one is missing is now bounded — without
`clientAssignedMessageId` the `ignore` backstop carries R10 alone, at one card
per reply Ralph posts; without `sender.type` a bot notice costs one card.

### 3.3 Eligibility: the approved room is the whole gate (your D6)

`sender_trusted` is set `true` when **all four** hold, and there is no fifth
test:

1. the message's space is in `CHAT_ROUTE_SPACES` — the same list `chat-reply`
   reads from the same instance file, so the two scripts cannot disagree about
   it (`chat-reply:44-50`), capped at eight spaces (`:49-50`);
2. it was created inside the five-minute window (§3.4);
3. its `sender.type` is a human, not `BOT`;
4. its `clientAssignedMessageId` does not start with `client-ralph-` (§3.2).

Nothing is asked about *who* the human is, and nothing is asked about what the
message says. A message is eligible because of the room it was sent in.

**What that drops, and on whose word.** Your D6: *"Keep it only to approved chat
rooms please"*, and *"@ mentions and direct messages can come later"*. So the
@mention test v4 and v5 carried is gone — the producer reads no annotations and
never asks whether you or Ralph were named — and direct messages are **not a
surface in v1** at all, because a direct message is not one of the rooms on the
list. Both come back as a later design, not as a setting. D4(a)'s carve-out for
a requester answering an open question goes too: it existed only to let an
unmentioned reply through the mention test, and there is no longer a test for it
to pass. The producer therefore reads **no markers and no tracker** to decide
eligibility; `.clarify-sent` and `.final-sent` are still counted, but only by
the ceilings (§4.2), and only after the agent has decided.

**What it costs, in one place so it is not buried.** Every human message in a
pilot room is an operation, an intake card, and one model run — your
conversation with someone else in that room included. The `ignore` decision
(`SOUL.md:19`) is what ends those, one card each, and your D4 waived any volume
cap, so nothing bounds the count except how busy the room is. Choose pilot
rooms accordingly; §0.3 says what reverts this if it is not what you meant.

**What `sender_trusted` means now, honestly.** It stops being a judgement about
a person and becomes a restatement of condition 1: inside a listed space it is
true for every human, outside one the message is never staged. `chat-route`'s
`item["sender_trusted"] is not True` check (`:63-64`) and `chat-reply`'s
(`:78-80`) still run and still refuse a hand-written file, so the field keeps
its defensive value against a malformed or forged `item.json` — it just no
longer encodes a second policy. The one gate that matters is the room list, and
it is yours: `CHAT_ROUTE_SPACES` is an instance file on your machine, not a
constant in the code.

**This diverges from the merged PRD, by your instruction.** R18's surface is a
direct message or an @mention, and R18 says the feature adds no allowlist of its
own. v1 as you have now decided it is the opposite on both counts: an allowlist
of rooms *is* the surface, and no mention is needed inside one. That is a
decision you made with the trade-off in front of you, and it is the fourth item
on card `t_73bf3c94` (§0, D7). Until that card merges, `prd.md` R18 and this
section disagree and **this section is the newer one**.

`SOUL.md:11` already says the host sets `sender_trusted` from "the approved
intake policy". This section *is* that policy; nothing in the PRD needs a new
allowlist beyond the one `chat-reply` already enforces.

### 3.4 The read: five minutes, every scan (your D4)

Your instruction fixes three things v3 left loose.

**What it reads.** One `spaces.messages.list` per space in `CHAT_ROUTE_SPACES`,
with a filter of `createTime > (now − 5 minutes)`, newest first. Nothing walks
history, nothing backfills, nothing keeps a cursor.

**How often.** Every 60 seconds, the same `StartInterval` as the reply scan
(`install-chat-reply-launchd:19`), so each message falls inside roughly five
consecutive windows. The repeats cost nothing: the operation id is the sha256 of
the message's resource name (`chat-reply:82`), and the card carries it as the
idempotency key, so a message already staged produces no second directory and no
second card — the same no-op the mail producer relies on
(`inbox-producer:19-22,152`). v3's 900-second tick is retired.

**What it loses, stated plainly.** A message older than five minutes when the
scan runs is never seen, and nothing notices. If the machine sleeps, the
LaunchAgent is unloaded, or the network is out for six minutes, the requests
sent in that gap are lost silently — to Ralph; the requester sees no reply and
has no way to tell whether the message was read. That is the cost of "just check
the last 5 minutes" and it is accepted. A watermark (last successful scan time,
one file, bounded to a day) would close it and is the obvious later change; it
is a non-goal here because you did not ask for it.

**Why the producer does not shell out to `gchat.py messages`.** That wrapper
keeps five fields per message — name, createTime, sender display name, text,
thread — and drops everything else, including `clientAssignedMessageId` and
`sender.type` (`gchat.py:222-240`); `--json` does not bring them back, because
they are discarded before the output is built. Those two fields are exactly
what §3.2 runs on, and after your D6 they are the only two the producer needs
beyond what the wrapper keeps. So the producer imports the credential helpers
from the same module (`_key`, `_load_creds`, `_svc`, `gchat.py:106-136`) and
makes its own read-only `list` call. `gchat.py`'s own commands are unchanged,
and `messages().create` stays reachable through exactly one code path
(`gchat.py:264`), which is what §4.7's eval asserts.

**Unverified, with §3.2's fields:** that `spaces.messages.list` accepts a
`filter` on `createTime`. If it does not, the producer pages newest-first and
stops at the first message older than the window — same behaviour, one more
page read. Either way PR 2's first act confirms it.

### 3.5 Why this cannot run in the container (the question you asked on D3)

Four parts, shortest first.

**1. The token is bound to the machine and the user, cryptographically.**
`_key()` derives the Fernet key with HKDF over the string
`"<machine id>:<unix user>:gws-cli"` and a salt from `gws_config.json`
(`gchat.py:106-114`). The machine id is `IOPlatformUUID` from `ioreg` on macOS
and `/etc/machine-id` on Linux (`:72-103`). In the container all three inputs
differ: Linux rather than Darwin, user `hermes` rather than you, the image's
machine-id rather than your Mac's. So `chat_token.json.enc` fails to decrypt —
`_load_creds` returns `None` (`:129-136`) and the client secret exits with
"Cannot decrypt client_secret (machine changed?)" (`:117-126`). Mounting
`~/.config/gws-cli` into the container does not help: the key is not in the file.

**2. Nothing is mounted anyway.** The hermes service mounts the Hermes home, the
workspaces, the sandbox root, and read-only copies of `evals`, `bootstrap`,
`skills` and the private skills — and nothing else
(`compose/docker-compose.yml:158-183`). There is no path from inside to the
credential, by construction rather than by omission.

**3. The ways to change that are the ways the design exists to prevent.** Either
drop the machine binding from `_key()` — which is shared with `gws-cli` by
design, same `APP_ID`, same encrypted client secret (`gchat.py:54-59`), so it
weakens a credential you use elsewhere — or run `auth` inside the container and
keep a second long-lived Chat token there. Both end with a credential that can
call `messages.create` sitting in the container where `ralph-chat-maintainer`
runs, with a terminal and a repository token (`profiles/chat-maintainer/config.yaml:6-16`),
reading untrusted chat text as its whole job. The design's first sentence is
that this is impossible: "Holding no chat credential is enforced by where the
credential lives, not by this file or the SOUL" (`config.yaml:3-5`), and the
skill names exactly one automatic sender (`SKILL.md:44`). Host-side ceilings
(§4.2) and the content refusals (§4.4) are only a bound if there is no second
way to send; a token in the container is that second way.

**4. "On the host" does not mean outside Ralph.** The host scripts already reach
into the container for everything board-side: `chat-reply` runs
`docker exec … hermes kanban` to read a card and `chat-route` to make one
(`chat-reply:157,194,222`), and the mail producer does the same
(`inbox-producer:29-33`), for the same reason, in your own earlier design —
"Runs on the host, where the Gmail token lives; the token never enters the
hermes container" (`inbox-producer:4-5`). The chat producer is that shape: read
Chat on the host, create the card inside. The credential is the only thing that
stays out.

So the open question is narrow, and it is D5: a LaunchAgent, or you run the scan
by hand.

---

## 4. R15 and R8: how exactly the replies are bounded

Shipped `chat-reply` bounds three things: where a reply goes, that it goes once
per operation, and that a final reply follows an acknowledgement. It does **not**
bound how many replies a *request* gets, because it counts per operation (per
message), and every message in a thread is a new operation. PR 5 adds the
rest.

### 4.1 One decision, four meanings: `reply_kind`

`reply` today means a refusal, a direct answer, or a clarifying question
(`SOUL.md:18`), and the host cannot tell which. The decision gains a
`reply_kind` in `{clarify, answer, decline}` in its metadata, and
`chat-route`'s allowed-key set (`:56`) grows by that one key and by `kind`
(below). Markers become per kind: `.clarify-sent`, `.decline-sent`,
`.reply-sent` (answer), `.receipt-sent`, `.final-sent`.

`kind` — one of R4's four actionable kinds — is recorded on `route`, because E10
needs it: an `update` is acknowledged and nothing more. `chat-route` omits the
"finish with a final card" line from the orchestrator brief when the kind is
`update` (`chat-route:89-92`), so no final card exists to send.

### 4.2 The ceilings, read by the host from the thread

The **request is the thread**. `chat-reply` takes an exclusive `flock` on a lock
file named for the thread, before it counts and until it has marked, and then
counts markers across this operation's directory and every `prior_operations`
directory in `item.json` (host-staged, never agent text):

| Rule | Ceiling | Refusal |
|---|---|---|
| A17 | at most 2 `clarify` per thread (one question, one follow-up) | the reply is refused, the card is blocked (§4.8), the request stays pending |
| A15 | at most 1 `decline` per thread, and a decline creates **no** tracker item (R7) | refused and blocked |
| R6 | at most 1 `ack` per thread, and only after the tracker item exists | held, retried next scan |
| R11 | at most 1 `final` per thread, and only after `ack` (shipped, `:183-184`) | refused |

The lock is on the thread and not the operation for the reason v2's reviewer
found: several messages in one thread are separate operations, and
operation-scoped locks never see each other. A second host is out of scope
because the credential cannot be decrypted on a second machine (§3.5).

**These are not the cap your D4 waived.** D4 removed any limit on *requesters* —
how many people may ask, how often, how much work they create. The table above
limits *Ralph*: how many times one request may be messaged about. Four rules,
all of them from the PRD (R6, R7/A15, R11, A17), and together they are the
host-side half of the R15 grant.

### 4.3 R8, what is now true and what is not

**Done:** the send asks for `REPLY_MESSAGE_OR_FAIL`; a strict send without a thread, or with a thread in another space, fails before any API call (`gchat.py:251-252,258-259`; `test_gchat_send.py:34-50`).

**Not done, and not verifiable from here:** that the service honours it for a thread id captured earlier, and how old an id may be. v2's third check — compare the returned thread to the staged one — is **not available**: `cmd_send` returns only `name` and `createTime` (`gchat.py:266-267`), and `chat-reply:post` accepts `sent is True` alone. PR 5 adds a cheap partial check: the returned message `name` must begin with the staged space. It cannot detect a wrong thread in the right space. The pre-launch check under the decisions table is what closes that gap, and the token holds no delete scope (`gchat.py:60-66`), so a wrong-thread message can be reported and not withdrawn.

### 4.4 R19: the content bound, rewritten

v2 argued that the agent *could not reach* anything outside one request's evidence directory. That is no longer so. The chat-maintainer has `terminal` and a repository token (`config.yaml:6-16`); on the personal instance that token is the owner's (`docs/decisions/0002`), and both decisions record the exposure as accepted. A reply can now quote a private repository the requester cannot open.

**So the bound moves to the host, and it is a refusal, not a trim.** Before any send of agent text — `ack`, every `reply_kind`, `final` — `chat-reply` refuses text that contains:

- a tracker reference: the instance host, an `/issues/<n>` path, or a bare `#<n>` form (R1);
- a URL not in a `reachable` list the host builds, where a URL qualifies only if it is provably public (for a GitHub pull request, one API read showing the repository is public; everything else is named in words, R19);
- a credential-shaped string (the SOUL already says never; `SOUL.md:29`).

**The honest limit, stated plainly:** prose cannot be proven to come from this request. A refusal rule catches references and links; it cannot catch a sentence that *paraphrases* a private document. Your D2 answer makes these checks the whole content bound — there is no constant-text fallback behind them — so that residual risk is accepted rather than designed away. It is the same class of risk the ADRs already accepted for repository access, and it is smallest on `ack` (a restatement of what the requester just wrote) and largest on `final` (a summary of work Ralph did elsewhere).

### 4.5 The `SKILL.md:44` policy line

v2's request for an owner-approved edit is gone: the line is already edited and names `chat-reply`. It says "acknowledgement, reply and final response" and does not mention the tracker, ceilings or `reply_kind`. PR 5 amends that sentence to match what the script then does, in the same pull request as the code, so the policy and the behaviour change together.

### 4.6 The close: the final card is the trigger

The orchestrator ends a chat graph with one `ralph-chat-maintainer` card whose parents are the work cards (`orchestrator/SOUL.md:19`). Kanban dependencies mean it does not run until every parent is done, so **one work card finishing is not the request finishing** (R11, E7) by construction, and no tracker poll is needed. `chat-reply` sends the `final` and marks it; the next `tracker-sync --settle` sets the item to `Resolved` with the outcome and writes its own `.tracker-closed` sentinel, so a tracker outage delays the close and never re-sends the reply. The tracker still reads as canonical (R13) because every stage is written: created at the ack, `Waiting on requester` after a `clarify`, `Resolved` after the final.

**Silence on failure (E11), and what your D2 did to A14.** Partly it follows from the graph: a work card that is blocked or abandoned never lets the final card run, so nothing is sent. But a final card whose parents *completed with a failed result* is told to "say so plainly" (`SOUL.md:23`), and your D2 keeps that. So E11 splits in two: no reply when the work never finished, one honest reply when it finished badly — still bounded by the one-`final`-per-thread ceiling and the §4.4 checks. PRD A14 asks for silence in both cases, and is the third of the four items card `t_73bf3c94` amends (your D7, §0.3).

### 4.7 The eval, and the CI gap

PR 5 extends `evals/test_chat_reply.py` and adds the workflow that runs it. The tests it needs, beyond the fifteen it has:

| Assertion | Guards |
|---|---|
| a third `clarify` in one thread sends nothing and blocks the card | A17, E18 |
| a second `decline` in one thread sends nothing; a decline calls no tracker method | A15, R7, E3 |
| two operations in one thread serialise on the thread lock | A17 |
| a body containing the tracker host, `/issues/12`, or `#12` is refused | R1, R19 |
| a URL outside the `reachable` list is refused; the same outcome in words is sent | R19, E9 |
| a tracker write failure sends no acknowledgement | R9, E1 |
| `chat-reply` imports no tracker module, and `tracker-sync` contains no send call | §2, R15 |
| a second `--record` for one operation exits 0 and creates no second item | §5.2, R6/R9 |
| `--settle` closes an operation whose reply marker already exists | §5.2, E8 |
| `kind: update` leaves no final card for `chat-reply` to send | E10 |
| exactly one `messages ( ) . create` path exists in the host scripts, and `REPLY_MESSAGE_OR_FAIL` is the only reply option used | R15, R8 |

**The gap, still open.** `.github/workflows/` holds `scope-check.yml` and `skills-public.yml`; the second runs two named evals only. None of the chat evals run in CI, so today nothing fails a build if someone adds a second send path or removes a ceiling. PR 5 ships the workflow with the code, triggered on changes to `bootstrap/chat-*`, `skills/ops/google-chat/**` and `evals/test_chat_*`. Wiring the older mail eval in is one line and an observation (§9), not done here.

### 4.8 How you see a failure

`chat-reply` signals trouble by exit code and stderr only: `1` for retry, `2` for refused (`chat-reply:160,190,198,208`). It blocks nothing and tells nobody, so a refusal — a ceiling, a R19 check, a tracker failure — is invisible, and a retryable failure repeats every 60 seconds without a word. PR 5 therefore adds the two existing mechanisms from `maintain-pr`:

1. **Block the card** through `hermes kanban block`, with a reason naming the reply kind, the thread and the cause (`maintain-pr:396`);
2. **Say so at once** with the best-effort webhook post that tolerates its own failure (`maintain-pr:301-303`, `RALPH_NOTIFY_URL` or `GCHAT_WEBHOOK_URL`).

Both fire on a **refusal (exit 2) and on a retry (exit 1) alike**, because the
failure that matters most is the one that repeats quietly: a `--record` that
keeps failing holds an acknowledgement for a request Ralph has, and nothing in
the exit code says whether the next scan will do better. The alert is posted
once per operation and reply kind, guarded by an `.alerted-<kind>` sentinel, so
a loop reports itself once rather than every minute.

`notifier/poller.py` stays the backstop: it polls blocked cards, recovers the reason from a `BLOCKED:` comment (`:93-94`), and announces each once (`:121-125`). E6 needs no new machinery.

---

## 5. The tracker, and the remaining unknowns

### 5.1 Unknown 1 — writing to the tracker, settled by your D1

| Path | Verdict |
|---|---|
| **Enable REST, a dedicated Ralph account, a per-user API key** | **Approved by you, 2026-11-06.** One setting, one account, a documented interface, the application's own validation. What remains is operational: create the account, generate its key, and put the key on the host beside the chat token, owner-only (PR 9's runbook). |
| `rails runner` over a shell on the tracker host | **Dropped.** It existed only as the fallback if REST stayed off. With it goes a second credential and a write path that skips validation. |
| Incoming email | Rejected: email is a PRD non-goal. |
| Direct database writes | Rejected. |

All tracker access sits behind one module, `bootstrap/tracker.py`, with `find`, `create`, `update`, `note`. *General product knowledge; the instance was not inspected (A1), so PR 1's eval runs against a fake and the first live call is a manual one.*

### 5.2 `bootstrap/tracker-sync`: one job, two entry points

One sentence: **make the work tracker match the state of a chat operation.** It
holds the tracker key, imports `bootstrap/tracker.py`, and contains no send
call and no policy about whether a reply may go.

| Entry point | Called by | Does | Writes |
|---|---|---|---|
| `--record <op>` | `chat-reply`, in the branch that already holds the ack until `chat-route` confirms an orchestrator card (`:193-200`) | finds or creates the item for the thread, with the staged request text and the card's `ack` as the restatement | `.tracker-recorded`, holding the item id |
| `--settle` | `chat-reply`, once at the top of each scan | for each operation with `.clarify-sent` and no `.tracker-waiting`: create or update the pending record (R5) and set `Waiting on requester`. For each with `.final-sent` and no `.tracker-closed`: set `Resolved` with the outcome from the report card | `.tracker-waiting`, `.tracker-closed` |

The order for an acknowledgement becomes: orchestrator card, then
`--record`, then the reply. A non-zero exit sends nothing and the operation
retries next scan, exactly as a failed `chat-route` does today (`:197-199`).

**Every `tracker-sync` write is idempotent, including its own markers — this is
load-bearing.** `chat-reply` creates its markers `O_EXCL` (`:145-147`), which is
right for a send that must happen once; copying that for `.tracker-recorded`
would be a deadlock. `--record` is re-run whenever anything after it fails: the
item is already created, the marker is already there, and a create-or-fail
marker would make the second call exit non-zero, which `chat-reply` reads as
"retry next scan" — forever, on the one path R6 and R9 exist to protect. So
`--record` treats an existing `.tracker-recorded` for this operation as
**success**, and the same holds for `.tracker-waiting` and `.tracker-closed`.
The item lookup is idempotent by the thread key already.

**How `--settle` finds its work.** It walks the `chat-inbox` directories on
disk and reads their markers. It does **not** reuse `chat-reply`'s
`list --status done` loop, which skips an operation once its reply marker
exists (`:239-242`) — that loop is for deciding whether to send, and a close
has to stay retryable after the sending is over.
Everything the settle pass does happens **after** a reply has already gone out, so a
tracker failure there delays a record and never changes what was said. It is
called `--settle` and not a sweep because A18 uses "sweep" for chasing a
requester, which this never does: it writes to the tracker and sends nothing.

**Why `tracker-sync` reads cards.** `--record` needs the restatement and
`--settle` needs the outcome; both are agent text and both live on a card. It
finds them with the same listing `chat-reply` already makes
(`hermes kanban … list --assignee ralph-chat-maintainer --status done`,
`chat-reply:221-224`), matched by operation id. It never reads a decision in
order to *act* on one: `chat-reply` has already decided by then, and before a
settle pass the reply has already gone out. One reader of policy, one sender,
one writer of reply markers.

**Rule of three, and the shared reader.** All three scripts must validate the
same staged `item.json`, and that check already exists twice — `chat-reply:69-74`
and `chat-route:42-47` are near-copies. `tracker-sync` would be the third, so
PR 6 extracts `bootstrap/chat_evidence.py` (the staged-item reader,
`operation_of`, `decision_of`) and imports it in all three. No behaviour
changes and the fifteen tests in `evals/test_chat_reply.py` and those in
`test_chat_route.py` must pass untouched. `bootstrap` is mounted read-only into
the container (`compose/docker-compose.yml:171`), so the module resolves for
`chat-route` as well.

**Finding the item.** Because the thread is the request, the lookup key is a
custom field holding the thread name; a message in a known thread finds its
item first, and an unconfident cross-thread match becomes a new item with a
`related to` relation (R12, E5, E13).

**Until the key file exists.** `tracker-sync` exits 0 with "tracker not
configured" when there is no key on the host, so `chat-reply`'s gate is
satisfied, the ack goes out, and the feature runs exactly as shipped. That is
what makes PRs 7 and 8 safe to merge before PR 9's runbook is followed.

### 5.3 What an item holds, and its statuses

Stored: the requester's display name, the space and thread names, the request text as sent (truncated), Ralph's restatement, the state and the next action (R3). Not stored: other threads, member lists, attachments. Retention is the item's lifetime; the staged copy is owner-only (`0600`). **Who deletes it, and when:** `tracker-sync --settle` removes the operation's `evidence/` directory immediately after it writes `.tracker-closed`, keeping the marker files so the thread's history still counts (§4.2). Nothing chat-derived enters this repository.

| Status | Means | Reaches the requester |
|---|---|---|
| `New` | recorded, not understood | nothing |
| `Waiting on requester` | a clarification is open (R5, R20) | the question, once, plus at most one follow-up |
| `In progress` | understood and accepted | the acknowledgement, once |
| `Resolved` + outcome | worked to a result | the final reply, once |
| `Rejected` / `Duplicate` / `Closed` | stopped, refused, folded in | nothing, or the one honest "it failed" reply your D2 keeps (§4.6) |

`Waiting on requester` never expires and is never swept (A18). One saved query,
sorted by age, shows the pile; building it is your click. The pending record is
written **after** the question goes out, by `tracker-sync --settle` (§0.2, §5.2);
only `In progress` is gated on a write that already succeeded.

### 5.4 Unknown 6 — who classifies

The chat-maintainer, as shipped. Its decisions stay `route`, `reply`, `ignore`; v3 adds the `kind` and `reply_kind` metadata above and nothing else. "Other" is `reply_kind: decline`. "Unclear" is `reply_kind: clarify`. `final` stays on the second card type.

---

## 6. The implementation pull requests, against `Zhachory1/ralph`

Each under 400 lines and one concern (`STANDARDS.md`). Each is inert on merge; the gate column says what makes it live.

| PR | Concern | Files | Inert because | Gate |
|---|---|---|---|---|
| 1 | Tracker client | `bootstrap/tracker.py`, `evals/test_tracker.py` | nothing imports it | none left (D1 approved); the key file is PR 9 |
| 2 | Chat read, eligibility, five-minute window, schema; no cards | `bootstrap/chat-producer` (dry-run default), `evals/test_chat_producer.py` | dry-run prints, creates nothing | first act: confirm the API fields and the `createTime` filter (§3.2, §3.4) |
| 3 | Producer stages evidence and creates intake cards | same, plus `bootstrap/install-chat-producer-launchd` | the LaunchAgent is not installed | **D5** (install it, or you run the scan) |
| 4 | Decision metadata | `profiles/chat-maintainer/SOUL.md`, `bootstrap/chat-route`, `evals/test_chat_route.py` | adds optional keys; unchanged behaviour without them | none left (D2 settled: shipped text stays) |
| 5 | Ceilings, R19 checks, block and alert, CI workflow | `bootstrap/chat-reply`, `evals/test_chat_reply.py`, `.github/workflows/chat-reply.yml`, `skills/ops/google-chat/SKILL.md` | tightens only; refuses more, sends nothing new | none left |
| 6 | Extract the staged-evidence reader (rule of three, §5.2) | `bootstrap/chat_evidence.py`, `bootstrap/chat-reply`, `bootstrap/chat-route` | no behaviour change; the existing evals pass untouched | none |
| 7 | The tracker script | `bootstrap/tracker-sync`, `evals/test_tracker_sync.py` | nothing calls it | none |
| 8 | Wire it: `--record` before the ack, `--settle` per scan | `bootstrap/chat-reply`, `evals/test_chat_reply.py`, `evals/test_tracker_sync.py` | `tracker-sync` exits 0 "not configured" without a key file (§5.2) | the key file |
| 9 | Turn it on | `compose/README.md` runbook | documentation | **D5**, the tracker account and key, the pre-launch check |

Order: 5 first. It adds no behaviour and closes the most exposure, because the
shipped sender has none of the ceilings today. Then 1, 4, 2, 3, 6, 7, 8, 9. PRs 2 and
3 stay separate for v2's reason: the half that reads untrusted content and the
half that acts are reviewed apart. PRs 7 and 8 are separate for the same reason
in the other direction: the file that can write to the tracker and the change
that lets a reply depend on it are different reviews.

---

## 7. Acceptance examples, mechanism and test

| # | Mechanism | Test |
|---|---|---|
| E1 | `route` → orchestrator card → `tracker-sync --record` → acknowledgement, in that order | `test_chat_reply`: order; one send; a non-zero `--record` sends nothing |
| E2 | Operation id = hash of the message name; card idempotency key = that id | `test_chat_producer`: a second tick over one message creates no second card |
| E3 | `reply_kind: decline`; no tracker call; second decline in the thread refused | `test_chat_reply`: a decline calls `tracker-sync` not at all; the second is refused |
| E4 | `client-ralph-` id and `sender.type` skip the message; `ignore` as backstop | `test_chat_producer`: both are skipped |
| E5 | Thread-keyed lookup first; else a new item with a `related to` relation | `test_tracker_sync` against a fake tracker |
| E6 | `--strict-thread`; failure blocks the card and posts the webhook | `test_chat_reply`: API error blocks; refusal blocks |
| E7 | The final card depends on every work card | `test_orchestration`: one of three parents done yields no final send |
| E8 | The final card's `send` posts once; the next `--settle` closes the item | `test_chat_reply`: a second scan sends nothing. `test_tracker_sync`: the settle pass closes once |
| E9 | `reachable` list; other URLs refused | `test_chat_reply`: private-repo URL refused; the same outcome in words sent |
| E10 | `kind: update`: no final card requested | `test_chat_route`: update omits the final-card line |
| E11 | Blocked or abandoned work never runs the final card; work that finished badly gets one honest reply (your D2, §4.6) | `test_chat_reply`: a blocked parent sends nothing; a failed-but-complete graph sends exactly one `final` |
| E12 | R3 fields and statuses on the item (§5.3) | `test_tracker` for the fields; `test_tracker_sync` for the statuses |
| E13 | Brief says decide and link, never stop | reviewed in PR 4; a judgement, not a branch |
| E14 | Eligibility: a human message in a `CHAT_ROUTE_SPACES` space, created inside the five-minute window, and nothing else (your D6, §3.3) | `test_chat_producer`: a message in an unlisted space is not staged; a message older than the window is not staged; an **unmentioned** message in a listed space **is** staged |
| E15 | `clarify`: no acknowledgement; the settle pass then sets `Waiting on requester` | `test_chat_reply`: no ack marker. `test_tracker_sync`: `.clarify-sent` yields the pending record once |
| E16 | Whole thread staged; the requester's unmentioned answer is eligible because the room is, with no marker read and no mention test (§3.3). A third party's message in the same thread is staged too, and the agent's `ignore` is what ends it (`SOUL.md:19`) | `test_chat_producer`: the requester's reply is staged and carries `prior_operations`; `test_chat_route`: the brief treats it as steering for the open request (`SOUL.md:13`) |
| E17 | No sweep, no reminder; no final card exists for a waiting item | `test_chat_producer`: nothing re-sent for a pending thread |
| E18 | Third `clarify` in a thread refused and blocked | `test_chat_reply` |

---

## 8. Non-goals

- **A second host sender.** `chat-reply` is the sender. There is no `chat-writer`, and `tracker-sync` contains no send call (§2).
- **A second LaunchAgent for the tracker.** `chat-reply` calls `tracker-sync`; the tracker step has no tick of its own (§0.2).
- **A closure producer or any tracker poll.** The final card is the trigger.
- **A chat bot identity.** Replies come from the owner's account; a requester cannot tell Ralph's text from yours. A different consent.
- **A new `gchat.py` command.** Its CLI is unchanged; the producer reuses its credential helpers and makes its own read-only `list` call (§3.4).
- **Running any of this in a container.** Not a preference — the credential cannot decrypt there, and putting one there would undo the boundary (§3.5).
- **A watermark or any backfill of messages older than the window** (§3.4). Your D4 says five minutes; nothing recovers what falls outside it.
- **Unsending**, **a retry that re-sends**, and **sweeping pending requests** (A18).
- **Any rate limit or volume cap on requesters** (your D4). The per-thread reply ceilings in §4.2 are not that, and stay.
- **Reaching "all chats"** beyond `CHAT_ROUTE_SPACES` (your D4: "we will expand to all chats later"; your D6: "keep it only to approved chat rooms"). Expanding is a change to §3.3, not a setting: the producer would have to derive the space set itself, and `chat-reply:49-50`'s cap of eight would go with it.
- **Direct messages**, and **any @mention test** (your D6: "can come later"). The producer reads no annotations in v1, and a direct message is not a listed room, so neither surface exists to be got wrong. Adding them later is one condition each in §3.3.
- **Any requester-facing surface**, and any tracker content reaching a requester. Unchanged from the PRD.
- **Amending the PRD.** Your D7 approved it, and it is card `t_73bf3c94`, gated on this one — R6, R7, A14 and R18, in their own pull request against `prd.md`. Not done here, because a design document does not get to edit the requirement it implements in the same commit.
- **Implementing any of this.** PR 1 starts after this document is merged.

---

## 9. Assumptions, and observations

### Assumptions

Every inference the PRD, the card, your answers and the verified source did not
state. Numbering is v4's; three of v3's twelve retired when you decided.

1. **Redmine's capabilities are as generally documented** (REST toggle, a per-user key, custom fields, relations). Not inspected, and must not be.
2. **An API key for a dedicated tracker account can live on the host** beside the chat token, owner-only. Your D1 approves the account; where its key sits is my inference.
3. **`spaces.messages.list` returns `clientAssignedMessageId` and `sender.type`, and accepts a `filter` on `createTime`.** Unverified; PR 2 confirms it before anything else (§3.2, §3.4). Still the assumption that would change the design rather than a detail — but **smaller after your D6**, which took the mention annotations out of it, and they were its least certain third.
4. **"The thread is the request"** for counting and for item lookup. A requester who restates the ask in a new thread is a new request that R12 then links.
5. **A requester cannot tell Ralph's text from yours**, and you accept that.
6. **One host, one scan at a time per instance.** launchd does not start a second copy of a running job, and the thread lock covers the rest. I did not verify launchd's overlap behaviour against your version.
7. **The producer reads `CHAT_ROUTE_SPACES` from the same instance file** `chat-reply` reads, so they cannot diverge.
8. **`reply_kind` and `kind` are safe to add as optional metadata**, because `chat-reply` ignores unknown keys (`:90-93`) and PR 4 widens `chat-route`'s allow-list in the same change.
9. **"Don't cap anything" means no limit on requesters**, not the removal of the per-thread reply ceilings (§0.1, §4.2). The narrower reading, stated so you can correct it.
10. **Retired by your D6**, and kept as a number so nothing renumbers. It read "for the account from the room" as *keeping* the @mention test; you answered the other way (§0.3, §3.3).
11. **A five-minute window scanned every 60 seconds loses only what arrives while the host is not scanning**, and losing it silently is acceptable (§3.4).
12. **You will run the one live thread-reply check** before the first real send.
13. **A public GitHub repository is a sufficient proxy for "the requester can already reach it".** Anything subtler is treated as unreachable and named in words.
14. **Splitting the tracker out of the sender is mine to settle**, because it is file structure rather than product or policy. The proposal arrived on your behalf and §2 takes it; if you want the shape it described instead — a separate LaunchAgent on its own tick — that is a sentence from you, not a redesign.
15. **An operation's workspace directory is a sound place for `tracker-sync`'s own sentinels.** It is already where the reply markers live (`chat-reply:27,145-147`), owner-only, and removed when the item closes (§5.3).
16. **Your D6's two sentences mean the room list is the whole eligibility gate**, not that the @mention test survives inside it. The reading comes from the second sentence: if the mention test stayed, deferring it would leave *nothing* eligible, because it was the only sender test the producer had. This is the inference most worth correcting if it is wrong, and correcting it is one sentence (§0.3).
17. **An approved room's traffic is low enough that one intake card and one model run per human message is acceptable**, given that your D4 waived any volume cap. I have no measurement of your rooms, so this is a guess about your usage rather than a property of the design (§3.3).

Retired by your answers: v3's A1 fallback clause (`rails runner`, D1), its A9
("the shipped behaviour was considered, not drift" — D2 settles it either way),
its A10 (the 900-second tick, replaced by 11 above), and now A10 above, which
your D6 answered in the opposite direction.

### Observations, out of scope

One line each. Reported, not fixed.

- `chat-route` refuses any decision metadata key outside `decision, body, ack, worker_session_id` (`chat-route:55-56`), while `chat-reply` ignores extra keys (`chat-reply:90-93`; commit `b657ee2`). A worker that adds a key such as a diagnosis note makes `chat-route` exit 2, and `chat-reply` then reports "cannot confirm orchestrator card" and returns retry (`:198`), so the acknowledgement would retry every scan and never send. Read from the code, not reproduced.
- `chat-reply` has no block, alert or log beyond stderr; a refused or failing reply is invisible (§4.8).
- If a send succeeds and the marker write then fails, `chat-reply` says "retry is safe" (`:208`). A retry reuses the same `messageId`; what the Chat service returns for a duplicate custom id I did not verify, and the retry could fail forever on it.
- No chat eval runs in CI (§4.7), and `evals/test_inbox_writer.py` is in no workflow either.
- `ralph-chat-maintainer` has a repository token (`docs/decisions/0001`) and reads untrusted chat text; the ADR names this as accepted risk. R19 now rests on host checks because of it.
- The staged-evidence safety check exists twice already: `chat-reply:69-74` raises `Refused`, `chat-route:42-47` raises `ValueError`, and they test the workspace symlink in different places (`chat-reply:69`, `chat-route:38`). That is why PR 6 extracts one reader rather than adding a third copy.
- `install-chat-reply-launchd` refuses to replace a plist whose settings differ (`:44-46`) and offers no `--force`, so any later change to the scan command is a manual unload and reinstall on your machine.
