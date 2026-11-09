---
id: dd-ralph-redmine-chat
title: How a chat message reaches Ralph, and how its replies are bounded
status: in review
owner: Zhachory1
card: t_1b391076
revision: v1
date: 2026-11-09
implements: prd-ralph-redmine
part: 1 of 2 (chat setup; the tracker is its own design)
supersedes: Zhachory1/docs#7   # closed at the owner's direction, not on substance
prs:
  - Zhachory1/docs#14
---

# How a chat message reaches Ralph, and how its replies are bounded

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker", and Google Chat is a vendor name and is named.

Status: **in review**. Nothing is built; no chat message was sent or read, no
credential touched, no tracker contacted. Merging this is the approval.

**Scope.** Everything between a message arriving and a reply leaving: the
producer that stages it, the gate that makes it eligible, the ceilings and
refusals on what the sender may post. The tracker is a sibling design; the only
thing the two share is the item record in §3.1, which this document publishes.

**Citations.** Every claim about Ralph is from a fresh clone of
`Zhachory1/ralph` at `e5ce16400cf0347c7c5945391d1278938ffa90f9` (`main`,
2026-11-09), re-read line by line. The closed design was checked at `95bd4d9`
and **six of its citations have drifted since**: `compose/docker-compose.yml` by
roughly seventy lines, `bootstrap/install` by thirty, `orchestrator/SOUL.md`
from `:19` to `:25`, `maintain-pr` by two, `run-notifier` and `run-review-loop`
by one each. I read the code; I did not run it. Redmine claims belong to the
other half. Every `prd.md:N` is `projects/ralph-redmine/prd.md` at **v7**
(`Zhachory1/docs#12`, merged 2026-11-09), the current `main` of this repository.

---

## Review header

### Decisions needed from you: two

| # | Decision | Why it is yours | If you say no |
|---|---|---|---|
| **D8** | **Put a send-capable Chat credential in a container.** Your D5 chose the sidecar, and a sidecar cannot mint its own token: `cmd_auth` calls `run_local_server(port=0)` (`gchat.py:161-166`), which needs a browser on the same machine and a loopback port nobody can publish in advance. So the token arrives from outside. The recommended path (§4.2, option **b**) copies the grant you already have: decrypt the host token once, re-encrypt it under a pinned machine id and user, write it into the sender's own volume. **Two costs, both yours:** that copy has **no machine binding left** (`gchat.py:106-114` keys on a file you now supply), and it can post as you until you revoke the grant in your Google account. The client secret never enters the container, and your host token does not move. | Where a credential that can post as you may live is not a design question. The boundary — "holding no chat credential is enforced by where the credential lives" (`profiles/chat-maintainer/config.yaml:3-5`) — is yours, and this is the first thing that spends it. | The sidecar is off. The sender stays on your machine as a LaunchAgent (`install-chat-reply-launchd`, not deleted either way) or you run `--scan` by hand. Nothing else changes: §4.2's cost table is the whole difference. |
| **D9** | **Ship the send switch, defaulting off.** One environment variable, read once per scan. With it off the producer stages, the agent decides, `chat-route` routes and the tracker records — and `post()` is never called (`chat-reply:123`, called at `:200`). You flip it when you want requesters to hear from Ralph. **Recommended: take it.** It buys a week of real volume and classification quality under D6 before anyone is replied to, and because the §4.5 refusals still run while held, it is a live dry run of the content bound with nothing reaching a requester. The cost, stated in §4.8: **phase 1 satisfies no requester-facing requirement.** R6, R7, R11 and R20 are deferred, not met. | It decides whether the first thing you see is a backlog or a conversation, and when a requester first hears from an automatic sender. A product call, not a file layout. | The sender posts as soon as the producer runs. Drop §4.8 and the two held-marker conditions the tracker design would otherwise inherit. |

**Yours, but not decisions.** Run one threaded reply in a space of your own
before the first real send: `evals/test_gchat_send.py:34-40` proves the code
asks for `REPLY_MESSAGE_OR_FAIL`, not what the service does with a thread id
captured minutes earlier. And pick pilot rooms knowing Ralph reads every human
message in them (§3.2).

### Decisions that already bind

| # | What you decided | What it fixes here |
|---|---|---|
| **D2** | Where shipped code and the PRD disagreed, the shipped behaviour wins: Ralph writes the acknowledgement and the refusal in its own words. | The sender's refusal checks are the **whole** content bound (§4.5); no constant text sits behind them. PRD v7 already carries it (`prd.md:175-177` and `prd.md:424-425`), so no `SOUL.md` line changes. |
| **D4** | "Don't cap anything. Just check for chats from the last 5 minutes for the account from the room." | No rate limit on requesters, ever (§5). A five-minute window every 60 seconds (§3.3). The per-thread reply ceilings are **not** that cap and stay (§4.4). |
| **D5** | "Take the sidecar please." | Producer and sender run as a compose service beside the agent, not as a second LaunchAgent (§4.1). D8 is the consent that answer requires. |
| **D6** | "Keep it only to approved chat rooms please" / "@ mentions and direct messages can come later." | The approved room is the whole eligibility gate (§3.2). PRD v7 already says this (`prd.md:556-562`), so this design no longer diverges from it. |
| — | The agent never holds the chat credential; the tracker is its own script the sender calls and cannot import. | §4.1 (placement is the boundary) and §4.7 (the eval asserts both halves). |

### Council dissent

Three blocking findings from the closed pull request's six rounds still shape
this document and are **kept, not re-litigated**: the lock is per thread, not
per operation, because several messages in one thread are separate operations
and operation-scoped locks never see each other (§4.4); a failed reply must
block its card and alert, because `inbox-writer` blocks nothing (§4.6); and a
widening of who gets a reply goes to the owner as a decision rather than being
settled in prose, which is how D6 reached you.

**Where this disagrees with what it was asked to carry.** The sidecar request
and the send-switch proposal were both drafted by an assistant. Both are taken;
neither as written.

1. **The sidecar is not a credential boundary.** A container holding the chat
   token that can `docker exec` into any container on the proxy
   (`docker-compose.yml:31`) is isolated from the **model**, not from the board.
   That is the only boundary §4.1 claims; more would be claiming something
   `EXEC: 1` disproves.
2. **Its headline is wrong and its own footnote is the real problem.** "A
   sidecar mints its own credential" — it cannot (`gchat.py:163`). The machine
   binding does not stop being an objection; it stops being a security property
   and becomes a cost you are asked to accept on purpose. That is D8.
3. **The send switch needed a change to be safe, not tidy.** Held with no
   marker, every held operation is rescanned every pass
   (`chat-reply:239-242`), so flipping the switch on posts the whole pilot week
   into real threads in one pass. The proposal priced that as a re-run cost; it
   is a send-flood risk, and §4.8 fixes it with a held marker.

**One disagreement with D6, on framing only.** "Keep it only to approved chat
rooms" sounds like a restriction and is a widening: inside those rooms it
removes the only sender test there was. The decision is yours and I do not
object; §3.2 states the cost where it cannot be missed.

### Scope: medium — six to seven pull requests for this half

The units: the sender's ceilings, refusals, alerting and switch; the producer's
read and eligibility; the producer's staging and card creation, kept apart from
the read because the half that parses untrusted content and the half that acts
are reviewed separately; two optional decision-metadata keys; one extracted
staged-evidence reader (that check exists twice already — `chat-reply:69-74`
raises `Refused`, `chat-route:42-47` raises `ValueError`, and they test the
workspace symlink in different places); the sidecar services and the credential
step; the runbook. The sender's work comes **first**: it adds no behaviour and
closes the most exposure, because the shipped sender has none of the ceilings
today. The sidecar comes **last**: everything before it runs by hand. The
breakdown belongs to the implementation plan.

**Length, declared rather than hidden.** This is 662 lines against the card's
450-line target — 58% of the 1146-line document it replaces, and it carries
roughly half of that document's subject matter, so it is denser per topic but
still over budget. The three blocks I would cut first, in order, and what each
costs you: §4.1's mount derivation and exec-reach paragraphs (~25 lines; you
then take the mount list on trust instead of seeing which line of code forces
each entry), §6's assumptions 5-12 (~35 lines; the inferences stop being
reviewable), and §4.5's R8 and policy-line paragraphs (~25 lines; R8's
unverified half stops being visible before a send happens). Say the word and
any of the three goes.

### The interface this half publishes

§3.1 is the item record the tracker design reads. It is the closed design's
§3.1 with **one addition**: the record carries `sender`, as
`profiles/chat-maintainer/SOUL.md:11` already documents and the closed schema
omitted. No code reads it — only the model does — but R3 ("a person opening one
item can tell who asked", `prd.md:399-401`) needs it, so the tracker should
take the requester identity from that field rather than invent a second source.
Nothing else in §3.1 moves. If D9 is taken, §4.8 names two further things the
tracker design inherits.

---

## 1. The answer, in one page

Ralph already runs half of this: an intake card, an agent that only decides, and
a sender that posts what it decided. Missing are the half that feeds it and the
half that bounds it.

```
chat ──> [NEW] bootstrap/chat-producer ──> chat-inbox/<op>/evidence/item.json
         (sender sidecar, chat token)       + intake card on `queues`
                                                      │
                                                      ▼
                                      ralph-chat-maintainer  (route | reply | ignore)
                                                      │  (hermes container, no chat credential)
                                                      ▼
        bootstrap/chat-reply (same sidecar) ─ [NEW] ceilings ─ [NEW] refusals ──> thread
          chat token, no tracker key        └─ [NEW] send switch (§4.8)
                    ├─ ack:     chat-route makes the orchestrator card, then the
                    │           tracker record, then the reply
                    ├─ clarify: the question goes out; its pending record follows
                    └─ final:   the card the orchestrator ends the graph with
```

The agent still cannot send: it holds no chat credential, and after D5 that is a
volume mounted into one service and not into the one the model runs in
(`config.yaml:3-5`). The sender still cannot improvise: the destination comes
from staged evidence, never agent text (`chat-reply:64-84`), which ships today.
New is that the sender also decides *whether* a reply may go — from counts it
reads itself (§4.4), refusals it applies itself (§4.5), and one switch you
control (§4.8). No new datastore (R14): sent-once state is the marker files that
already exist (`chat-reply:27,145-147`).

---

## 2. Options considered

| Option | Verdict |
|---|---|
| A. Put the tracker step inside `chat-reply` as well | Rejected. One file would carry ack, reply, final, ceilings, refusals **and** the tracker; its one-sentence description grows an "and", the standards' test for a file that is too big, and the tracker key would be read into the process that sends. |
| B. Write a second sender beside it | Rejected. Two senders under one policy line, two places to forget a ceiling, and it re-implements what already shipped three days after it was proposed. |
| **C. One sender; the tracker is a separate script the sender calls** | **Chosen.** `chat-reply` stays "post the maintainer's replies to the original thread" and gains the ceilings and refusals, which are *about replies*; one grep over two files still proves there is exactly one way to send (§4.7). It is the shape `chat-reply` already uses for `chat-route` (`:193-199`), so the gating, retry and failure paths are the ones in the file today. It costs a second file, a second eval, one more exit-code contract, and the sender still *drives* the ordering. |

The argument for C is not credential isolation: §4.7's bound is checked by
reading one file, and that file should not also hold tracker behaviour. D5 then
makes the split true of the process too — the sender calls the tracker script
through `docker exec` exactly as it calls `chat-route` today
(`chat-reply:194-196`), so the two credentials sit in different containers with
no rendezvous, no second tick, and still inside the thread lock.

---

## 3. The producer: the missing half

Nothing writes staged chat evidence today: `grep -rn chat-producer` at
`e5ce164` returns nothing, and `SKILL.md:46` says the host scan "does not start
inbound Chat polling". `bootstrap/inbox-producer` is the shape to copy — the
same read beside the token, the same staged evidence, the same `hermes kanban …
create --idempotency-key` through `docker exec` (`:4-5,19-22,29-33`).

### 3.1 The item record

`chat-reply` and `chat-route` read this and refuse anything that differs.

- Directory `chat-inbox/chat-<sha256 of the latest message's resource name>/`,
  with `evidence/item.json` mode `0600`, a regular file, one hard link, under
  64 KiB, no symlink on the path (`chat-reply:69-74,82`; `chat-route:42-47`).
- `kind: "inbound"`; `sender_trusted: true`; `space`; `thread`, matching
  `<space>/threads/<t>`; `messages`, oldest first, each
  `{name, text, from_ralph}` with `text` as a list of lines
  (`chat-reply:76-83`; `chat-route:63-66`; `SOUL.md:11`).
- `sender` — the requester, for the model and for R3. No code reads it;
  `SOUL.md:11` documents it and the agent uses it.
- `prior_operations`: up to 20 earlier operation ids from the same thread, none
  equal to this one (`chat-route:71-74`), copied into the orchestrator brief as
  `continues_operations` (`:86`).
- One intake card on `queues`, assignee `ralph-chat-maintainer`, body carrying
  exactly one `operation_id:` line, workspace `/workspaces/chat-inbox/<op>`
  (`chat-reply:171-172`), idempotency key the operation id.

The idempotency key is **fixed**, not chosen: the hash of the message resource
name, which makes a duplicate delivery a no-op (E2). A thread's whole history is
staged on every operation, which is how a later message reaches the right
request as steering rather than as a second item (`SOUL.md:13`; E16).

### 3.2 Eligibility: the approved room is the whole gate

`sender_trusted` is `true` when all four hold, and there is no fifth test: the
space is in `CHAT_ROUTE_SPACES` — the same list `chat-reply` reads from the same
instance file, capped at eight spaces (`chat-reply:41-53`); the message was
created inside the five-minute window (§3.3); its `sender.type` is a human, not
`BOT`; and its `clientAssignedMessageId` does not start with `client-ralph-`.

The last two are R10. Replies carry `messageId = client-ralph-<kind>-<digest>`
(`chat-reply:131-134`; `gchat.py:33`), so the producer recognises Ralph's own,
marks them `from_ralph: true` and stages no operation. The agent's `ignore`
decision (`SOUL.md:19`) is the backstop, at one card and one model run each.

Nothing is asked about *who* the human is or what the message says: a message is
eligible because of the room it was sent in. That is R18 as the merged PRD now
states it (`prd.md:556-562`), so no divergence is left to reconcile. **The cost,
where it cannot be missed:** every human message in a pilot room is an
operation, an intake card and one model run — your conversation with someone
else in that room included. `ignore` ends those at one card each, D4 waived any
volume cap, and the PRD states the consequence itself — nothing bounds the
volume (`prd.md:578-584`). Choose pilot rooms you are willing to have Ralph
read in full.

`sender_trusted` therefore stops being a judgement about a person and becomes a
restatement of the first condition. `chat-route:63-64` and `chat-reply:78` still
refuse a file that lacks it, so the field keeps its defensive value against a
forged `item.json`; it no longer encodes a second policy. The gate that matters
is the room list, and it is yours: an instance file, not a constant in the code.

**Unverified and load-bearing:** that `spaces.messages.list` returns
`clientAssignedMessageId` and `sender.type`. Documented as part of the Message
resource, which I am recalling rather than reading; one read-only call confirms
them before anything is built. Bounded either way — without the first, `ignore`
carries R10 alone at one card per reply Ralph posts; without the second, a bot
notice costs one card.

### 3.3 The read: five minutes, every scan

One `spaces.messages.list` per space in `CHAT_ROUTE_SPACES`, filtered to
`createTime > (now − 5 minutes)`, newest first; nothing walks history,
backfills, or keeps a cursor. Every 60 seconds — a loop sleep in the service
(`run-notifier:10` is the pattern), a `StartInterval` on the fallback
(`install-chat-reply-launchd:19`) — so each message falls inside roughly five
windows. The repeats cost nothing: the operation id is the hash of the message
name (`chat-reply:82`) and the card carries it as its idempotency key, so a
message already staged makes no second directory and no second card
(`inbox-producer:19-22`).

**Why not shell out to `gchat.py messages`.** That wrapper keeps five fields per
message and discards everything else before building its output, including both
fields §3.2 runs on (`gchat.py:222-240`); `--json` does not bring them back. So
the producer imports the credential helpers from the same module (`_key`,
`_load_creds`, `:106-114,129-136`) and makes its own read-only `list` call.
`gchat.py`'s CLI is unchanged and `messages().create` stays reachable through
exactly one path (`:264`), which is what §4.7 asserts.

**What it loses, plainly.** A message older than five minutes when the scan runs
is never seen, and nothing notices: if the service is down or the network is out
for six minutes, requests sent in that gap are lost silently — to Ralph; the
requester sees no reply and cannot tell whether the message was read. That is
the cost of "just check the last 5 minutes" and it is accepted; a watermark
would close it and is the obvious later change. **Also unverified:** that the
API accepts a `filter` on `createTime`. If not, the producer pages newest-first
and stops at the first message older than the window — one more page read.

---

## 4. The sender: where it runs, and how its replies are bounded

Shipped `chat-reply` bounds three things: where a reply goes, that it goes once
per operation, and that a final follows an acknowledgement (`:64-84`,
`:145-151`, `:183-184`). It does **not** bound how many replies a *request*
gets: it counts per operation — per message — and every message in a thread is a
new operation.

### 4.1 Where it runs (your D5)

The old answer to "why not in a container" rested on cryptography: the token's
key derives from your machine id and unix user, so it decrypts nowhere else.
That was never the real argument. What `config.yaml:3-5` protects against is a
**model** that reads untrusted chat text as its whole job, with a terminal and a
repository token (`:6-8`, `:12-17`). The ceilings and refusals are a bound only
if there is no second way to send, and a token reachable by that model is the
second way. None of that is about Docker, so the sidecar keeps the boundary by
**placement**:

| Service | Holds | Mounts | Why it cannot reach the other's credential |
|---|---|---|---|
| `chat-sender` (producer loop + reply loop) | the Chat token, in its own named volume | `chat-creds` (rw), `../bootstrap:ro`, `../skills:ro`, `${RALPH_WORKSPACES}:/workspaces` | The tracker key is in a volume it does not mount. It runs no model and no agent; its only executables are two read-only repository scripts. |
| the tracker service (invoked, not ticked) | the tracker key, in its own named volume | its own creds volume, `../bootstrap:ro`, `/workspaces` | The Chat token is in a volume it does not mount, and the file it runs contains no send call (§4.7). |
| `hermes` (unchanged) | the repository token | as today (`compose/docker-compose.yml:163-171`) | Neither credential volume is mounted into it, and neither is `hermes-home` — the one volume `init`, `hermes` and `hermes-webui` all share (`:90`, `:164`, `:369`, `:371`). |

**Every mount is forced by a line of code, not chosen.** `../skills:ro` because
`post()` resolves the sender two levels up from its own file and shells out to
`<repo>/skills/ops/google-chat/gchat.py` (`chat-reply:128`), as `init` already
mounts it (`docker-compose.yml:93`). `/workspaces` at the same host path as the
hermes service, because `bootstrap/install` finds the chat-maintainer's
read-only evidence mount by inspecting *that* container's `/workspaces` source
(`:170-179`, `:186-191`) — a producer writing anywhere else stages evidence the
agent cannot open. `chat-creds` writable, not read-only, because a refresh
rewrites the token in place (`gchat.py:139-143`, from `:150-152`).

**The exec reach, stated rather than waved at.** `socket-proxy` runs `EXEC: 1`
(`docker-compose.yml:31`), so any service on the proxy can exec into any
container it can name, and the sender needs exactly that: it is how `chat-reply`
reads a card and runs `chat-route` today (`:157,194,222`). **Not a new class of
risk** — `notifier` already holds a chat webhook URL and the same proxy access
(`:226-228`), `review-loop` a repository token that can push (`:261`). New is
one combination, a credential that can post *as you* in a container that can
exec, and the answer is the mount list above. Ralph also already runs two
host-shaped loops as services for a stated reason — "a crash is visible as a
restarting container instead of a silently missing job" (`run-notifier:4-5`;
`run-review-loop:3-4`) — which is the failure §3.3 accepts from an unloaded
LaunchAgent.

**Three costs the plan carries, named so they are not discovered late.** The
image needs four Python packages `gchat.py` imports (`:46-52`) and
`Dockerfile.notifier` does not install (`:6-16`) — extend it rather than add a
second image, and measure the delta, which **I did not**. `instance_config`'s
three variables become environment variables with the same validation
(`chat-reply:41-53,49-50`), with the file path kept for the fallback. And the
one token stays one: its scope list already holds both
`chat.messages.readonly` and `chat.messages.create` (`gchat.py:60-66`), so a
read-only producer credential would cost a second consent screen and a change
to a script `gws-cli` shares (`:59`).

### 4.2 The credential, which is the whole of D8

| Path | What it costs | Verdict |
|---|---|---|
| **a. Mint inside the container**: pin a machine-id file, publish a temporary port, run `gchat.py auth` through it | Needs `port=0` to become a fixed port — a change to a script `gws-cli` shares (`gchat.py:59,163`) — plus a published port on a service that should publish none, and a browser redirect that reaches it. Produces a genuinely **second** OAuth grant: a second consent screen and a second thing to revoke. | Rejected. The change lands in the wrong file and the port is a new surface. |
| **b. Mint on your machine as today, re-encrypt for the sidecar** | A one-time script reads the host token with the host key and writes it back under HKDF of `"<pinned machine id>:<pinned user>:gws-cli"` (`gchat.py:106-114`) into the sender's volume. No `gchat.py` change: `_load_creds` and `_creds_or_die` need only the token and the salt, because the client id and secret sit inside the authorized-user JSON (`:129-136,146-154`). **The encrypted client secret never enters the container.** | **Chosen.** The only path that keeps `gchat.py` unchanged and the client secret on your machine. |
| **c. A Docker secret holding the refresh token** | `gchat.py` reads one encrypted path and nothing else (`:57,129-136`), so this needs a code change, and compose secrets are a file mount with extra steps. | Rejected: option b with more moving parts. |

**What b gives up, because it is what D8 approves.** Today
`chat_token.json.enc` is useless copied off your machine: the key derives from
the machine id, your unix user and the shared salt (`gchat.py:106-114`). After
b, a copy exists whose three key inputs all travel with the volume, so anyone
who can read that volume can post as you until you revoke the grant. The volume
is mounted into one service and is not `hermes-home`; that is the whole of the
protection, and it is placement rather than cryptography. **The fallback, if you
refuse:** both scripts run as you, from the repository, with `--instance-env`
and `--scan`, exactly as `install-chat-reply-launchd` already invokes
`chat-reply` (`:18-19`). Everything here still works; only the scheduling and
the container go away.

### 4.3 One decision, four meanings: `reply_kind`

`reply` today means a refusal, a direct answer or a clarifying question
(`SOUL.md:18`), and the sender cannot tell which. The decision gains a
`reply_kind` in `{clarify, answer, decline}`, and `chat-route`'s allowed-key set
(`:55-56`) grows by that key and by `kind`. Markers become per kind —
`.clarify-sent`, `.decline-sent`, `.reply-sent`, `.receipt-sent`, `.final-sent`
— extending the three that ship (`chat-reply:27`). `kind`, one of R4's
actionable kinds, is recorded on `route` because an **update** is acknowledged
and nothing more (R6, `prd.md:428-429`; E10, `prd.md:673-674`): `chat-route`
omits the "finish with one ralph-chat-maintainer card" line from the brief when
the kind is `update` (`:89-92`), so no final card exists to send. Both keys are
safe to add — `chat-reply` ignores unknown metadata keys (`:90-93`) and
`chat-route`'s allow-list widens in the same change.

### 4.4 The ceilings, read by the sender from the thread

The **request is the thread**. `chat-reply` takes an exclusive `flock` on a lock
file named for the thread, before it counts and until it has marked, then counts
markers across this operation's directory and every `prior_operations` directory
in `item.json` — staged evidence, never agent text.

| Rule | Ceiling | On breach |
|---|---|---|
| A17 (`prd.md:797-802`; applied at E18, `prd.md:714-718`) | at most 2 `clarify` per thread: one question, one follow-up | refused; the card is blocked and an alert posted (§4.6); the request stays pending |
| A15 / R7 (`prd.md:440-442`) | at most 1 `decline` per thread, and a decline creates no tracker item | refused and blocked |
| R6 (`prd.md:416-417,426-427`) | at most 1 `ack` per thread, and only after the tracker item exists | held, retried next scan |
| R11 (`prd.md:464-465,482-483`) | at most 1 `final` per thread, and only after `ack` — shipped at `chat-reply:183-184` | refused |

The lock is on the thread, not the operation, for the reason the first review
round found: several messages in one thread are separate operations, and
operation-scoped locks never see each other. **It counts sent markers only**, so
a reply held by the send switch (§4.8) cannot consume a ceiling.

**These are not the cap D4 waived.** D4 removed any limit on *requesters*; this
table limits *Ralph* — how many times one request may be messaged about. All
four rules come from the PRD, and together they are the sender-side half of the
R15 grant. Remove them and R15 has no enforcement and §4.7 nothing to assert.

### 4.5 The content bound (R19), the thread bound (R8), the policy line

The chat-maintainer has `terminal` and a repository token
(`config.yaml:6-8,12-17`); on the personal instance that token is the owner's
(`docs/decisions/0002`), and both ADRs record the exposure as accepted. A reply
can therefore quote a private repository the requester cannot open, so the bound
cannot be "the agent could not reach it". **It is a refusal in the sender, not a
trim.** Before any send of agent text — `ack`, every `reply_kind`, `final` — the
sender refuses text containing a tracker reference (the instance host, an
`/issues/<n>` path, a bare `#<n>`; R1, `prd.md:594-595`); a URL not in a
`reachable` list the sender builds, where a URL qualifies only if provably
public — for a GitHub pull request, one API read showing the repository is
public, everything else named in words (R19, `prd.md:597-598`); or a
credential-shaped string (`SOUL.md:29`). **The honest limit:** prose cannot be
proven to come from this request. A refusal rule catches references and links,
not a sentence that *paraphrases* a private document. D2 makes these checks the
whole content bound, so that residual risk is accepted rather than designed
away — smallest on `ack`, largest on `final`.

**R8 is half done.** The send asks for `REPLY_MESSAGE_OR_FAIL`, and a strict
send with no thread, or a thread in another space, fails before any API call
(`gchat.py:249-259`; `test_gchat_send.py:34-52`). What is not verifiable from
here is whether the service honours it for a thread id captured earlier, and
comparing the returned thread to the staged one is **not available**:
`cmd_send` returns only `name` and `createTime` (`:265-266`) and `post()`
accepts `sent is True` alone. Asserting the returned `name` begins with the
staged space is cheap and worth adding, but cannot detect a wrong thread in the
right space; your pre-launch check closes that gap, and the token holds no
delete scope (`:60-66`), so a wrong-thread message can be reported and not
withdrawn.

**`SKILL.md:44`** already names `chat-reply` as the one owner-approved automatic
exception, for "acknowledgement, reply and final response", and says it never
takes a destination from agent text. It mentions neither the tracker, the
ceilings, `reply_kind` nor a switch, so it is amended to match what the script
then does, **in the same pull request as the code**. `SKILL.md:46` gains the
sidecar beside the LaunchAgent.

### 4.6 The close, and how you see a failure

The orchestrator ends a chat graph with exactly one `ralph-chat-maintainer` card
whose parents are the cards that do the work (`orchestrator/SOUL.md:25`). Kanban
dependencies mean it does not run until every parent is done, so **one work card
finishing is not the request finishing** by construction (R11, E7), and no
tracker poll is needed. The sender posts the `final` and marks it; the tracker's
settle pass closes the item afterwards, so a tracker outage delays the close and
never re-sends the reply. **Silence on failure (E11)** partly follows from the
graph — a blocked or abandoned work card never lets the final card run — but a
final card whose parents completed with a failed result is told to "say so
plainly" (`SOUL.md:23`), and D2 keeps that; PRD v7 splits E11 the same way
(`prd.md:676-682`).

**Failure is invisible today.** `chat-reply` signals by exit code and stderr
only — `1` retry, `2` refused (`:160,190,198,208`). It blocks nothing and tells
nobody, so a refused reply is silent and a retryable failure repeats every 60
seconds without a word. Two mechanisms already exist in `bootstrap/maintain-pr`
and are reused: block the card through `hermes kanban … block` (`:398`), and
post the best-effort alert that tolerates its own failure (`:303-305`,
`RALPH_NOTIFY_URL` or `GCHAT_WEBHOOK_URL`). **Both fire on a refusal and on a
retry alike**, because the failure that matters most is the one that repeats
quietly; once per operation and reply kind, guarded by an `.alerted-<kind>`
sentinel. `notifier/poller.py` stays the backstop: it recovers the reason from a
`BLOCKED:` comment (`:91-94`) and announces each once (`:121-127`).

### 4.7 The eval, and the CI gap

Beyond the fifteen tests `evals/test_chat_reply.py` already has:

| Assertion | Guards |
|---|---|
| a third `clarify` in one thread sends nothing and blocks the card; a second `decline` likewise, and a decline calls no tracker method | A17, A15, R7, E3, E18 |
| two operations in one thread serialise on the thread lock | A17 |
| a body containing the tracker host, `/issues/12` or `#12` is refused; a URL outside the `reachable` list is refused and the same outcome in words is sent | R1, R19, E9 |
| a tracker write failure sends no acknowledgement | R9, E1 |
| `chat-reply` imports no tracker module, and the tracker script contains no send call | §2, R15 |
| `kind: update` leaves no final card to send | E10 |
| exactly one `messages().create` path exists, and `REPLY_MESSAGE_OR_FAIL` is the only reply option used | R15, R8 |
| a message in an unlisted space, or older than the window, is not staged; an unmentioned message in a listed space **is** | E14, E16 |
| with the switch off: `chat-route` runs, `post` is not called, `.held-<kind>` is written, the next scan skips it, and a body failing a §4.5 check is still refused and still blocks | §4.8, R19 |
| turning the switch on sends nothing for an operation already carrying a held marker | §4.8, R11 |

**The gap, still open.** `.github/workflows/` holds `scope-check.yml` and
`skills-public.yml`, and the second runs two named evals only (`:33`, `:36`).
`test_chat_reply.py`, `test_chat_route.py` and `test_gchat_send.py` are in
neither, so nothing fails a build today if someone adds a second send path or
removes a ceiling. The workflow ships with the sender's code, triggered on
changes to `bootstrap/chat-*`, `skills/ops/google-chat/**` and
`evals/test_chat_*`.

### 4.8 The send switch (D9)

One environment variable, read once per scan, defaulting to **off**. With it off
the sender reads the decision, validates the destination, runs every §4.5 check,
runs `chat-route`, calls the tracker — and does not call `post()`
(`chat-reply:123`, called at `:200`). Merging it off changes nothing that runs
today, because there is nothing to send: no producer exists and `SKILL.md:46`
says the host scan does not start inbound polling.

**The one change to the proposal, and it is a safety fix.** The proposal holds a
reply by writing *no* marker. That cannot ship. `scan()` skips an operation only
when its sent marker exists (`chat-reply:239-242`), so a held operation is
re-handled every pass — which the proposal noticed and priced as a re-run cost.
The real cost is on the far side: **the day you flip the switch on, one pass
sends the whole pilot week at once**, an acknowledgement and a final into every
thread Ralph has seen, backdated and unasked for. The ceilings do not stop it,
because no thread has spent any of them. So a held reply writes `.held-<kind>`
in place of the marker `mark()` would have written, by the same `O_EXCL` open
(`:145-147`), and both `scan()` and `handle()` treat either marker as "nothing
to do here". The loop stays bounded, one `docker exec` per operation per kind
rather than one per minute forever; the ceilings stay exact, because they count
sent markers only; and **flipping the switch on is inert for everything already
held** — which, for a request whose acknowledgement was held, is what R11
requires anyway.

**One ordering detail that decides whether phase 1 is usable.** `handle()`
refuses a `final` whose thread has no acknowledgement marker (`:183-184`), and a
refusal blocks the card and alerts (§4.6). With every acknowledgement held,
*every* completed request would block — one blocked card and one alert each, for
a week. So the switch is read **before** that check: a held `final` is held, not
refused, and writes `.held-final` without touching the card. One `if`, and the
difference between a quiet pilot and a board full of blocked cards.

**What phase 1 does not satisfy, said plainly rather than argued away.** No
requester-facing requirement. R6, R7, R11 and R20 are deferred, not met, and the
acceptance examples that assert a message arriving — E1, E3, E8, E9, E11, E15,
E16, E18 — are tests of the code either way and observations of real behaviour
only once you flip the switch. It does satisfy R10 and R16, which hold either
way, plus the tracker half. **Two things the tracker design inherits** if D9 is
taken: its pending and close triggers key on send markers, so both must accept
the held marker too, and each held transition needs one fixed internal note —
"reply withheld" — so an item never claims a requester was told something they
were not (R3, R13).

**Where the held text is visible.** On the done card, where it already is: the
sender reads `decision`, `ack` and `body` straight out of that card's metadata
(`chat-reply:168-176,182`), and deliberately nowhere else — not through the alert
webhook, which posts to a space of yours, because routing withheld reply text
through it would put agent-composed text into Chat by a second path, the one
thing §4.7's bound exists to prevent.

---

## 5. Non-goals

- **A second sender**, and **a tick of its own for the tracker**: `chat-reply`
  is the sender, the tracker script contains no send call, and it is invoked,
  never scheduled (§2).
- **Running the sender in the container the model runs in.** Neither credential
  volume is mounted into `hermes`, `init` or `hermes-webui`, and the sidecar
  carries no model and no agent (§4.1).
- **A second OAuth grant, a second consent screen, or a read-only Chat scope**
  (§4.2), and **a new `gchat.py` command** (§3.3).
- **Deleting `install-chat-reply-launchd`.** It is the fallback if you refuse
  D8, and a later change if you do not.
- **A send switch that is per space, per kind or per requester.** One variable
  for the whole sender; anything finer is a policy surface, and policy surfaces
  are where ceilings get forgotten. **Holding anything but a send** is also out:
  with the switch off the producer still stages, the agent still decides,
  `chat-route` still routes and the tracker still records (§4.8).
- **A watermark or any backfill** of messages older than the window (§3.3),
  **unsending**, **a retry that re-sends**, and **sweeping pending requests**
  (A18).
- **Any rate limit or volume cap on requesters** (D4). §4.4's per-thread
  ceilings are not that, and stay.
- **Reaching "all chats"** beyond `CHAT_ROUTE_SPACES`: a change to §3.2, not a
  setting, because the producer would have to derive the space set itself and
  the cap of eight (`chat-reply:49-50`) would go with it. **Direct messages**
  and **any @mention test** are likewise deferred (D6: "can come later"), one
  condition each in §3.2 when you want them.
- **The tracker.** Item schema, statuses, classification and the tracker account
  are the sibling design. This document publishes §3.1, names the two things
  §4.8 forces on it, and says nothing else.
- **Editing the PRD.** `prd.md` is at v7 (`Zhachory1/docs#12`), which carries D2
  and D6 and has already corrected "Current state" and R15 to say that
  `bootstrap/chat-reply` ships. Nothing here needs the PRD changed; what v7
  left for this design — A17's ceiling and R19's provenance rule, its design
  unknowns 5 and 12 — is settled in §4.4 and §4.5.
- **Implementing any of this.** Work starts after this document is merged.

---

## 6. Assumptions

Every inference the PRD, the card, your answers and the verified source did not
state.

1. **`spaces.messages.list` returns `clientAssignedMessageId` and `sender.type`,
   and accepts a `filter` on `createTime`** (§3.2, §3.3). Unverified; one
   read-only call confirms it before anything is built. Still the assumption
   that would change the design rather than a detail, though D6 shrank it by
   removing the mention annotations it used to include.
2. **A pinned machine-id file and a fixed container user are a sound key input
   for the token copy** (§4.2). What D8 asks you to confirm, and it gives up a
   real property: a token encrypted this way decrypts wherever the volume goes.
3. **"The thread is the request"** for counting (§4.4). A requester who restates
   the ask in a new thread is a new request, which R12 then links.
4. **A requester cannot tell Ralph's text from yours**, and you accept that; a
   chat bot identity is a different consent.
5. **One scan at a time per instance** — one service, one replica, one
   sequential loop — and **the producer reads `CHAT_ROUTE_SPACES` from the same
   instance file** `chat-reply` reads, so the two cannot diverge (§3.2).
6. **A five-minute window scanned every 60 seconds loses only what arrives while
   nothing is scanning**, and losing it silently is acceptable (§3.3).
7. **You will run the one live thread-reply check** before the first real send,
   and **a public GitHub repository is a sufficient proxy for "the requester can
   already reach it"**; anything subtler is unreachable and named in words
   (§4.5).
8. **An approved room's traffic is low enough that one intake card and one model
   run per human message is acceptable**, given D4 waived any volume cap. I have
   no measurement of your rooms, so this is a guess about your usage rather than
   a property of the design (§3.2).
9. **The four Python packages install cleanly on `python:3.12-slim` and cost
   tens of megabytes, not hundreds** (§4.1). Not measured.
10. **Agent sandboxes do not need to reach the sender over the network, and I
    did not verify whether they can.** Nothing here depends on it: evidence
    reaches the agent as a read-only bind mount (`bootstrap/install:170-179`)
    and decisions come back on the board. The isolation spike probes mounts, the
    socket and the board rather than reachability
    (`docs/spikes/sandbox-isolation.md:28-41`), and the proxy refuses the
    network API (`docker-compose.yml:36`). One `docker inspect` settles it.
11. **`hermes kanban create --idempotency-key` returns the existing card on a
    repeat and exits 0**, which makes a rescan a no-op. Read from
    `inbox-producer:19-22`, not reproduced; the same source says archiving a
    card frees its key, so an archived orchestrator card would be recreated.
12. **The send switch defaults off without changing anything that runs today**,
    because nothing writes staged chat evidence at `e5ce164`, and **a week of
    intake with no replies is a ramp, not a broken promise**, because no
    requester has been told to expect anything (§4.8). The second is a judgement
    about your pilot rooms, not a property of the code.

### Observations, out of scope

Reported, not fixed.

- `chat-route` refuses any decision-metadata key outside
  `decision, body, ack, worker_session_id` (`:55-56`) while `chat-reply` ignores
  extra keys (`:90-93`), so a worker that adds a diagnosis note makes
  `chat-route` exit non-zero and `chat-reply` retries "cannot confirm
  orchestrator card" every scan forever (`:197-199`).
- If a send succeeds and the marker write then fails, `chat-reply` says "retry
  is safe" (`:208`); the retry reuses the same `messageId`, and what the service
  returns for a duplicate custom id I did not verify.
- No chat eval runs in CI (§4.7), and `evals/test_inbox_writer.py` is in no
  workflow either.
- `install-chat-reply-launchd` refuses to replace a plist whose settings differ
  (`:46`) with no `--force`, so any later change to the scan command is a manual
  unload and reinstall — one more reason the sidecar is the better home for the
  switch.
- `chat-reply` resolves `gchat.py` by walking two directories up from its own
  file (`:128`) and runs it with `sys.executable` (`:131`), so a flatter layout,
  or a Python without the four packages, fails silently or at send time.
- `_creds_or_die` refreshes and re-encrypts the token in place
  (`gchat.py:146-154`), so the credential volume cannot be mounted read-only;
  and `bootstrap/install` finds the host path behind `/workspaces` by inspecting
  the hermes container's own mounts (`:186-191`), so any new service that stages
  evidence must bind the same host directory and nothing checks that it did.
