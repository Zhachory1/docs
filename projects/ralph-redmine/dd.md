---
id: dd-ralph-redmine
title: Build the work tracker onto the chat path Ralph already shipped — one producer to feed it, one tracker step inside the host sender, and host-side ceilings on every reply
status: in review
owner: Zhachory1
card: t_b5db2cf0
revision: v3
date: 2026-11-05
implements: prd-ralph-redmine
prs:
  - Zhachory1/docs#7   # v1, v2, v3
supersedes: none
---

# Build the work tracker onto the chat path Ralph already shipped — one producer to feed it, one tracker step inside the host sender, and host-side ceilings on every reply

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester", the tracker is "the work
> tracker" (a self-hosted Redmine), and Google Chat is a vendor name and is
> named. The PRD's banner applies unchanged.

Status: **in review** — a design, awaiting your approval. Nothing is built, no
chat message was sent or read, no REST API was enabled, no credential and no
work instance was touched. Merging this pull request is the approval.

Implements `prd-ralph-redmine` v5, merged at `Zhachory1/docs#6`. The PRD is not
re-opened here, but §0 lists three places where shipped code now disagrees with
it, and D2 asks you which one wins.

**Citations.** Every claim about Ralph is from a fresh clone of
`Zhachory1/ralph` at commit `95bd4d960e9178a0ea00c8791afa75b851138e1c`
(2026-11-05, `main`). v2 was written against `b41c99d` (2026-11-02). Thirty-eight
commits landed between them and several of v2's central claims are no longer
true. I read the code; I did not run it, send a message, or call any API.
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
| `SKILL.md:44` needs an owner-approved paragraph (D2) | `SKILL.md:44` already names `bootstrap/chat-reply` as the one automatic exception, "owner-approved", for "acknowledgement, reply and final response" | v2's D2 **no longer exists as a text edit.** What is left is a scope question (D2 below). |
| The chat-maintainer has no network and no credential | It has the `terminal` toolset and a repository token on every instance, and the owner's own token on the personal instance (`profiles/chat-maintainer/config.yaml:6-16`; `docs/decisions/0001`, `0002`) | **v2's R19 argument no longer holds** (§4.4). The agent can read what the owner can read. |
| `sender_trusted` has no host side at all | Still no code *sets* it. But `chat-reply` and `chat-route` now both *require* it `is True`, and `chat-reply` also requires the space to be in `CHAT_ROUTE_SPACES`, at most **eight** spaces (`chat-reply:44-50,78-80`; `chat-route:63-64`) | A real gate exists and it is **narrower than R18** (D4c). |
| Own-message suppression needs a `SENT:` ledger on the board (v2 §4.2) | Replies carry a `client-ralph-*` message id "Ralph can recognize" (`gchat.py:31-33`; `chat-reply:134`), and `item.json` messages carry `from_ralph` (`SOUL.md:11`) | The ledger is **dropped**; the producer recognises Ralph's own messages by id (§3.2). |
| Request identity is a tracker item; counts live on it | Operation id is `chat-` + sha256 of the latest message name, enforced by both scripts (`chat-reply:80-81`; `chat-route:68`). Sent-once state is a marker file per operation (`chat-reply:27,145-151`). `item.json` lists `prior_operations` for the thread (`chat-route:71-74`) | Counts can live **on the thread's operation directories**, no tracker field needed to enforce a ceiling (§4.2). |
| A closure producer polls the tracker for `Resolved` + outcome | The orchestrator ends every chat graph with one `ralph-chat-maintainer` card that depends on the work cards and writes the final reply (`profiles/orchestrator/SOUL.md:19`; `chat-route:89-92`) | **The closure producer is dropped.** The final card is the trigger (§4.6). |
| Nothing is wired to CI | Still true: the workflows are `scope-check.yml` and `skills-public.yml`, and the latter runs two named evals (`skills-public.yml:22-23,33,36`). `evals/test_chat_reply.py`, `test_chat_route.py` and `test_gchat_send.py` are in neither | The bound R15 asks to prove is **still unenforced between hand runs** (§4.7). |
| Inbound chat is read by a producer (v2 PRs 2-3) | **Still no producer.** `SKILL.md:46` says the host scan "does not start inbound Chat polling", and nothing in the repository writes `chat-inbox/<op>/evidence/item.json` | The producer is **still the largest missing piece**, and it now has a fixed schema to meet (§3). |
| No tracker integration | None. `grep -ri redmine` over the clone finds nothing | Unchanged: D1. |

**The three places shipped code and the PRD disagree** (decision D2):

1. **Acknowledgement text.** PRD R6: one short sentence that Ralph has it. v2 made it a writer constant. Shipped: the agent writes `ack` "in your own words, saying what you understood and that you are working on it" (`SOUL.md:17`).
2. **The "cannot help" reply.** PRD R7: "I am not sure how to help with that." Shipped: the agent writes a plain refusal "with the reason and where they could go instead" (`SOUL.md:18`), and one decision, `reply`, covers a refusal, a direct answer and a clarifying question alike.
3. **A request that fails.** PRD A14: silence toward the requester. Shipped: "If the work failed or is incomplete, say so plainly" (`SOUL.md:23`).

---

## Review header

### Decisions needed from you: four

| # | Decision | Why it is yours | If you say no |
|---|---|---|---|
| **D1** | **Enable the tracker's REST API and give Ralph its own tracker account.** Unchanged from v2. | R17 forbids a runtime change without your word, and the setting is instance-wide. | The fallback is `rails runner` on the tracker host (§5.1), which skips the application's validation and needs a second credential. Nothing tracker-side runs until you choose; chat replies still work without it. |
| **D2** | **Which wins where shipped `chat-reply` and the PRD disagree** (§0, items 1-3). **Recommended: the shipped behaviour, with the safety moved from the sentence to the host.** Ralph keeps writing its own acknowledgement and refusal text, and tells a requester when work failed. `chat-reply` then refuses any text that carries a tracker reference or a URL not on a public list (§4.4), and enforces the ceilings in §4.2. | You wrote or approved the shipped behaviour; the PRD says otherwise; I will not pick between two of your own decisions. Option A (recommended) trusts agent prose bounded by host checks. Option B restores the PRD: constants for R6 and R7, silence on failure — safer, and it removes behaviour you chose to ship. | Under B the shipped `SOUL.md` lines change and PR 4 grows by a few lines. Either way the PRD needs a v6 amendment, because R6, R7 and A14 would not describe what runs. |
| **D3** | **Install a second host LaunchAgent for the producer**, beside the one for `chat-reply`. | `chat-reply` already has `bootstrap/install-chat-reply-launchd` (60-second scan). The same constraint holds for the producer: the chat credential's key is derived from the machine and the user (`gchat.py:72-76,112`), so it cannot run in a container. | Run it by hand. Replies already flow automatically once a card exists; only *noticing new messages* waits on you. |
| **D4** | **The eligible surface. Three things sit outside what R18 settled.** (a) While a clarification question is open, that one thread is eligible for that one requester without an @mention (§3.3). (b) No volume cap on who can create work. **(c) New: the shipped gate is an explicit list of at most eight spaces (`CHAT_ROUTE_SPACES`), and a direct message is its own space.** PRD R18 and your `prd.md:62` comment say "all chats for a user and rooms they define". | (c) is the one that matters. As shipped, "all chats" is not reachable: a ninth conversation is refused by design. v3 keeps the list as the v1 boundary (smaller than R18, never wider) and makes the producer derive eligibility *inside* it. | Say "lift it" and the producer must derive the space set itself and the cap of eight goes. That removes the only host-side limit on volume. Say "cap it" for (b) and §3 gains a per-sender daily counter. |

**One pre-launch action is also yours:** run one threaded reply in a space of
your own and confirm it lands in the thread. `test_gchat_send.py` proves the
code asks for `REPLY_MESSAGE_OR_FAIL`; it cannot prove what the Chat service does
with an old thread id. I cannot tell from the repository whether you have done
this. `SKILL.md:46` says "after a controlled test".

### Council dissent

None for v3 yet. v2's independent review found three blocking issues, all
accepted and fixed in v2 (recorded in v2's review reply on the PR): `inbox-writer` blocks
nothing; the lock was on the operation rather than the request; and thread-scoped
eligibility was settled in prose rather than escalated. **v3 keeps all three
fixes** and re-grounds them: the block precedent is re-cited at the new line
numbers (§4.8), the lock is now per thread (§4.2), and the eligibility widening
is D4(a).

### Scope: medium — seven pull requests

Smaller than v2's eight and with less new machinery, because the sender, the
strict-thread send, the policy line, the destination check and a launchd
installer now exist. The work left is the producer, the tracker step, the
host-side ceilings, and the CI wire that makes them hold. Breakdown in §6.

### What changed since v2

- **`chat-writer` and the closure producer are gone**; their jobs move into `chat-reply` and the existing final card.
- **Three PRD conflicts surface as D2**, and the eligibility gate's eight-space cap surfaces as D4(c).
- **R19's enforcement is rewritten** (§4.4): the evidence-root argument is false now that the agent holds a repository token.
- **Own-message suppression uses the message id Ralph already sets** rather than a ledger comment on the board.
- **Ceilings count across the thread's operation directories**, so R15 is enforceable without the tracker.
- **Observations added** (§9): a metadata-key mismatch between `chat-route` and `chat-reply` that can make an acknowledgement retry forever, and silent refusals in `chat-reply`.

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
        bootstrap/chat-reply  (host, exists) ── [NEW] ceilings ── [NEW] tracker step ── send ──> thread
                │                                                                       
                ├─ ack: chat-route makes the orchestrator card, THEN the tracker item, THEN the reply
                └─ final: the card the orchestrator ends the graph with; THEN the tracker closes
```

1. **The agent still cannot send to chat.** It holds no chat credential: that stays on the host (`config.yaml:3-5`). It *can* now read repositories with a token (§0), which is why §4.4 changes.
2. **The host cannot improvise.** Destination comes from staged evidence, never agent text (`chat-reply:64-84`). That stays. New: the host also decides *whether* a reply may go, from counts it reads itself (§4.2).
3. **The record precedes the acknowledgement.** `chat-reply` already holds the ack until `chat-route` confirms an orchestrator card (`:193-200`). The tracker item joins that step, so "Ralph says it has the work" and "the work is recorded" cannot come apart (R9).
4. **No new datastore.** Sent-once state is the marker files that exist. The thread-to-item map is two fields on the tracker item. Nothing else is added (R14).

---

## 2. Options considered

Two ways to add the tracker. Standards require at least two.

| Option | For | Against |
|---|---|---|
| **A. Extend `chat-reply` and the producer (chosen)** | Reuses the sender, the destination check, the markers and the launchd installer. One place decides whether a reply goes. | `chat-reply` is 265 lines today and would carry ack, reply, final, ceilings and the tracker step. Standards call over 800 lines a refactor candidate, so there is room. |
| B. Write the v2 `chat-writer` beside it | Cleaner single responsibility; v2's eval design applies as written. | Two host senders with one policy line. Two places to forget a ceiling. Two LaunchAgents. It re-implements what shipped three days after v2 was written. |

If `chat-reply` crosses the standards' size line during PR 5, the split is by
concern (ceilings and tracker step into their own module), not by sender.

---

## 3. The producer: the missing half

The producer is **the same job v2 described**, now with a contract it must meet
rather than one it could choose. It runs on the host as the owner (D3), calls
`spaces.messages.list`, and for each eligible message stages the evidence and
creates one intake card on the `queues` board.

### 3.1 The schema it must write

`chat-reply` and `chat-route` read, and refuse anything that differs from:

- directory `chat-inbox/chat-<sha256 of the latest message's resource name>/`, with `evidence/item.json` mode `0600`, a regular file, one link, under 64 KiB, no symlinks (`chat-reply:64-84`; `chat-route:38-48`);
- `kind: "inbound"`, `sender_trusted`, `space`, `thread` (`spaces/<s>/threads/<t>`), and `messages` oldest first, each `{name, text (list of lines), from_ralph}`;
- `prior_operations`: up to 20 earlier operation ids from the same thread (`chat-route:71-74`);
- an intake card on `queues`, assignee `ralph-chat-maintainer`, body containing exactly one `operation_id:` line, workspace `/workspaces/chat-inbox/<op>` (`chat-reply:171-172`), idempotency key = the operation id.

Two consequences. First, the idempotency key v2 proposed is now **fixed**: it is the hash of the message name, and E2 follows from it. Second, a thread's whole history is staged on every operation — the maintainer reads it and treats a later message as steering (`SOUL.md:13`) — which is how R20's answer reaches the right request without a second item (E16).

### 3.2 R10: Ralph's own messages

Replies are created with `messageId = client-ralph-<kind>-…` (`chat-reply:134`). The producer marks `from_ralph: true` for any message whose `clientAssignedMessageId` starts with `client-ralph-`, and stages **no operation** for such a message. Bot senders (`sender.type = BOT`) are skipped the same way. The agent's `ignore` decision (`SOUL.md:19`) stays as the backstop and costs one card and one model run.

**Unverified, and load-bearing:** that `spaces.messages.list` returns `clientAssignedMessageId`, `sender.type` and the mention annotations. It is documented as part of the Message resource, which I am recalling, not reading. PR 2's first act is one read-only call to confirm it. If a field is absent the design changes there, not later. This is v2's A3, unchanged in weight.

### 3.3 Eligibility, and what `sender_trusted` means now

`sender_trusted` is set `true` exactly when the message is inside the surface:

- a direct message from a human, or
- in a space, a structural @mention of the owner or of Ralph (a resolved user annotation, never the typed string), or
- (D4a, if you approve) a message from the requester who opened a request in `Waiting on requester`, in that request's own thread, with no mention.

**And the space must be in `CHAT_ROUTE_SPACES`.** That list exists and the producer reads the same line from the same instance file, so the two scripts cannot disagree about it. Eligibility is therefore an intersection: R18's rule inside the owner's list. The producer never widens it. Lifting the cap of eight is D4(c).

`SOUL.md:11` already says the host sets `sender_trusted` from "the approved intake policy". This section *is* that policy; nothing in the PRD needs a new allowlist.

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
because the credential cannot be decrypted on a second machine (§0, D3).

### 4.3 R8, what is now true and what is not

**Done:** the send asks for `REPLY_MESSAGE_OR_FAIL`; a strict send without a thread, or with a thread in another space, fails before any API call (`gchat.py:251-252,258-259`; `test_gchat_send.py:34-50`).

**Not done, and not verifiable from here:** that the service honours it for a thread id captured earlier, and how old an id may be. v2's third check — compare the returned thread to the staged one — is **not available**: `cmd_send` returns only `name` and `createTime` (`gchat.py:266-267`), and `chat-reply:post` accepts `sent is True` alone. PR 5 adds a cheap partial check: the returned message `name` must begin with the staged space. It cannot detect a wrong thread in the right space. The pre-launch check under the decisions table is what closes that gap, and the token holds no delete scope (`gchat.py:60-66`), so a wrong-thread message can be reported and not withdrawn.

### 4.4 R19: the content bound, rewritten

v2 argued that the agent *could not reach* anything outside one request's evidence directory. That is no longer so. The chat-maintainer has `terminal` and a repository token (`config.yaml:13-16`); on the personal instance that token is the owner's (`docs/decisions/0002`), and both decisions record the exposure as accepted. A reply can now quote a private repository the requester cannot open.

**So the bound moves to the host, and it is a refusal, not a trim.** Before any send of agent text — `ack`, every `reply_kind`, `final` — `chat-reply` refuses text that contains:

- a tracker reference: the instance host, an `/issues/<n>` path, or a bare `#<n>` form (R1);
- a URL not in a `reachable` list the host builds, where a URL qualifies only if it is provably public (for a GitHub pull request, one API read showing the repository is public; everything else is named in words, R19);
- a credential-shaped string (the SOUL already says never; `SOUL.md:29`).

**The honest limit, stated plainly:** prose cannot be proven to come from this request. A refusal rule catches references and links; it cannot catch a sentence that *paraphrases* a private document. Under D2 option A that residual risk is the owner's to accept; under option B it shrinks (constants carry no prose) but the `answer` and `final` replies still carry it. The ADRs already accepted this class of risk for repository access.

### 4.5 The `SKILL.md:44` policy line

v2's D2 is gone: the line is edited and names `chat-reply`. It says "acknowledgement, reply and final response" and does not mention the tracker, ceilings or `reply_kind`. PR 5 amends that sentence to match what the script then does, in the same pull request as the code, so the policy and the behaviour change together.

### 4.6 The close: the final card is the trigger

The orchestrator ends a chat graph with one `ralph-chat-maintainer` card whose parents are the work cards (`orchestrator/SOUL.md:19`). Kanban dependencies mean it does not run until every parent is done, so **one work card finishing is not the request finishing** (R11, E7) by construction, and no tracker poll is needed. `chat-reply` sends the `final`, then sets the item to `Resolved` with the outcome text. The tracker still reads as canonical (R13) because the host writes it at each stage: created at the ack, `Waiting on requester` at a `clarify`, `Resolved` at the final.

**Silence on failure (E11, A14)** follows from the graph only partly: a work card that is blocked or abandoned never lets the final card run, so nothing is sent. But a final card whose parents *completed with a failed result* is told to "say so plainly" (`SOUL.md:23`). That is D2 item 3. Under option A that message is sent; under option B the maintainer's instruction changes and `chat-reply` refuses a `final` whose card says the work failed.

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
| `kind: update` leaves no final card for `chat-reply` to send | E10 |
| exactly one `messages ( ) . create` path exists in the host scripts, and `REPLY_MESSAGE_OR_FAIL` is the only reply option used | R15, R8 |

**The gap, still open.** `.github/workflows/` holds `scope-check.yml` and `skills-public.yml`; the second runs two named evals only. None of the chat evals run in CI, so today nothing fails a build if someone adds a second send path or removes a ceiling. PR 5 ships the workflow with the code, triggered on changes to `bootstrap/chat-*`, `skills/ops/google-chat/**` and `evals/test_chat_*`. Wiring the older mail eval in is one line and an observation (§9), not done here.

### 4.8 How you see a failure

`chat-reply` signals trouble by exit code and stderr only: `1` for retry, `2` for refused (`chat-reply:160,190,198,208`). It blocks nothing and tells nobody, so a refusal — a ceiling, a R19 check, a tracker failure — is invisible, and a retryable failure repeats every 60 seconds without a word. PR 5 therefore adds the two existing mechanisms from `maintain-pr`:

1. **Block the card** through `hermes kanban block`, with a reason naming the reply kind, the thread and the cause (`maintain-pr:396`);
2. **Say so at once** with the best-effort webhook post that tolerates its own failure (`maintain-pr:301-303`, `RALPH_NOTIFY_URL` or `GCHAT_WEBHOOK_URL`).

`notifier/poller.py` stays the backstop: it polls blocked cards, recovers the reason from a `BLOCKED:` comment (`:93-94`), and announces each once (`:121-125`). E6 needs no new machinery.

---

## 5. The tracker, and the remaining unknowns

### 5.1 Unknown 1 — writing to the tracker with REST off

Unchanged from v2.

| Path | Verdict |
|---|---|
| **Enable REST, a dedicated Ralph account, a per-user API key** | **Chosen.** One setting, one account, a documented interface, validation runs. Cost: the setting is instance-wide, and the key is a credential held on the host beside the chat token (D1). |
| `rails runner` over a shell on the tracker host | Fallback if you refuse D1. Bypasses validation; a second trust boundary. |
| Incoming email | Rejected: email is a PRD non-goal. |
| Direct database writes | Rejected. |

All tracker access sits behind one module, `bootstrap/tracker.py`, with `find`, `create`, `update`, `note`; the fallback replaces one file. *General product knowledge; the instance was not inspected (A1).*

### 5.2 Where the tracker step sits

Inside `chat-reply:handle`, in the branch that already holds the acknowledgement until the orchestrator card exists (`:193-200`). The order becomes: orchestrator card, then tracker item, then reply. If the tracker write fails, nothing is sent and the operation retries. Because the thread is the request, the lookup key is a custom field holding the thread name; a message in a known thread finds its item first, and an unconfident cross-thread match becomes a new item with a `related to` relation (R12, E5, E13).

The tracker module is **optional at runtime**: with D1 unanswered, `chat-reply` runs exactly as shipped, replies and all. That keeps the tracker from gating a feature that already works, which v2's design did not.

### 5.3 What an item holds, and its statuses

Stored: the requester's display name, the space and thread names, the request text as sent (truncated), Ralph's restatement, the state and the next action (R3). Not stored: other threads, member lists, attachments. Retention is the item's lifetime; the staged copy is owner-only (`0600`) and is removed when the item closes. Nothing chat-derived enters this repository.

| Status | Means | Reaches the requester |
|---|---|---|
| `New` | recorded, not understood | nothing |
| `Waiting on requester` | a clarification is open (R5, R20) | the question, once, plus at most one follow-up |
| `In progress` | understood and accepted | the acknowledgement, once |
| `Resolved` + outcome | worked to a result | the final reply, once |
| `Rejected` / `Duplicate` / `Closed` | stopped, refused, folded in | nothing, or per D2 item 3 |

`Waiting on requester` never expires and is never swept (A18). One saved query, sorted by age, shows the pile; building it is your click.

### 5.4 Unknown 6 — who classifies

The chat-maintainer, as shipped. Its decisions stay `route`, `reply`, `ignore`; v3 adds the `kind` and `reply_kind` metadata above and nothing else. "Other" is `reply_kind: decline`. "Unclear" is `reply_kind: clarify`. `final` stays on the second card type.

---

## 6. The implementation pull requests, against `Zhachory1/ralph`

Each under 400 lines and one concern (`STANDARDS.md`). Each is inert on merge; the gate column says what makes it live.

| PR | Concern | Files | Inert because | Gate |
|---|---|---|---|---|
| 1 | Tracker client | `bootstrap/tracker.py`, `evals/test_tracker.py` | nothing imports it | D1 before any live call |
| 2 | Chat read, eligibility, schema; no cards | `bootstrap/chat-producer` (dry-run default), `evals/test_chat_producer.py` | dry-run prints, creates nothing | first act: confirm the three API fields (§3.2) |
| 3 | Producer stages evidence and creates intake cards | same, plus `bootstrap/install-chat-producer-launchd` | the LaunchAgent is not installed | D3 |
| 4 | Decision metadata | `profiles/chat-maintainer/SOUL.md`, `bootstrap/chat-route`, `evals/test_chat_route.py` | adds optional keys; unchanged behaviour without them | D2 |
| 5 | Ceilings, R19 checks, block and alert, CI workflow | `bootstrap/chat-reply`, `evals/test_chat_reply.py`, `.github/workflows/chat-reply.yml`, `skills/ops/google-chat/SKILL.md` | tightens only; refuses more, sends nothing new | D2 |
| 6 | Tracker step in `chat-reply` | `bootstrap/chat-reply`, `bootstrap/tracker.py`, evals | the tracker module is absent until D1 | D1 |
| 7 | Turn it on | `compose/README.md` runbook | documentation | D3, the pre-launch check |

Order: 5 first. It adds no behaviour and closes the most exposure, because the
shipped sender has none of the ceilings today. PRs 2 and 3 stay separate for
v2's reason: the half that reads untrusted content and the half that acts are
reviewed apart.

---

## 7. Acceptance examples, mechanism and test

| # | Mechanism | Test |
|---|---|---|
| E1 | `route` → orchestrator card → tracker item → acknowledgement, in that order | `test_chat_reply`: order; one send; a tracker failure sends nothing |
| E2 | Operation id = hash of the message name; card idempotency key = that id | `test_chat_producer`: a second tick over one message creates no second card |
| E3 | `reply_kind: decline`; no tracker call; second decline in the thread refused | `test_chat_reply`: decline makes no tracker call; second is refused |
| E4 | `client-ralph-` id and `sender.type` skip the message; `ignore` as backstop | `test_chat_producer`: both are skipped |
| E5 | Thread-keyed lookup first; else a new item with a `related to` relation | `test_chat_reply` against a fake tracker |
| E6 | `--strict-thread`; failure blocks the card and posts the webhook | `test_chat_reply`: API error blocks; refusal blocks |
| E7 | The final card depends on every work card | `test_orchestration`: one of three parents done yields no final send |
| E8 | The final card's `send` posts once, then closes the item | `test_chat_reply`: a second scan sends nothing |
| E9 | `reachable` list; other URLs refused | `test_chat_reply`: private-repo URL refused; the same outcome in words sent |
| E10 | `kind: update`: no final card requested | `test_chat_route`: update omits the final-card line |
| E11 | Blocked or abandoned work never runs the final card; D2 item 3 decides the rest | `test_chat_reply` per D2 |
| E12 | R3 fields and statuses on the item (§5.3) | `test_tracker` |
| E13 | Brief says decide and link, never stop | reviewed in PR 4; a judgement, not a branch |
| E14 | Eligibility: a DM or a resolved annotation, inside `CHAT_ROUTE_SPACES` | `test_chat_producer`: unmentioned and typed-name messages are not staged |
| E15 | `clarify`: no acknowledgement; item `Waiting on requester` | `test_chat_reply` |
| E16 | Whole thread staged; D4(a) makes the unmentioned answer eligible | `test_chat_producer`: requester's reply staged, a third party's not |
| E17 | No sweep, no reminder; no final card exists for a waiting item | `test_chat_producer`: nothing re-sent for a pending thread |
| E18 | Third `clarify` in a thread refused and blocked | `test_chat_reply` |

---

## 8. Non-goals

- **A second host sender.** `chat-reply` is the sender. There is no `chat-writer`.
- **A closure producer or any tracker poll.** The final card is the trigger.
- **A chat bot identity.** Replies come from the owner's account; a requester cannot tell Ralph's text from yours. A different consent.
- **Teaching `gchat.py` more.** It is unchanged by this design.
- **Running any of this in a container.** The chat credential forbids it.
- **Unsending**, **a retry that re-sends**, and **sweeping pending requests** (A18).
- **A rate limit**, unless D4 says otherwise.
- **Any requester-facing surface**, and any tracker content reaching a requester. Unchanged from the PRD.
- **Implementing any of this.** PR 1 starts after this document is merged.

---

## 9. Assumptions, and observations

### Assumptions

Every inference the PRD, the card and the verified source did not state.

1. **Redmine's capabilities are as generally documented** (REST off by default, a per-user key, custom fields, relations, `rails runner`). Not inspected.
2. **An API key for a dedicated tracker account can live on the host** beside the chat token, owner-only.
3. **`spaces.messages.list` returns `clientAssignedMessageId`, `sender.type` and mention annotations.** Unverified; PR 2 confirms it first (§3.2).
4. **"The thread is the request"** for counting and for item lookup. A requester who restates the ask in a new thread is a new request that R12 then links.
5. **A requester cannot tell Ralph's text from yours**, and you accept that.
6. **One host, one scan at a time per instance.** launchd does not start a second copy of a running job, and the thread lock covers the rest. I did not verify launchd's overlap behaviour against your version.
7. **The producer reads `CHAT_ROUTE_SPACES` from the same instance file** `chat-reply` reads, so they cannot diverge.
8. **`reply_kind` and `kind` are safe to add as optional metadata**, because `chat-reply` ignores unknown keys (`:90-93`) and PR 4 widens `chat-route`'s allow-list in the same change.
9. **The shipped behaviour in D2 reflects a considered decision**, not drift. The ADRs say the owner decided 0001 and 0002; I found no ADR for the chat-reply behaviour.
10. **900 seconds** is the producer's tick, matching `RALPH_POLL_SECONDS`; `chat-reply` already scans every 60 s.
11. **You will run the one live thread-reply check** before the first real send.
12. **A public GitHub repository is a sufficient proxy for "the requester can already reach it".** Anything subtler is treated as unreachable and named in words.

### Observations, out of scope

One line each. Reported, not fixed.

- `chat-route` refuses any decision metadata key outside `decision, body, ack, worker_session_id` (`chat-route:55-56`), while `chat-reply` ignores extra keys (`chat-reply:90-93`; commit `b657ee2`). A worker that adds a key such as a diagnosis note makes `chat-route` exit 2, and `chat-reply` then reports "cannot confirm orchestrator card" and returns retry (`:198`), so the acknowledgement would retry every scan and never send. Read from the code, not reproduced.
- `chat-reply` has no block, alert or log beyond stderr; a refused or failing reply is invisible (§4.8).
- If a send succeeds and the marker write then fails, `chat-reply` says "retry is safe" (`:208`). A retry reuses the same `messageId`; what the Chat service returns for a duplicate custom id I did not verify, and the retry could fail forever on it.
- No chat eval runs in CI (§4.7), and `evals/test_inbox_writer.py` is in no workflow either.
- `ralph-chat-maintainer` has a repository token (`docs/decisions/0001`) and reads untrusted chat text; the ADR names this as accepted risk. R19 now rests on host checks because of it.
