---
id: dd-ralph-redmine
title: Build the work tracker onto the chat path Ralph already shipped — one producer to feed it, one tracker script the sender calls but cannot import, and sender-side ceilings on every reply
status: in review
owner: Zhachory1
card: t_db313c63
revision: v7
date: 2026-11-06
implements: prd-ralph-redmine
prs:
  - Zhachory1/docs#7   # v1, v2, v3, v4, v5, v6, v7
supersedes: none
---

# Build the work tracker onto the chat path Ralph already shipped — one producer to feed it, one tracker script the sender calls but cannot import, and sender-side ceilings on every reply

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker" (a self-hosted Redmine), and Google Chat is a vendor name and is
> named. The PRD's banner applies unchanged.

Status: **in review** — a design, awaiting your approval. Nothing is built, no
chat message was sent or read, no REST API was enabled, no credential and no
work instance was touched. Merging this pull request is the approval.

Implements `prd-ralph-redmine` v5, merged at `Zhachory1/docs#6`. **You answered v3's
four decisions on 2026-11-06, v5's remaining two the same day, and D5 the same
day again; §0.1, §0.3 and §0.4 record what each one moved.** One decision is
left, D8, and it is a consent rather than a shape: a send-capable Chat
credential has to exist inside a container for the answer you gave on D5 to run
at all. **Your D5 answer moves the sender off your machine** — `chat-reply` and
the producer become a compose service beside the agent, not a second LaunchAgent
(§3.5) — and **your D7 answer sends the PRD reconciliation to its own card**,
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
`gchat.py:222-240` to rewrite eligibility; all three hold as cited. **v7 is the
largest re-read since v3**: `compose/docker-compose.yml` and
`compose/Dockerfile.notifier` in full, `bootstrap/run-notifier`,
`bootstrap/run-review-loop`, `bootstrap/install:133-160`, `gchat.py` in full,
`bootstrap/chat-route` in full, and `bootstrap/inbox-producer:1-36`, because
§3.5 and §6 now rest on them rather than on a machine-id argument. **Two of the
sidecar request's load-bearing claims did not survive the re-read, and one of
its mount lists is short by a directory** — §0.4 says which. I also pulled
`main` at `64152e6` (2026-11-06) to check for drift since
`95bd4d9`: five commits, touching `bootstrap/cleanup-sandboxes`,
`evals/test_cleanup_sandboxes.py`, `evals/test_profile_config.py` and the public
standards — **nothing this document cites**, so every line number below is still
exact on `main` today. I read the code; I did not run it, send a message, or
call any API.
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
| **D1 — Approved** | The tracker's REST API goes on, Ralph gets its own tracker account and a per-user API key. | §5.1's `rails runner` fallback is **dropped**, with it the second trust boundary and the "skips validation" cost. PRs 1, 7 and 8 lose their gate; the only prerequisite left is the key file, which after your D5 lives in the tracker service's own volume and is one line of PR 9's runbook (§3.5). |
| **D2 — Let's go with the recommendation** | Where shipped `chat-reply` and the PRD disagree, **the shipped behaviour wins and the safety moves from the sentence to the sender.** Ralph keeps writing its own acknowledgement and refusal, and tells a requester when work failed. | §4.4's refusal checks are now the **whole** content bound, not one of two options; the conditional branches in §4.4 and §4.6 are gone. No `SOUL.md` line changes, so PR 4 shrinks. And the merged PRD now has three sentences (R6, R7, A14) that describe nothing that runs — **your D7, answered in §0.3, sends them to card `t_73bf3c94`**. |
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
meant those too, say so and §4.2 goes with them — but then R15 has no sender-side
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

**Your D5 repays most of that subtraction, which v5 could not.** In a compose
stack the two scripts can run in different containers with different credential
volumes, and the sender can still invoke the tracker step with no rendezvous —
§2 says how, and §3.5 lists the volumes. On the by-hand fallback the subtraction
stands exactly as written above.

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

## 0.4 What you decided on D5, and the two requests that arrived with it

Two comments on `Zhachory1/docs#7`, thirty-three minutes apart, both from
`zhach1`. Quoted exactly, because the second one reverses the first.

| You said | It settles | What moved in this document |
|---|---|---|
| **D5 — "A second host launch agent is fine. 👍"** (16:19:44 UTC), then **"Take the sidecar please."** (16:52:44 UTC) | The **later** answer controls: the producer and the sender run as a compose service beside the agent, not as a second LaunchAgent on your machine. The fallback is unchanged — `chat-producer --scan` by hand. | §3.5 is rewritten from the ground up: the machine-id argument stops being the headline and becomes a cost (§3.5.1). §6's PR 3 loses the LaunchAgent installer, and a new **PR 10** carries the image, the services and the credential step. §2 gains the sidecar's effect on Option C. D5 leaves the decisions table and **D8 takes its place** — not the same question, and §3.5.1 says why it is new. |

**One word changes meaning, so it is defined here.** This document said "the
host" for two different things: *your machine*, and *the process that holds the
chat credential and sends*. After D5 those are no longer the same place. From
here the second one is **the sender**, and the property it names is unchanged:
the destination comes from staged evidence and the ceilings from marker counts,
never from agent text. Where a sentence is about your machine specifically, it
still says so. Where the sentence is history — what v2 proposed, what shipped —
it still says "host", because that is what shipped.

**The sidecar request, re-verified.** Its direction is now your decision, so
what matters is which of its premises hold at `95bd4d9`. Most do. Three do not,
and two of them change the work.

| The request said | At `95bd4d9` | So v7 |
|---|---|---|
| "`notifier` and `review-loop` … both share `/workspaces` at the same path" | **Only `review-loop` does** (`compose/docker-compose.yml:270`). `notifier` mounts `../notifier`, `../bootstrap` and its own state volume, and no workspaces at all (`:227-230`) | names the mount explicitly in §6's PR 10. It is not inherited by copying the notifier service, and the producer writes nothing the agent can read without it (`bootstrap/install:140-149`) |
| "a sidecar mints its own [credential]" — so the machine binding "stops being the objection" | **A sidecar cannot mint one at this commit.** `cmd_auth` calls `flow.run_local_server(port=0)` (`gchat.py:163`), which needs a browser on the same machine and a port nobody can publish in advance. The request lists this as its own cost 1, which contradicts its headline | takes the minting problem as **the** open question, not a footnote: §3.5.1 picks a path, and **D8** asks you to approve it |
| the sidecar "runs only read-only-mounted repo scripts (`../bootstrap:/ralph/bootstrap:ro`)" | Right, and **one directory short**. `post()` resolves the sender two levels up from its own file — `<repo>/skills/ops/google-chat/gchat.py` (`chat-reply:128`) — so `../skills:/ralph/skills:ro` is required too, as `init` already mounts it (`:88`) | lists both mounts in PR 10, and §9's observations record the path-resolution coupling |

Three smaller corrections, none of which change anything: `instance_config`
begins at `chat-reply:41`, not `:40`; of the three `docker exec` call sites the
request cites as `hermes kanban`, one is `chat-route` (`chat-reply:194-196`);
and `run-notifier`'s header says "a crash is visible as a restarting container
instead of a silently missing job" (`:3-4`) rather than the `run-review-loop`
wording it was quoted as (`run-review-loop:2-3`). The request's two best
arguments both survive intact: Ralph already runs host-shaped loops as compose
services with `restart: unless-stopped`, and a LaunchAgent that is unloaded is
the silent failure §3.4 accepts.

**The second request is a proposal, and this document treats it as one.** A
send switch in `chat-reply`, so intake, classification and the tracker can run
for a week while no requester is replied to. It came from the Ralph stack's own
account at the owner's request and not from `zhach1`, and it asked to be costed
rather than adopted. **It is costed in §4.9 and adopted, with one change**, and
the change is not cosmetic: held replies need a marker of their own, because
without one, flipping the switch on would post a week of backdated
acknowledgements into real threads in a single 60-second pass (`chat-reply:239-242`).
§4.9 shows the mechanism and prices all four of the open problems it listed.

---

## Review header

### Decisions needed from you: one

Numbers never repeat across revisions. v3's **D1-D4 are answered** (§0.1), v5's
**D6 and D7 are answered** (§0.3), and **D5 is answered** (§0.4). What is open is
**D8**, and it exists *because* you answered D5 the way you did: the shape you
chose cannot run without a send-capable Chat credential inside a container, and
that is a consent, not a file layout. It gates **PR 10 only**; every other pull
request in §6 is inert on merge either way, and the fallback is the one you
already have — run `chat-producer --scan` by hand.

| # | Decision | Why it is yours | If you say no |
|---|---|---|---|
| **D8** | **Put a send-capable Chat credential in a container**, in a volume mounted only into the sender service. The recommended path copies the grant you already have rather than asking Google for a second one: decrypt the host token once, re-encrypt it under a pinned machine id and user, and write it into the sender's own volume (§3.5.1, option **b**). Two things follow and both are yours: that copy is a long-lived send credential with **no machine binding left** (`gchat.py:106-114` keys on a file you now supply), and it can post as you until you revoke it in your Google account. The client secret never enters the container, and your own host token does not move. | Where a credential that can post as you is allowed to live is not a design question. The repository's own boundary — "holding no chat credential is enforced by where the credential lives" (`profiles/chat-maintainer/config.yaml:3-5`) — was written by you, and this is the first thing that spends it. | The sidecar is off: no container may hold the token, so `chat-reply` stays on your machine and the producer goes back to the LaunchAgent you approved at 16:19, or to `--scan` by hand. Nothing else in the design changes — §3.5.1's cost table is the whole of the difference. |

**Three things that are yours and are not decisions.** Run one threaded reply in
a space of your own and confirm it lands in the thread, before the first real
send: `test_gchat_send.py` proves the code asks for `REPLY_MESSAGE_OR_FAIL`; it
cannot prove what the Chat service does with an old thread id. I cannot tell
from the repository whether you have done this. `SKILL.md:46` says "after a
controlled test". Choose the pilot rooms knowing what D6 now means — Ralph reads
every human message in them (§0.3). And **flip the send switch when you are
ready for requesters to hear from Ralph** (§4.9): it ships off, so intake and the
tracker run first and you decide when replies start.

### Council dissent

v2's independent review found three blocking issues, all accepted and fixed:
`inbox-writer` blocks nothing; the lock was on the operation rather than the
request; and thread-scoped eligibility was settled in prose rather than
escalated. **v3 through v6 keep all three fixes** — the block precedent is cited at
`maintain-pr` (§4.8), the lock is per thread (§4.2), and the eligibility
widening went to you as D4(a), which you approved. v3 reached you without an
independent review; **v4 was reviewed** and raised one blocking finding — two
new cross-references pointed at a decision number that no longer exists (D3
rather than D7) — fixed in that revision. **v5 was reviewed too**: one blocking
finding — `--record` would deadlock the acknowledgement path if its own marker
were written `O_EXCL` like every other marker in the design — and three notes,
all accepted and fixed. **v6 was reviewed and had no blocking findings**; the
reviewer's one substantive act was to flag the D5 conflict in §0.4 rather than
pick an answer, which is why you answered it twice and the second answer stands.

**Where v7 disagrees with what it was asked to write.** Both requests on the
pull request were drafted by an assistant rather than by you, and both are taken.
Neither is taken as written.

1. **The sidecar request's headline is wrong, and its footnote is the real
   problem.** It says the machine binding "stops being the objection" because "a
   sidecar mints its own" credential. A sidecar cannot mint one (`gchat.py:163`),
   so the binding does not stop being an objection — it stops being a *security*
   property and becomes a cost you are now asked to accept on purpose (D8). The
   request's own cost 1 says the same thing, two paragraphs later.
2. **The sidecar is not a credential boundary either, for the same reason §0.2
   gave about `tracker-sync`.** A container that holds the chat token and can
   `docker exec` into any container on the proxy (`compose/docker-compose.yml:31`)
   is not isolated from the board; it is isolated from the *model*. That is the
   boundary worth having and it is the one §3.5 now claims. Claiming more would
   be claiming something `EXEC: 1` disproves.
3. **The send switch needed a change to be safe, not just to be tidy.** Held
   with no marker, every held operation is rescanned every pass
   (`chat-reply:239-242`), so the day you flip the switch on, the scan sends the
   whole week at once. The proposal's open problem 3 noticed the rescan and
   priced it as a re-run cost; it is a send-flood risk. §4.9 fixes it with a
   held marker and makes the flip inert for everything already held.

The older dissents still stand. **The split proposal** (§0.2): its concern is
taken and its mechanism is not. **D4** (§0.1): "don't cap anything" is
implemented as no rate limit on requesters, and the per-thread reply ceilings
stay. **D6** (§0.3): I have no objection to the decision and one to its framing —
"keep it only to approved chat rooms" sounds like a restriction and is a
widening, because inside those rooms it removes the only sender test there was.

### Scope: medium — ten pull requests

Nine in v6. PR 3 sheds the LaunchAgent installer and a new **PR 10** picks up
the sidecar: the image change, the compose services, the environment-variable
configuration, and the one-time credential step. PR 5 grows by the send switch
(§4.9) and PR 7 by two marker conditions. Breakdown in §6.

### What changed since v6

Your D5, the two requests it arrived with, and the re-read they forced. No
decision below D5 was reopened and the PRD is untouched (`t_73bf3c94` still owns
it).

- **§0.4 is new**: your two D5 answers, why the later one controls, the sidecar request re-verified against `95bd4d9` — three of its premises do not hold — and what v7 does with the send-switch proposal.
- **§3.5 is rewritten and gains §3.5.1**: the question is no longer "why not in a container" but "which container, holding what". The machine-id argument is demoted from the answer to a cost, and the cost is priced.
- **§4.9 is new**: the send switch, its held markers, and all four of the proposal's open problems costed. Intake and the tracker go live first; replies wait for you.
- **§2's Option C is amended**: the sender calls `tracker-sync` through `docker exec`, the same way it already calls `chat-route`, so the two credentials sit in different containers without a rendezvous.
- **§6 has ten rows**: PR 3 is gate-free, PR 10 is new and carries D8, PR 9's runbook gains the credential step and the switch.
- **Decisions: D8 only.** D5 is closed by you. D8 is the consent D5's answer requires.


---

## 1. The answer, in one page

Ralph already runs half of this: *an intake card, an agent that only decides, a
sender that posts what it decided*. The tracker needs three additions and
nothing else. After your D5 the sender and the producer move into a container
beside the agent rather than onto your machine (§3.5); nothing about what they
may send changes with them.

```
chat ─> [NEW] bootstrap/chat-producer ─> chat-inbox/<op>/evidence/item.json + intake card (queues)
        (sender sidecar, chat token)                 │
                                                     ▼
                                            ralph-chat-maintainer   (decides: route | reply | ignore)
                                                     │        (hermes container, no chat credential)
                                                     ▼
        bootstrap/chat-reply  (same sidecar) ── [NEW] ceilings ── send ──> thread
          chat credential, no tracker key   │                 └─ [NEW] send switch (§4.9)
                │                           └─ docker exec ─> [NEW] bootstrap/tracker-sync
                │                                             (own service, tracker key, no send path)
                ├─ ack:   chat-route makes the orchestrator card, THEN `tracker-sync --record`, THEN the reply
                ├─ clarify: the question goes out, THEN the pending record on the next `--settle`
                └─ final: the card the orchestrator ends the graph with; THEN `--settle` closes the item
```

1. **The agent still cannot send to chat.** It holds no chat credential: that stays out of the container the model runs in (`config.yaml:3-5`). It *can* now read repositories with a token (§0), which is why §4.4 changes.
2. **The sender cannot improvise.** Destination comes from staged evidence, never agent text (`chat-reply:64-84`). That stays. New: the sender also decides *whether* a reply may go, from counts it reads itself (§4.2) and from one switch you control (§4.9).
3. **The record precedes the acknowledgement.** `chat-reply` already holds the ack until `chat-route` confirms an orchestrator card (`:193-200`). `tracker-sync --record` joins that same step, so "Ralph says it has the work" and "the work is recorded" cannot come apart (R9). Nothing else waits on the tracker (§0.2).
4. **No new datastore.** Sent-once state is the marker files that exist. The thread-to-item map is two fields on the tracker item. Nothing else is added (R14).

---

## 2. Options considered

Three ways to add the tracker. Standards require at least two.

| Option | For | Against |
|---|---|---|
| A. Put the tracker step inside `chat-reply` (v4's choice) | Fewest files. One place decides whether a reply goes and whether it was recorded. No new contract between processes. | `chat-reply` would carry ack, reply, final, ceilings, refusals **and** the tracker. Its one-sentence description grows an "and", which is the standards' test for a file that is too big. The tracker key would be read into the process that sends. |
| B. Write the v2 `chat-writer` beside it | Cleaner single responsibility; v2's eval design applies as written. | Two host senders with one policy line. Two places to forget a ceiling. Two LaunchAgents. It re-implements what shipped three days after v2 was written. |
| **C. One sender, one tracker script, called by the sender (chosen)** | `chat-reply` stays "post the maintainer's replies to the original thread" and gains the ceilings and the refusals, which are *about replies*. `bootstrap/tracker-sync` holds the tracker key, imports nothing from the sender and contains no send call, so one grep over the two scripts still proves there is exactly one way to send (§4.7). It is the shape `chat-reply` already uses for `chat-route` (`:194-199`), so the gating, the retry and the failure path are the ones in the file today. | A second file, a second eval, and one more exit-code contract to get right. `chat-reply` still *drives* the tracker step: the ordering lives in the sender even though the tracker code does not. Two scripts must agree on the workspace layout. |

**Why C over A.** Both are safe; C is the one that keeps the file describable.
The decisive argument is not credential isolation — §0.2 says plainly that on
one host the split is a code boundary and not a process boundary — it is that
§4.7's bound is checked by reading one file, and that file should not also be
where tracker behaviour lives.

**What your D5 adds to C, and it is the thing you asked for.** On your machine
both scripts run as you, so "neither holds both credentials" was true of the
code and not of the process (§0.2). In a compose stack it can be true of both,
because `chat-reply` does not have to *import* or even *contain* `tracker-sync`
to call it: it calls it exactly as it calls `chat-route` today, through
`docker exec` into another container (`chat-reply:194-196`). So `tracker-sync`
becomes its own small service with the tracker key and no chat token, the sender
keeps the chat token and no tracker key, and the call is still one invocation,
inside the thread lock, gated on an exit code, with no rendezvous and no second
tick. §6's PR 8 changes one call site from a local subprocess to a
`docker exec`; nothing else in §5.2 moves.

**Why C is not a rendezvous.** The two scripts do not meet in a shared tick or a
shared file. `chat-reply` calls `tracker-sync --record <op>` before the
acknowledgement and `tracker-sync --settle` once per scan; `tracker-sync` never
reads the board for a decision and never decides whether a reply may go (§5.2).
That keeps one reader of the decision metadata, one writer of the reply
markers, and one lock.

If you would rather have the split exactly as proposed — its own tick, its own
schedule — say so. In the sidecar shape that is a third compose service with a
loop interval of its own (`run-notifier:10` is the pattern) rather than a
replacement plist, so it is cheaper than v5 priced it; it still costs one tick
of acknowledgement latency and a shared lock that neither script has today.

---

## 3. The producer: the missing half

The producer is **the same job v2 described**, now with a contract it must meet
rather than one it could choose. It runs in the sender service as a loop (§3.5),
calls `spaces.messages.list` over a five-minute window (§3.4), and for each
eligible message stages the evidence and creates one intake card on the `queues`
board. `bootstrap/inbox-producer` is the shape to copy: the same read beside the
token, the same `stage_once` evidence at mode `0600` under a `0700` directory,
the same `hermes kanban … create --idempotency-key` through `docker exec`
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

**How often.** Every 60 seconds, the same interval as the reply scan — a
`StartInterval` on the fallback (`install-chat-reply-launchd:19`), a loop sleep
in the service (`run-notifier:10` is the pattern) — so each message falls inside
roughly five consecutive windows. The repeats cost nothing: the operation id is the sha256 of
the message's resource name (`chat-reply:82`), and the card carries it as the
idempotency key, so a message already staged produces no second directory and no
second card — the same no-op the mail producer relies on
(`inbox-producer:19-22,152`). v3's 900-second tick is retired.

**What it loses, stated plainly.** A message older than five minutes when the
scan runs is never seen, and nothing notices. If the machine sleeps, the service
is down or never started, the LaunchAgent is unloaded, or the network is out for
six minutes, the requests sent in that gap are lost silently — to Ralph; the requester sees no reply and
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

### 3.5 Where the sender runs, and what holds its credential (your D5)

v4 and v6 answered a question you have now closed: *why can this not run in the
container?* The honest answer was never "because it is a container". It was
"because the only container on offer was the one the model runs in". You chose a
different container, so the argument has to be rebuilt for that shape. It
rebuilds, and the thing it rests on changes from cryptography to placement.

**What the boundary actually protects.** `profiles/chat-maintainer/config.yaml:3-5`
says "holding no chat credential is enforced by where the credential lives, not
by this file or the SOUL". The thing on the other side of that sentence is a
**model** that reads untrusted chat text as its whole job, with a terminal and a
repository token (`config.yaml:6-16`). The sender-side ceilings (§4.2) and the
content refusals (§4.4) are a bound only if there is no second way to send, and
a token reachable by that model is the second way. Nothing in that argument is
about Linux, Docker, or where a process happens to be scheduled.

**So the sidecar keeps it, by placement.** Two new services, and what each one
may touch:

| Service | Holds | Mounts | Why it cannot reach the other's credential |
|---|---|---|---|
| `chat-sender` (producer loop + `chat-reply --scan` loop) | the Chat token, in its own named volume | `chat-creds` (rw), `../bootstrap:ro`, `../skills:ro`, `${RALPH_WORKSPACES}:/workspaces` | The tracker key is in a volume it does not mount. It runs no model and no agent; its only executables are two read-only repository scripts. |
| `tracker-sync` (invoked, not ticked) | the tracker API key, in its own named volume | `tracker-creds` (ro), `../bootstrap:ro`, `${RALPH_WORKSPACES}:/workspaces` | The Chat token is in a volume it does not mount, and the file it runs contains no send call (§4.7). |
| `hermes` (unchanged) | the repository token | as today (`compose/docker-compose.yml:158-183`) | Neither credential volume is mounted into it, and neither is `hermes-home` — the one volume `init`, `hermes` and `hermes-webui` all share (`:85`, `:159`, `:336`, `:338`). |

**Every mount in that table is forced by a line of code, not chosen.**
`../skills:ro` because `post()` resolves the sender two levels up from its own
file and shells out to `<repo>/skills/ops/google-chat/gchat.py`
(`chat-reply:128`). `/workspaces` at the same host path as the hermes service
because `bootstrap/install` finds the chat-maintainer's read-only evidence mount
by inspecting *that* container's `/workspaces` source (`:140-149`, `:151-160`),
so a producer writing anywhere else stages evidence the agent cannot open.
`chat-creds` writable rather than read-only because a refresh rewrites the token
in place (`gchat.py:139-143`, reached from `:150-152`).

**The exec reach, stated rather than waved at.** `socket-proxy` runs with
`EXEC: 1` (`compose/docker-compose.yml:31`), so any service on the proxy can
`docker exec` into any container it can name. The sender needs exactly that —
it is how `chat-reply` reads a card and runs `chat-route` today
(`chat-reply:157,194,222`). This is **not a new class of risk**: `notifier`
already holds a chat webhook URL and the same proxy access (`:221-223`), and
`review-loop` holds a repository token that can push (`:256`). What is new is
one combination — a credential that can post *as you* in a container that can
exec — and the answer is the mount list above: the sender runs two read-only
scripts, holds no model, and shares its credential volume with nothing.

**"In a container" does not mean outside Ralph, and "on your machine" never
did.** `chat-reply` already reaches into the hermes container for everything
board-side (`chat-reply:157,194,222`), and the mail producer does the same
(`inbox-producer:29-33`) for the same reason, in your own earlier design: "Runs
on the host, where the Gmail token lives; the token never enters the hermes
container" (`inbox-producer:4-5`). The sidecar keeps the second half of that
sentence and drops the first. Ralph already runs two host-shaped loops this way,
and both say in their own headers why: "a crash is visible as a restarting
container instead of a silently missing job" (`run-notifier:3-4`;
`run-review-loop:2-3`). That is the same failure §3.4 accepts from an unloaded
LaunchAgent, and `restart: unless-stopped` narrows it.

**Can the agent's sandboxes reach the sidecar? Unverified, and nothing depends
on it.** The request asked me to check rather than assert, so: I did not run
Docker, and I will not assert it either way. What the repository shows is that
the question is not load-bearing. No agent sandbox needs to talk to the sender:
evidence reaches the chat-maintainer as a **read-only bind mount** of
`chat-inbox` into its sandbox (`bootstrap/install:140-149`), and decisions come
back on the board, which the sender reads by `docker exec`. There is no network
call between them to secure. What is also true: nothing in this repository sets
a network for a sandbox — the only sandbox knobs compose sets are the image and
a forwarded-environment list (`:133`, `:136`) — the proxy refuses the network
API outright (`NETWORKS: 0`, `:36`), and the isolation spike probes mounts, the
socket and the board, not reachability
(`docs/spikes/sandbox-isolation.md:29-41`). If PR 10 ever needs the answer, it
is one command on your machine (`docker inspect` a live sandbox's
`NetworkSettings`), and §9 carries it as an assumption until then.

**The image: extend `Dockerfile.notifier`, do not add a second one.** `gchat.py`
imports `cryptography`, `google_auth_oauthlib`, `googleapiclient` and
`google.auth` (`:46-52`); the notifier image is `python:3.12-slim` plus
`docker-cli`, `git` and `gh`, and installs no Python package at all
(`compose/Dockerfile.notifier:6-16`). The PEP 723 header in `gchat.py:1-9` does
not help, because `post()` runs it with `sys.executable` rather than `uv run`
(`chat-reply:131`), so the four packages must be importable by the image's own
Python. One `pip install` line in the shared image is the smallest change that
works. The notifier image's stated reason for being small is that the Hermes
image "costs 4.5GB to carry a CLI" (`Dockerfile.notifier:3`); four declared
packages and their transitive dependencies are tens of megabytes against that,
which does not threaten the reason — but **I did not measure it**, and checking
the image-size delta before merge is a PR 10 step rather than a claim here. A
second image would also mean a second build target and a second tag for three
services to choose between, which is an abstraction for one consumer (rule of
three).

**Configuration: environment variables, with the file kept.** `instance_config`
reads `RALPH_INSTANCE`, `RALPH_WORKSPACES` and `CHAT_ROUTE_SPACES` out of a host
env file and resolves the workspaces path relative to that file
(`chat-reply:41-53`). In a service those are environment variables, so PR 10
teaches both scripts to take them from the environment when `--instance-env` is
absent, with the same validation — the instance-name pattern, the non-empty
space set, the cap of eight, the `spaces/<id>` shape (`:49-51`). The file path
stays, because it is what the LaunchAgent fallback uses if you refuse D8. One
small thing improves: inside the service `RALPH_WORKSPACES` is `/workspaces`, so
the root the sender derives and the `workspace_path` the card carries are the
same string for the first time (`chat-reply:171`).

**One Chat token, not two, and the producer sits beside the sender on purpose.**
`gchat.py` mints one token for one scope list that already includes both
`chat.messages.readonly` and `chat.messages.create` (`:60-66`), and
`_load_creds` reads exactly one token path (`:129-136`). A read-only credential
for the producer would need a second scope list, a second consent screen and a
change to a script `gws-cli` shares, which buys little: the producer only
*copies* message text into a file, and the thing the boundary protects against is
a model holding a send path, not text passing through a container.

#### 3.5.1 The credential, which is the whole of D8

This is the part the sidecar request priced as cost 1 and then contradicted in
its headline. **A sidecar cannot mint its own token.** `cmd_auth` runs
`InstalledAppFlow.run_local_server(port=0)` (`gchat.py:161-166`), which needs a
browser on the same machine and a loopback port chosen at random — nothing the
host can publish in advance. So the token has to arrive from outside, and how it
arrives is a consent question.

| Path | What it costs | Verdict |
|---|---|---|
| **a. Mint inside the container**: pin a machine-id file in the volume, publish a temporary port, run `gchat.py auth` through it once | Needs `port=0` to become a fixed port — a change to a script `gws-cli` shares (`gchat.py:59`, `:163`) — plus a published port on a service that should publish none, and a browser redirect that reaches it. Produces a genuinely **second** OAuth grant, which is a second consent screen and a second thing to revoke. | Rejected. The change lands in the wrong file and the port is a new surface. |
| **b. Mint on your machine as today, re-encrypt for the sidecar (chosen)** | A one-time `bootstrap/chat-credential` that reads the host token with the host key and writes it back under HKDF of `"<pinned machine id>:<pinned user>:gws-cli"` (`gchat.py:106-114`), into the sender's volume. No `gchat.py` change: `_load_creds` and `_creds_or_die` need only the token and the salt, because the client id and secret are inside the authorized-user JSON (`:129-136`, `:146-154`). **The encrypted client secret never enters the container.** The cost is the honest one: the machine binding for that copy is gone, because the machine id is now a file you supply. | **Chosen.** It is the only path that keeps `gchat.py` unchanged and the client secret on your machine. |
| **c. A Docker secret holding the refresh token** | `gchat.py` reads one encrypted path and nothing else (`:57`, `:129-136`), so this needs a code change to read a secret, and compose secrets are a file mount with extra steps. | Rejected: it is option b with more moving parts. |

**What b gives up, in plain words, because it is what D8 asks you to approve.**
Today `chat_token.json.enc` is useless if copied off your Mac: the key is derived
from `IOPlatformUUID`, your unix user and the shared salt (`gchat.py:72-114`), so
the file alone decrypts nowhere. After b, a copy of that token exists whose key
inputs are a pinned machine-id file, a fixed container user and the same salt —
all three of which travel with the volume. Anyone who can read that volume can
post to Chat as you until you revoke the grant in your Google account. The volume
is mounted into one service and is not `hermes-home`; that is the whole of the
protection, and it is a placement guarantee rather than a cryptographic one. The
reason to accept it is that the alternative — the LaunchAgent you approved at
16:19 — is the failure mode `run-notifier:3-4` was written to avoid.

**How you do it, once, and where it is written down.** PR 10 adds the script;
PR 9's runbook adds the three lines that use it, between creating the tracker
account and starting the services. Nothing in either PR runs until you have run
them, and `chat-reply` with no readable token simply fails its send and retries
(`gchat.py:154`; `chat-reply:200-202`), which is the same behaviour as a
revoked token today.

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

### 4.2 The ceilings, read by the sender from the thread

The **request is the thread**. `chat-reply` takes an exclusive `flock` on a lock
file named for the thread, before it counts and until it has marked, and then
counts markers across this operation's directory and every `prior_operations`
directory in `item.json` (staged by the sender, never agent text):

| Rule | Ceiling | Refusal |
|---|---|---|
| A17 | at most 2 `clarify` per thread (one question, one follow-up) | the reply is refused, the card is blocked (§4.8), the request stays pending |
| A15 | at most 1 `decline` per thread, and a decline creates **no** tracker item (R7) | refused and blocked |
| R6 | at most 1 `ack` per thread, and only after the tracker item exists | held, retried next scan |
| R11 | at most 1 `final` per thread, and only after `ack` (shipped, `:183-184`) | refused |

The lock is on the thread and not the operation for the reason v2's reviewer
found: several messages in one thread are separate operations, and
operation-scoped locks never see each other. **It counts only sent markers.** A
reply held by the send switch writes a marker of its own (§4.9) and is not a
send, so holding a week of replies cannot consume a ceiling, and flipping the
switch on cannot find a thread that has silently spent one. A second sender is
out of scope: one service, one replica, and `restart: unless-stopped` does not
start a second copy of a running container.

**These are not the cap your D4 waived.** D4 removed any limit on *requesters* —
how many people may ask, how often, how much work they create. The table above
limits *Ralph*: how many times one request may be messaged about. Four rules,
all of them from the PRD (R6, R7/A15, R11, A17), and together they are the
sender-side half of the R15 grant.

### 4.3 R8, what is now true and what is not

**Done:** the send asks for `REPLY_MESSAGE_OR_FAIL`; a strict send without a thread, or with a thread in another space, fails before any API call (`gchat.py:251-252,258-259`; `test_gchat_send.py:34-50`).

**Not done, and not verifiable from here:** that the service honours it for a thread id captured earlier, and how old an id may be. v2's third check — compare the returned thread to the staged one — is **not available**: `cmd_send` returns only `name` and `createTime` (`gchat.py:266-267`), and `chat-reply:post` accepts `sent is True` alone. PR 5 adds a cheap partial check: the returned message `name` must begin with the staged space. It cannot detect a wrong thread in the right space. The pre-launch check under the decisions table is what closes that gap, and the token holds no delete scope (`gchat.py:60-66`), so a wrong-thread message can be reported and not withdrawn.

### 4.4 R19: the content bound, rewritten

v2 argued that the agent *could not reach* anything outside one request's evidence directory. That is no longer so. The chat-maintainer has `terminal` and a repository token (`config.yaml:6-16`); on the personal instance that token is the owner's (`docs/decisions/0002`), and both decisions record the exposure as accepted. A reply can now quote a private repository the requester cannot open.

**So the bound moves to the sender, and it is a refusal, not a trim.** Before any send of agent text — `ack`, every `reply_kind`, `final` — `chat-reply` refuses text that contains:

- a tracker reference: the instance host, an `/issues/<n>` path, or a bare `#<n>` form (R1);
- a URL not in a `reachable` list the sender builds, where a URL qualifies only if it is provably public (for a GitHub pull request, one API read showing the repository is public; everything else is named in words, R19);
- a credential-shaped string (the SOUL already says never; `SOUL.md:29`).

**The honest limit, stated plainly:** prose cannot be proven to come from this request. A refusal rule catches references and links; it cannot catch a sentence that *paraphrases* a private document. Your D2 answer makes these checks the whole content bound — there is no constant-text fallback behind them — so that residual risk is accepted rather than designed away. It is the same class of risk the ADRs already accepted for repository access, and it is smallest on `ack` (a restatement of what the requester just wrote) and largest on `final` (a summary of work Ralph did elsewhere).

### 4.5 The `SKILL.md:44` policy line

v2's request for an owner-approved edit is gone: the line is already edited and names `chat-reply`. It says "acknowledgement, reply and final response" and does not mention the tracker, ceilings or `reply_kind`. PR 5 amends that sentence to match what the script then does — the ceilings, the refusals, `reply_kind`, and that the whole of it is behind one switch you control (§4.9) — in the same pull request as the code, so the policy and the behaviour change together.

### 4.6 The close: the final card is the trigger

The orchestrator ends a chat graph with one `ralph-chat-maintainer` card whose parents are the work cards (`orchestrator/SOUL.md:19`). Kanban dependencies mean it does not run until every parent is done, so **one work card finishing is not the request finishing** (R11, E7) by construction, and no tracker poll is needed. `chat-reply` sends the `final` and marks it — or holds it and marks it held (§4.9) — and the next `tracker-sync --settle` sets the item to `Resolved` with the outcome and writes its own `.tracker-closed` sentinel, so a tracker outage delays the close and never re-sends the reply. The tracker still reads as canonical (R13) because every stage is written: created at the ack, `Waiting on requester` after a `clarify`, `Resolved` after the final.

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
| exactly one `messages ( ) . create` path exists in `bootstrap`, and `REPLY_MESSAGE_OR_FAIL` is the only reply option used | R15, R8 |
| with the send switch off, a `route` decision runs `chat-route` and `--record`, writes `.held-ack`, calls `post` not at all, and the next scan skips it | §4.9 |
| with the send switch off, a body that fails a §4.4 check is still refused and still blocks | §4.9, R19 |
| turning the switch on sends nothing for an operation that already carries a held marker | §4.9, R11 |

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

### 4.9 The send switch: the tracker goes live first

A proposal, not a decision, arrived on the pull request from the Ralph stack's
own account at your request (§0.4). **It is taken, with one change.** The
backlog is the half of this feature you will see value from, and it does not
need to wait for the thread-reply check, the `reachable` list or the R19
refusals.

**What it is.** One environment variable, `RALPH_CHAT_SEND`, read once per scan
by `chat-reply` and defaulting to **off**. With it off, `chat-reply` reads the
decision, validates the destination, runs every §4.4 content check, runs
`chat-route`, calls `tracker-sync --record` and `--settle` — and does not call
`post()` (`chat-reply:123`, called at `:200`).

**Why merging it off is not a behaviour change.** There is nothing for
`chat-reply` to send today: no producer exists, nothing in the repository writes
`chat-inbox/<op>/evidence/item.json`, and `SKILL.md:46` says the host scan "does
not start inbound Chat polling". So a default of off switches off a path with no
traffic on it, which is what lets PR 5 keep its place at the front of the order.

**The one change, and it is a safety fix rather than a tidy-up.** The proposal
holds a reply by writing *no* marker. That cannot ship. `scan()` skips an
operation only when its sent marker exists (`chat-reply:239-242`), so a held
operation is re-handled on every 60-second pass — which the proposal noticed and
priced as a re-run cost. The real cost is on the far side: **the day you flip the
switch on, one pass sends the whole pilot week at once**, an acknowledgement and
a final into every thread Ralph has seen, backdated and unasked for. The
ceilings do not stop it, because each thread has spent none of them.

So a held reply writes `.held-<kind>` in place of the marker `mark()` would have
written — `.receipt-sent`, `.clarify-sent`, `.decline-sent`, `.reply-sent` or
`.final-sent` (`chat-reply:27`, §4.1) — by the same `O_EXCL` open
(`chat-reply:145-147`), and both `scan()` and `handle()` treat either marker as
"nothing to do here". Three things follow:

- the loop stays bounded — one `docker exec` per operation per kind, ever, not one per minute forever;
- the ceilings stay exact, because they count sent markers only (§4.2);
- **flipping the switch on is inert for everything already held.** It changes what happens to the *next* request, not to the last two hundred. For a request whose acknowledgement was held, that is also what R11 requires: "A request that was never acknowledged never gets a closing reply either" (`prd.md:352`).

**One ordering detail that decides whether phase 1 is usable.** `handle()`
refuses a `final` whose thread has no acknowledgement marker (`chat-reply:183-184`),
and a refusal blocks the card and alerts (§4.8). With every acknowledgement held,
*every* completed request would block — one blocked card and one alert each, for
a week. So the switch is read **before** that check: a held `final` is held, not
refused, and writes `.held-final` without touching the card. The distinction is
one `if` and it is the difference between a quiet pilot and a board full of
blocked cards.

**The four problems the proposal asked me to cost.**

| Its question | At `95bd4d9` | What v7 does |
|---|---|---|
| 1. Pending clarification records never trigger, because `--settle` keys on `.clarify-sent` | Correct, and **it does not breach R5**: "may be recorded as pending" (`prd.md:297-301`, `:453-454`) is permissive, so a missing pending record is a gap in what you can see, not a failed requirement | `--settle`'s two triggers each gain one more condition: `.clarify-sent` **or** `.held-clarify`, `.final-sent` **or** `.held-final`. The record lands, the status moves, and nothing is sent. Each held transition also writes one fixed internal note — "reply withheld: send switch off" — so the item never claims a requester was told something they were not (R13, R3). Two conditions and one note, in PR 7 |
| 2. Where the held reply text is visible | On the done card, which is where it already is: `hermes kanban … show <id> --json` returns the decision metadata the sender reads (`chat-reply:157-162`) | Nowhere else, deliberately. It does **not** go to the `RALPH_NOTIFY_URL` webhook (§4.8): that posts to a space of yours, so routing withheld reply text through it would put agent-composed text into Chat by a second path, which is the one thing §4.7's bound exists to prevent. PR 9's runbook names the one command that lists held operations (`ls` the markers) and the one that prints a card |
| 3. Whether `chat-route` is idempotent on a repeat | **Yes, and verified.** It writes nothing but the orchestrator card, and creates it with `--idempotency-key chat-route-<op>` (`chat-route:96`), then prints the id the create returned (`:102`). A repeat returns the existing id and makes no second card — the same no-op the mail producer relies on (`inbox-producer:19`). **One caveat, from the same source:** "Archiving a card frees its key" (`inbox-producer:22`), so archiving an orchestrator card during the pilot would let a later pass create a second one | With held markers, repeats stop after the first pass, so the caveat only matters for the single retry window. §9 carries it as an assumption and PR 5's eval asserts the skip |
| 4. Whether gating the acknowledgement on the record still fits R6 and R9 | **Consistent, and it is the easy half.** R6 and R9 bound when Ralph may *say* work is underway (`prd.md:303-308`, `:323-327`); saying nothing satisfies both trivially. The uncomfortable half is R6's first clause and E1's second: "exactly one short acceptance reply appears in that same thread" (`prd.md:477-481`) | Said plainly rather than argued away: **phase 1 satisfies no requester-facing requirement.** R6, R7, R11 and R20 are deferred, not met, and the acceptance examples that assert a send (E1, E3, E8, E9, E15, E18) are only exercised once you flip the switch. What phase 1 does satisfy is R1-R5, R12, R13 and R14 — the tracker half — plus R10 and R16, which hold either way |

**What phase 1 buys, and it is the argument for accepting that cost.** Three
things the proposal named — a backlog that works before replies do, a week of
real volume and classification quality under D6 before any requester hears from
Ralph, and a clean gate for turning replies on — and one it did not: because the
§4.4 refusals still run while held, **a refused body still refuses, still blocks
the card and still alerts** (§4.8). Phase 1 is therefore a live dry run of the
content bound, measured on real traffic, with nothing reaching a requester. That
is the cheapest version of the evidence R15's grant asks for.

**Which pull request carries it.** PR 5, as the proposal expected: the switch,
the held markers and the skip belong in the same file and the same sentence as
the ceilings — "the sender decides whether a reply may go". PR 7 carries the two
`--settle` conditions and the withheld note, because that is where `--settle`
reads markers. PR 9's runbook names the flip, the two commands under problem 2,
and the order: producer first, switch second. **One risk to watch:** PR 5 is
already the largest of the ten, and the standards cap a pull request at 400
changed lines. If the diff exceeds it, the switch splits off as PR 5b and merges
immediately after; it does not move behind PR 3, because the point of the switch
is to exist before the producer does.

---

## 5. The tracker, and the remaining unknowns

### 5.1 Unknown 1 — writing to the tracker, settled by your D1

| Path | Verdict |
|---|---|
| **Enable REST, a dedicated Ralph account, a per-user API key** | **Approved by you, 2026-11-06.** One setting, one account, a documented interface, the application's own validation. What remains is operational: create the account, generate its key, and put the key in the `tracker-creds` volume — **not** beside the chat token, so neither service holds both (§3.5, PR 9's runbook). |
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
| `--record <op>` | `chat-reply`, by `docker exec` into the tracker service, in the branch that already holds the ack until `chat-route` confirms an orchestrator card (`:193-200`) | finds or creates the item for the thread, with the staged request text and the card's `ack` as the restatement | `.tracker-recorded`, holding the item id |
| `--settle` | `chat-reply`, once at the top of each scan, the same way | for each operation with `.clarify-sent` **or `.held-clarify`** and no `.tracker-waiting`: create or update the pending record (R5) and set `Waiting on requester`. For each with `.final-sent` **or `.held-final`** and no `.tracker-closed`: set `Resolved` with the outcome from the report card. A held transition also writes the one fixed internal note from §4.9 | `.tracker-waiting`, `.tracker-closed` |

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
the hermes container (`compose/docker-compose.yml:171`) and into both new
services (§3.5), so the module resolves for all three.

**Finding the item.** Because the thread is the request, the lookup key is a
custom field holding the thread name; a message in a known thread finds its
item first, and an unconfident cross-thread match becomes a new item with a
`related to` relation (R12, E5, E13).

**Until the key file exists.** `tracker-sync` exits 0 with "tracker not
configured" when there is no key in its volume, so `chat-reply`'s gate is
satisfied, the ack goes out, and the feature runs exactly as shipped. That is
what makes PRs 7 and 8 safe to merge before PR 9's runbook is followed.

### 5.3 What an item holds, and its statuses

Stored: the requester's display name, the space and thread names, the request text as sent (truncated), Ralph's restatement, the state and the next action (R3). Not stored: other threads, member lists, attachments. Retention is the item's lifetime; the staged copy is mode `0600` under a `0700` directory, owned by the sender service's user rather than yours now that the producer runs there — `chat-reply` and `chat-route` check the mode, not the owner (`chat-reply:72-74`; `chat-route:45-47`), and the agent sees it read-only (`bootstrap/install:140-149`). **Who deletes it, and when:** `tracker-sync --settle` removes the operation's `evidence/` directory immediately after it writes `.tracker-closed`, keeping the marker files so the thread's history still counts (§4.2). Nothing chat-derived enters this repository.

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
| 3 | Producer stages evidence and creates intake cards | same | you run `--scan`; nothing runs it for you | **none** — your D5 moved the scheduling to PR 10, and the by-hand scan needs no gate |
| 4 | Decision metadata | `profiles/chat-maintainer/SOUL.md`, `bootstrap/chat-route`, `evals/test_chat_route.py` | adds optional keys; unchanged behaviour without them | none left (D2 settled: shipped text stays) |
| 5 | Ceilings, R19 checks, block and alert, **the send switch and its held markers** (§4.9), CI workflow | `bootstrap/chat-reply`, `evals/test_chat_reply.py`, `.github/workflows/chat-reply.yml`, `skills/ops/google-chat/SKILL.md` | tightens only; refuses more, sends nothing new, and the switch defaults off over a path with no traffic (§4.9) | none |
| 6 | Extract the staged-evidence reader (rule of three, §5.2) | `bootstrap/chat_evidence.py`, `bootstrap/chat-reply`, `bootstrap/chat-route` | no behaviour change; the existing evals pass untouched | none |
| 7 | The tracker script, including the held-marker triggers and the withheld note (§4.9) | `bootstrap/tracker-sync`, `evals/test_tracker_sync.py` | nothing calls it | none |
| 8 | Wire it: `--record` before the ack, `--settle` per scan, locally or by `docker exec` into the tracker service (§2) | `bootstrap/chat-reply`, `evals/test_chat_reply.py`, `evals/test_tracker_sync.py` | `tracker-sync` exits 0 "not configured" without a key file (§5.2) | the key file |
| 9 | Turn it on | `compose/README.md` runbook | documentation | the tracker account and key, the credential step, the pre-launch check, then the switch |
| 10 | **The sidecar** (your D5): the loops run as services | `compose/docker-compose.yml`, `compose/Dockerfile.notifier`, `bootstrap/chat-credential`, `bootstrap/run-chat-sender`, `bootstrap/chat-reply`, `bootstrap/chat-producer`, `evals/test_chat_sidecar.py` | the services are new entries; nothing starts until you `up` them, and neither credential volume has anything in it | **D8** (the credential), and PR 9's runbook step that fills the volume |

Order: 5 first. It adds no behaviour and closes the most exposure, because the
shipped sender has none of the ceilings today. Then 1, 4, 2, 3, 6, 7, 8, 10, 9.
PRs 2 and 3 stay separate for v2's reason: the half that reads untrusted content
and the half that acts are reviewed apart. PRs 7 and 8 are separate for the same
reason in the other direction: the file that can write to the tracker and the
change that lets a reply depend on it are different reviews. **PR 10 is last of
the code** because everything before it works by hand — the whole design runs
with `chat-producer --scan` and `chat-reply --scan` typed by you, which is also
the fallback if you refuse D8.

**What PR 10 contains, since it is new.** One `pip install` line in
`Dockerfile.notifier` (§3.5); two services in `docker-compose.yml` — the sender
(producer loop and reply loop, `chat-creds`, `../bootstrap:ro`, `../skills:ro`,
`/workspaces`) and `tracker-sync` (`tracker-creds`, `../bootstrap:ro`,
`/workspaces`), both with `restart: unless-stopped`, `DOCKER_HOST` and
`RALPH_HERMES_CONTAINER` as `notifier` already sets them (`:222-223`); two named
volumes; one loop script in the shape of `run-notifier`; the
environment-variable configuration in `chat-reply` and `chat-producer` (§3.5);
and `bootstrap/chat-credential`, the one-time re-encryption step (§3.5.1). The
eval asserts what a reviewer cannot see by reading compose: that the sender's
service definition mounts no tracker volume and the tracker service mounts no
chat volume. **`install-chat-reply-launchd` is not deleted** — it is the
fallback, and deleting it is a later change once the services have run for a
while.

**Two details that would break PR 10 on its first day.** The tracker service is
*exec'd into*, not ticked, and `docker exec` needs a running container — so it
needs an idle command and `restart: unless-stopped`, not a `command` that exits
like `init`'s does (`compose/docker-compose.yml:63`, `:92`). And PR 8 lands
before PR 10, so `chat-reply` has to work both ways: it calls `tracker-sync`
locally when no tracker container is named and by `docker exec` when one is,
which is the shape `inbox-producer` already uses for the board
(`inbox-producer:29-33`). That also keeps the by-hand fallback working, where
both scripts run as you on one machine and §0.2's honest subtraction applies
again.

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

**Which of these phase 1 exercises.** Every row that asserts a message arriving
in a thread — E1, E3, E8, E9, E11, E15, E16, E18 — is a test of the code either
way, and an observation of real behaviour only once you flip the send switch on
(§4.9). The rows that do not depend on a send — E2, E4, E5, E7, E10, E12, E13,
E14, E17 — are what phase 1 shows you on live traffic.

---

## 8. Non-goals

- **A second sender.** `chat-reply` is the sender. There is no `chat-writer`, and `tracker-sync` contains no send call (§2).
- **A tick of its own for the tracker.** `chat-reply` calls `tracker-sync`; the tracker step is invoked, never scheduled (§0.2, §2).
- **A closure producer or any tracker poll.** The final card is the trigger.
- **A chat bot identity.** Replies come from the owner's account; a requester cannot tell Ralph's text from yours. A different consent.
- **A new `gchat.py` command.** Its CLI is unchanged; the producer reuses its credential helpers and makes its own read-only `list` call (§3.4).
- **Running the sender in the container the model runs in.** The boundary is placement and it is the one thing §3.5 does not trade: neither credential volume is mounted into `hermes`, `init` or `hermes-webui`, and the sidecar carries no model and no agent.
- **A second OAuth grant, a second consent screen, or a read-only Chat scope.** D8 copies the grant you have; a narrower scope would need a change to a script `gws-cli` shares (§3.5, §3.5.1).
- **Deleting `install-chat-reply-launchd`.** It is the fallback if you refuse D8, and a later change if you do not (§6).
- **A send switch that is per space, per kind or per requester.** One variable, off or on, for the whole sender (§4.9). Anything finer is a policy surface, and policy surfaces are where ceilings get forgotten.
- **Showing held reply text anywhere but the card.** Not in the tracker beyond the one fixed withheld note, and never through the alert webhook (§4.9).
- **Holding anything but a send.** With the switch off the producer still stages, the agent still decides, `chat-route` still routes and the tracker still records. The switch gates `post()` and nothing else (§4.9).
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
2. **An API key for a dedicated tracker account can live in a compose volume mounted into one service**, and not beside the chat token. Your D1 approves the account; where its key sits is my inference, and after D5 it is the inference that keeps the two credentials apart (§3.5).
3. **`spaces.messages.list` returns `clientAssignedMessageId` and `sender.type`, and accepts a `filter` on `createTime`.** Unverified; PR 2 confirms it before anything else (§3.2, §3.4). Still the assumption that would change the design rather than a detail — but **smaller after your D6**, which took the mention annotations out of it, and they were its least certain third.
4. **"The thread is the request"** for counting and for item lookup. A requester who restates the ask in a new thread is a new request that R12 then links.
5. **A requester cannot tell Ralph's text from yours**, and you accept that.
6. **One scan at a time per instance.** The sidecar is one service with one replica running one sequential loop, so overlap is a question only for the LaunchAgent fallback, where launchd does not start a second copy of a running job. The thread lock covers the rest. I did not verify launchd's overlap behaviour against your version, and under the sidecar I no longer need to.
7. **The producer reads `CHAT_ROUTE_SPACES` from the same instance file** `chat-reply` reads, so they cannot diverge.
8. **`reply_kind` and `kind` are safe to add as optional metadata**, because `chat-reply` ignores unknown keys (`:90-93`) and PR 4 widens `chat-route`'s allow-list in the same change.
9. **"Don't cap anything" means no limit on requesters**, not the removal of the per-thread reply ceilings (§0.1, §4.2). The narrower reading, stated so you can correct it.
10. **Retired by your D6**, and kept as a number so nothing renumbers. It read "for the account from the room" as *keeping* the @mention test; you answered the other way (§0.3, §3.3).
11. **A five-minute window scanned every 60 seconds loses only what arrives while nothing is scanning**, and losing it silently is acceptable (§3.4). A restarting container narrows that window; it does not close it.
12. **You will run the one live thread-reply check** before the first real send.
13. **A public GitHub repository is a sufficient proxy for "the requester can already reach it".** Anything subtler is treated as unreachable and named in words.
14. **Splitting the tracker out of the sender is mine to settle**, because it is file structure rather than product or policy. The proposal arrived on your behalf and §2 takes it; if you want the shape it described instead — its own tick rather than an invocation — that is a sentence from you, not a redesign (§2).
15. **An operation's workspace directory is a sound place for `tracker-sync`'s own sentinels, and for §4.9's held markers.** It is already where the reply markers live (`chat-reply:27,145-147`), mode `0600`, and removed when the item closes (§5.3).
16. **Your D6's two sentences mean the room list is the whole eligibility gate**, not that the @mention test survives inside it. The reading comes from the second sentence: if the mention test stayed, deferring it would leave *nothing* eligible, because it was the only sender test the producer had. This is the inference most worth correcting if it is wrong, and correcting it is one sentence (§0.3).
17. **An approved room's traffic is low enough that one intake card and one model run per human message is acceptable**, given that your D4 waived any volume cap. I have no measurement of your rooms, so this is a guess about your usage rather than a property of the design (§3.3).
18. **A pinned machine-id file and a fixed container user are a sound key input for the sidecar's token copy** (§3.5.1). It is the inference D8 asks you to confirm, and the one that gives up a real property: a token encrypted this way decrypts wherever the volume goes.
19. **The four Python packages install cleanly on `python:3.12-slim` and cost tens of megabytes, not hundreds** (§3.5). Not measured. PR 10 checks the image-size delta before merge; if it is large, a second image built on the notifier image is the fallback.
20. **Agent sandboxes do not need to reach the sender's network, and I did not verify whether they can** (§3.5). The evidence path is a read-only bind mount (`bootstrap/install:140-149`) and the decision path is the board, so nothing in this design depends on the answer. One `docker inspect` on your machine settles it.
21. **`hermes kanban create --idempotency-key` returns the existing card on a repeat and exits 0**, which is what makes a re-run of `chat-route` a no-op (§4.9). Read from `inbox-producer:19-22` and from the key's purpose, not reproduced. The same source says archiving frees the key, so an archived orchestrator card would be recreated.
22. **The send switch defaults off without changing anything that runs today**, because nothing writes staged chat evidence at `95bd4d9` (§4.9). If you have hand-staged an operation and are running the LaunchAgent, that is the one case where PR 5 stops a reply you expected; `SKILL.md:46` suggests you have not.
23. **A week of intake with no replies is a ramp and not a broken promise to a requester**, because no requester has been told to expect anything (§4.9). This is a judgement about your pilot rooms, not a property of the code.

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
- `ralph-chat-maintainer` has a repository token (`docs/decisions/0001`) and reads untrusted chat text; the ADR names this as accepted risk. R19 now rests on sender-side checks because of it.
- The staged-evidence safety check exists twice already: `chat-reply:69-74` raises `Refused`, `chat-route:42-47` raises `ValueError`, and they test the workspace symlink in different places (`chat-reply:69`, `chat-route:38`). That is why PR 6 extracts one reader rather than adding a third copy.
- `install-chat-reply-launchd` refuses to replace a plist whose settings differ (`:44-46`) and offers no `--force`, so any later change to the scan command is a manual unload and reinstall on your machine. The send switch would be one such change, because it is a plist environment variable (`:22`) — one more reason the sidecar is the better home for it.
- `chat-reply` resolves `gchat.py` by walking two directories up from its own file (`:128`), so the sender only works in a tree that has both `bootstrap/` and `skills/` beside each other. Read-only mounts satisfy it; a flatter layout would not, silently.
- `post()` runs `gchat.py` with `sys.executable` (`:131`), which ignores the PEP 723 dependency header the script carries (`gchat.py:1-9`). On your machine that works because your Python has the packages; anywhere else it is an `ImportError` at send time, which `post()` reports as a retryable failure forever.
- `_creds_or_die` refreshes and re-encrypts the token in place (`gchat.py:146-154`, `:139-143`), so the credential volume cannot be mounted read-only even though nothing should ever write to it by hand.
- `bootstrap/install` finds the host path behind `/workspaces` by inspecting the hermes container's own mounts (`:151-160`). Any new service that stages evidence must bind the same host directory, and nothing in the repository checks that it did.
