---
id: dd-ralph-redmine-tracker
title: How an accepted chat request becomes a tracked item and stays in sync — one script, two entry points, every write idempotent
status: approved
owner: Zhachory1
card: t_41227371
revision: v2
date: 2026-11-09
implements: prd-ralph-redmine
prs:
  - Zhachory1/docs#15
supersedes: Zhachory1/docs#7   # closed by the owner; this is part 2 of 2
---

# How an accepted chat request becomes a tracked item and stays in sync — one script, two entry points, every write idempotent

> **This repository is public.** No employer, person, space, address or message
> content appears here. The requester is "a requester" and the tracker is "the
> work tracker" (a self-hosted Redmine). The PRD's banner applies unchanged.

Status: **approved** — merged at `Zhachory1/docs#15` on 2026-11-09, on the
owner's direct instruction. Nothing is built. No chat message was read or sent,
no REST API was enabled, no credential and no work instance was touched. **This
revision is an amendment,** not a new design round: it closes the two blocking
findings from the review round and the T1 contract question. Merging it is its
approval.

**Part 2 of 2.** You closed `Zhachory1/docs#7` for having too many revisions and
split it. This half covers that document's §5 — the tracker script, what an item
holds, who classifies — plus your **D1**. The chat half is a sibling pull
request and this document does not wait on it.

**Grounding.** Every claim about Ralph is read from a fresh clone of
`Zhachory1/ralph` at `e5ce164` (2026-11-09, `main`). The closed design was
checked at `95bd4d9`; twenty-four commits landed since, and four things moved
that this half touches.

- **The chat intake producer exists, and it is not in this repository.** It is
  the private overlay cron job `chat_inbox.py`, staging the whole thread every
  two minutes (`docs/workflows.md:84,89`; installed by
  `bootstrap/install-overlay-jobs:1-25`). The closed design said no producer
  existed. **So the sender has live traffic today** (§3.2).
- **No tracker code of any kind exists.** `grep -ri redmine` finds only the two
  new architecture diagrams, where it is dashed and labelled "DD awaiting
  approval" (`docs/diagrams/system.dot:114,161`). No Redmine MCP server, no
  narrow tool surface.
- **Line numbers moved.** `bootstrap/install`'s read-only chat-inbox mount is now
  `:170-178`, the orchestrator's final-chat-card rule
  `profiles/orchestrator/SOUL.md:25`, `maintain-pr`'s block `:398` and its alert
  `:303-315`, the bootstrap mount into `hermes` `docker-compose.yml:176`.
- **`chat-reply`, `chat-route`, `inbox-producer`, `gchat.py`, `SKILL.md` and the
  whole `chat-maintainer` profile are byte-identical to `95bd4d9`**, so every
  citation this half inherits still holds.

Claims about Redmine itself are general product knowledge, and marked. Your
instance was not inspected and must not be.

---

## Review header

### Decisions needed from you: one *(budget: 200 words)*

Numbering continues the closed pull request's lineage, so `D1` still means what
you answered. New decisions are prefixed `T`, so they cannot collide with the
chat half's.

| # | Decision | Why it is yours | If you say no |
|---|---|---|---|
| **D1** | **Answered 2026-11-06: approved.** Turn the tracker's REST API on, create a dedicated Ralph account, generate its per-user API key, and put that key in the `tracker-creds` volume — not beside the chat token. What is left is operational, and it is still the whole gate: **everything here is inert until you do it.** One part is not an API call: Redmine creates custom fields through the admin interface, not over REST (general product knowledge, A1), so the thread-key field in §4 is one thing you add by hand, once. | It is a setting on your instance, and an account only you can create. | Nothing tracker-side runs. The sender keeps working exactly as today, because `tracker-sync` exits 0 "tracker not configured" with no key file (§3.2). |

**T1 is closed, and it was closed before it was asked.** v1 asked the chat half
to carry two fields and listed them as an open contract change. The merged
`dd-chat.md` already carries both, and v2 of that document states §3.1 is the
single authoritative copy of the schema:

- **(a) `sender`** is in its §3.1 item record (`dd-chat.md:276-277` at this
  revision) — "the requester, for the model and for R3. No code reads it;
  `SOUL.md:11` documents it and the agent uses it." The review round guessed
  this might already be free; it is.
- **(b) `kind`** is written on the done card's decision metadata by
  `chat-route`, which `dd-chat.md` §4.3 specifies (`dd-chat.md:443-447`) along
  with the allow-list widening it needs.

So §5 below cites those two sections instead of restating them, and nothing in
this half is an open ask of a sibling document. What is **not** closed is the
`chat-route:55-56` allow-list change `kind` depends on — it is the chat half's
pull request, named in its §4.3, and still listed in this document's
observations.

### Council dissent *(budget: 240 words)*

Six review rounds on the closed pull request produced four findings that bind
this half. All four are carried, not re-argued.

1. **`--record` must be idempotent against its own marker** (blocking, v5).
   Every other marker in the shipped sender is written `O_EXCL`
   (`chat-reply:145-147`); copying that deadlocks the acknowledgement on the
   first retry (§3.2).
2. **The lock belongs to the thread, not the operation** (blocking, v1). It is
   the chat half's lock; §3.3 says what this half depends on and what breaks.
3. **`--settle` must enumerate by walking the filesystem**, not by reusing the
   sender's `list --status done` loop, which stops looking at an operation once
   its reply marker exists (`chat-reply:239-242`). §3.3.
4. **A failure that repeats quietly is worse than one that refuses** (v5,
   out-of-scope note, taken): the alert fires on a retry as well as a refusal
   (§3.4).

**Where I disagree with the shape I was handed.** The sidecar is a **placement**
boundary, not an isolation one. The sender can `docker exec` into any container
the proxy will name (`docker-compose.yml:31`, `EXEC: 1`), so "neither service
holds both credentials" is a claim about which volume is mounted where, and
about one `grep` over two files. §2 claims that and no more; claiming more would
be claiming something `EXEC: 1` disproves.

**New, and what I would most like corrected.** The producer feeding all of this
is a private overlay file I cannot read. Everything this half consumes is
written by public code — the marker layout and the workspace path are pinned by
`chat-reply:52,171` and refused if they differ — so the mechanism does not rest
on the overlay. The *input record* does, and the only check available is that
the merged chat design says what the record holds (its §3.1). **v2 adds the part
that matters more:** that overlay runs in production and the host sender answers
requesters every sixty seconds, so this half is being designed against a live
loop, not a cold one (§3.2, Scope).

### Scope: small — three pull requests of its own *(budget: 80 words)*

Only this half's: the tracker client behind one module; `tracker-sync` with its
own eval, inert because nothing calls it; and the wiring that adds `--record`
before the acknowledgement and `--settle` once per scan, inert because
`tracker-sync` exits 0 without a key file. Shared and not claimed here: the
extracted staged-evidence reader all three scripts need (rule of three — it
exists twice already, at `chat-reply:69-74` and `chat-route:42-47`), the compose
services, the runbook. Order, file lists and sizes belong in the implementation
plan.

**What makes all three safe to merge while the chat loop is live.** Not "nothing
exists" — a producer does, and the host sender answers requesters every sixty
seconds (§3.2). All three are class 3 in `dd-chat.md`'s scope section: inert
until **D1**, because `tracker-sync` exits 0 "tracker not configured" with no
key file. The third also edits the live host sender (class 1), and is safe there
for a narrower reason: adding a `--record` call before the acknowledgement can
only **delay** a send, never add one, and the delay is bounded by the same
gate — no key file, exit 0, acknowledgement goes out unchanged. The
implementation plan states that class per pull request.

### What changed in v2 *(budget: 90 words)*

An amendment to the merged v1 (`Zhachory1/docs#15`), card `t_d306f347`, closing
the two blocking findings of the single review round
(`Zhachory1/docs#15`, comment `6745102361`):

1. **§3.3's convergence guard is now stated completely.** Detecting the race was
   not resolving it: the losing item's thread key is cleared and the item set
   `Duplicate`, and the operation's marker is written to the canonical item id.
2. **§6: unknown 10 moves to the closed list**, where `prd.md:895` already had
   it.
3. **T1 is closed** rather than asked, and §5 cites `dd-chat.md` instead of
   restating its schema.
4. **Scope says what makes these pull requests safe to merge** now the loop is
   live.

The round's non-blocking findings are untouched and still open, by the amending
card's instruction: §3.2's `chat-reply:203-209` precedent, the chat-side half of
unknown 9 (E6), the "same mechanism as R12/E5" framing (§3.3 now names the
difference, which is as far as this amendment goes), and §7's suggested trim.

---

## 1. The answer, in one page *(budget: 150 words)*

The sender already holds an acknowledgement until a second script confirms
something: it runs `chat-route` through `docker exec` and sends nothing unless
that exits 0 (`chat-reply:193-199`). The tracker is the same shape again — one
more script, called the same way, in the same place.

```
chat ─> (overlay chat_inbox.py) ─> chat-inbox/<op>/evidence/item.json + intake card
                                          ▼
                             ralph-chat-maintainer   route | reply | ignore
                                          ▼          (hermes container, no credential)
      bootstrap/chat-reply ── docker exec ──> bootstrap/tracker-sync
      (sender service, chat token)            (tracker service, key, no send call)
             ├─ ack:     chat-route card, THEN --record, THEN the reply
             ├─ clarify: the question goes out, THEN --settle records it pending
             └─ final:   the reply goes out,  THEN --settle closes the item
```

Two properties carry the rest. **The record precedes the acknowledgement, and
nothing else waits on the tracker** (R9): an outage delays a record and never
changes what a requester was told. **Every tracker write is idempotent,
including `tracker-sync`'s own markers** (§3.2). Neither adds a datastore (R14):
sent-once state is the marker files that exist, and the thread-to-item map is
one field on the item.

---

## 2. Two services, two volumes, one invocation *(budget: 170 words)*

Your **D5** put the sender in a compose service. This half takes the second half
of that answer: `tracker-sync` is its own service with its own volume.

| Service | Holds | Mounts | Why it cannot reach the other's credential |
|---|---|---|---|
| sender (producer loop + `chat-reply --scan` loop) | the Chat token | `chat-creds`, `../bootstrap:ro`, `../skills:ro`, `/workspaces` | The tracker key is in a volume it does not mount, and its source imports no tracker module. |
| `tracker-sync` | the tracker API key | `tracker-creds`, `../bootstrap:ro`, `/workspaces` | The Chat token is in a volume it does not mount, and its source contains no send call. |
| `hermes` (unchanged) | the repository token | as today (`docker-compose.yml:176`) | Neither credential volume is mounted into it, and neither is `hermes-home`. |

**How the separation holds, exactly.** By three things and not four: the two
volumes are mounted into different services; neither is mounted into the
container the model runs in, which is what
`profiles/chat-maintainer/config.yaml:3-5` means by "enforced by where the
credential lives"; and the call between them is a process invocation, so neither
file imports the other. It does **not** hold by network or namespace isolation.

**The tracker service is exec'd into, not ticked.** `docker exec` needs a
running container, so it needs an idle command and `restart: unless-stopped`,
not a command that exits the way `init`'s does (`docker-compose.yml:63`). The
wiring also lands before the compose services do, so `chat-reply` must work both
ways: `tracker-sync` locally when no tracker container is named, by `docker
exec` when one is — the shape `inbox-producer` already uses for the board
(`inbox-producer:29-33`).

---

## 3. `bootstrap/tracker-sync`: one job, two entry points

One sentence: **make the work tracker match the state of a chat operation.** It
holds the tracker key, puts every tracker call behind one module
(`bootstrap/tracker.py`: `find`, `create`, `update`, `note`), and contains no
send call and no policy about whether a reply may go.

### 3.1 The two entry points *(budget: 140 words)*

| Entry point | Called by | Does | Writes |
|---|---|---|---|
| `--record <op>` | the sender, in the branch that already holds the acknowledgement until `chat-route` confirms an orchestrator card (`chat-reply:193-199`) | finds or creates the item for the thread, with the requester, the staged request text, R4's kind and the card's `ack` as Ralph's restatement; sets `In progress` | `.tracker-recorded`, holding the item id |
| `--settle` | the sender, once at the top of each scan, the same way | for each operation with a clarification marker and no `.tracker-waiting`: write the pending record, set `Waiting on requester`. For each with a final marker and no `.tracker-closed`: set `Resolved` with the outcome from the report card | `.tracker-waiting`, `.tracker-closed` |

An acknowledgement goes: orchestrator card, `--record`, reply. A non-zero exit
sends nothing and the operation retries next scan, exactly as a failed
`chat-route` does today (`chat-reply:197-199`). Everything `--settle` does
happens **after** a reply has gone out.

**Why it reads cards.** `--record` needs the restatement and `--settle` the
outcome; both are agent text on a card, read with the listing the sender already
makes (`chat-reply:221-224`), matched by operation id. It never reads a decision
in order to act on one — by then the sender has decided. One reader of policy,
one sender, one writer of reply markers.

### 3.2 Every write is idempotent, including its own markers *(budget: 150 words)*

**The load-bearing paragraph.** The sender creates markers `O_EXCL`
(`chat-reply:145-147`): right for a send that must happen once, wrong for a step
re-run whenever anything after it fails. `--record` is such a step —
`chat-route` confirms, `--record` creates the item and its marker, the send
fails, `handle()` returns 1, and the next scan runs `--record` again. A
create-or-fail marker would make that second call find the item (fine) and then
exit non-zero recreating a marker that exists, which the sender reads as "retry
next scan", forever, on the one path R6 and R9 exist to protect.

So an existing `.tracker-recorded`, `.tracker-waiting` or `.tracker-closed` for
this operation is **success**, not an error; the item lookup is already
idempotent by the thread key (§4). **Until the key file exists**, `tracker-sync`
exits 0 with "tracker not configured": the gate is satisfied, the
acknowledgement goes out, the shipped path runs unchanged. That is what makes
these pull requests safe to merge before D1 is done — which now matters,
because the sender has live traffic.

### 3.3 How `--settle` finds its work, and the lock it depends on *(budget: 260 words)*

`--settle` **walks the `chat-inbox` directories on disk** and reads their
markers. It does not reuse the sender's `list --status done` loop, which skips
an operation once its reply marker exists (`chat-reply:239-242`): that loop
decides whether to send, and a close must stay retryable after sending is over.

The walk is well defined although the producer is not in this repository. The
sender derives its root as `<RALPH_WORKSPACES>/chat-inbox` (`chat-reply:52`),
refuses any card whose workspace is not `/workspaces/chat-inbox/<op>` (`:171`),
and keeps every marker in `root/<op>/` (`:27`, `:145-147`). So `root/*/` is the
operation set, and an overlay staging anything else is already refused.

**What this half depends on and does not own.** `--record` is serialised only
because the sender holds a per-thread `flock` across the span that calls it, and
that lock is the chat half's to build (`dd-chat.md` §4.4; it is not built
today). Without it, two messages arriving in one thread at once can both pass
the find step and create two items for one request. `tracker-sync` cannot take
the same lock — the sender already holds it — so it carries a convergence guard
instead.

**The guard, stated completely**, because detecting the race is not resolving
it. After a create, `--record` re-queries by thread key. If one item returns,
nothing happened. If more than one returns it takes **the oldest as canonical**
— the rule *every* thread-keyed lookup in this half uses, so two callers
reconciling at the same moment reach the same answer — and then does three
things, in this order:

1. **clears the thread-key custom field on every newer item** and sets that item
   `Duplicate` (§4), so a later thread-keyed lookup cannot return it at all;
2. **links it `related to` the canonical item**, which is where a person sees
   that the duplicate happened;
3. **writes, or rewrites, this operation's `.tracker-recorded` to the canonical
   item id** — never the id this particular call happened to create.

Step 1 is what stops the race recurring on every later message in the thread.
Step 3 is what stops this operation's own `--settle` pass from updating the
loser. The *other* operation's marker, written before reconciliation, is fixed
by the same rule from the other side: `--record` and `--settle` both resolve the
item by thread key at the start of every call and treat the id in a marker as
advisory, so once step 1 has run there is exactly one item any thread key can
reach, and R14's one-field map stays single-valued rather than forking.

The relation type is shared with R12 and E5; the job is not. R12 is an agent's
judgement about an ambiguous cross-thread match; this is automatic, takes the
oldest, and involves no judgement.

**The cost, stated rather than hidden.** This converges the map; it does not
prevent the duplicate. A lost race leaves one extra `Duplicate` item for a
person to see, and nothing tells a requester about it. The chat half's
per-thread lock removes even that, and remains the real fix.

### 3.4 How you see a tracker failure *(budget: 100 words)*

The sender signals by exit code and stderr only, so a tracker failure is
invisible today. The wiring reuses what `maintain-pr` has: block the card
through `hermes kanban block` (`maintain-pr:398`), and post the best-effort
alert that tolerates its own failure (`maintain-pr:303-315`, `RALPH_NOTIFY_URL`
or `GCHAT_WEBHOOK_URL`). Both fire on a **refusal and on a repeating retry
alike**, because the failure that matters most is the quiet one: a `--record`
that keeps failing holds an acknowledgement for work Ralph already has. One
alert per operation and reply kind, guarded by an `.alerted-<kind>` sentinel, so
a loop reports itself once rather than every minute.
`notifier/poller.py:121-127` stays the backstop.

---

## 4. What an item holds, and its statuses *(budget: 210 words)*

**Stored** (R3): the requester's display name, the space and thread names, the
request text as sent and truncated, Ralph's restatement, R4's kind, the state
and the next action. **Not stored:** other threads, member lists, attachments,
anything from another request. **Retention is the item's lifetime**, and nothing
chat-derived enters a repository.

**The lookup key is a custom field holding the thread name.** The thread is the
request (A5), so a message in a known thread finds its item first, and a
cross-thread follow-up that cannot be matched confidently becomes a new item
with a `related to` relation rather than a guess (R2, R12, E5, E13). That field
is also the whole thread-to-item map, which is how R14 is met without a new
store — and §3.3's reconciliation keeps it single-valued by clearing it on any
item it demotes to `Duplicate`.

| Status | Means | Reaches the requester |
|---|---|---|
| `New` | recorded, not understood | nothing |
| `Waiting on requester` | a clarification is open (R5, R20) | the question, once, plus at most one follow-up |
| `In progress` | understood, accepted, recorded | the acknowledgement, once |
| `Resolved` + a non-empty outcome | worked to a result | the final reply, once |
| `Rejected` / `Duplicate` / `Closed` | stopped, refused, folded in | nothing, or the one honest "it failed" reply your D2 keeps |

**`Resolved` plus a non-empty outcome is the only thing that counts as a worked
result**, which keeps E11's "finished badly" distinguishable from "never
finished". **A pending record is not acceptance** (R5): it is written after the
question goes out, by `--settle`, never before, and only `In progress` is gated
on a write that already succeeded. `Waiting on requester` never expires and is
never swept (A18, E17) — one saved query sorted by age shows the pile, and
building it is your click.

**The staged evidence copy** is mode `0600` under a `0700` directory, and the
agent sees it read-only (`bootstrap/install:170-178`). `--settle` removes the
operation's `evidence/` directory straight after writing `.tracker-closed`,
keeping the marker files so the thread's history still counts.

---

## 5. The input contract *(budget: 100 words)*

The chat half owns the item record and the decision metadata, and `dd-chat.md`
§3.1 is the single authoritative copy of both. This half **consumes** them and
defines nothing. The right-hand column is where each field is defined, not a
second definition of it.

| From | Field | Used for | Defined in |
|---|---|---|---|
| `evidence/item.json` | `thread`, `space` | the lookup key, and R3's "where it came from" | `dd-chat.md` §3.1 |
| `evidence/item.json` | `messages[].text` | the request text as sent | `dd-chat.md` §3.1 |
| `evidence/item.json` | `sender` | R3's "who asked" | `dd-chat.md` §3.1 |
| the done card | `ack` | Ralph's restatement | `dd-chat.md` §4.8, read at `chat-reply:168-176,182` |
| the done card | `kind` | R4's classification | `dd-chat.md` §4.3 |
| the done card | the final `body` | the outcome on `Resolved` | `dd-chat.md` §4.8, read at `chat-reply:168-176,182` |

**T1 is closed** (review header): `sender` and `kind` are both in the merged
chat design already, so neither is an ask, and neither is restated here.

**Eligibility is not read here at all.** Your **D6** makes the approved room the
whole gate, the chat half owns it, and this half records whatever that gate
admits. No second test, no tracker-side filter.

---

## 6. Unknowns settled, and unknowns left *(budget: 110 words)*

Settled here: **1** (how Ralph writes to the tracker — your D1), **2** (where
the thread-to-item map lives — one custom field, §4), **6** (who classifies —
the chat-maintainer as shipped, `SOUL.md:17-19`; "unclear" is a clarification
and "other" is a refusal, neither a sixth kind), **7** (what counts as a worked
result, §4), **8** (what is stored and for how long, §4), **14** (what a pending
request looks like and how long it lives, §4), and the tracker-side half of
**9** (how you see a failure, §3.4).

**Left to the chat design, in one line:** unknowns **4** (whether
`REPLY_MESSAGE_OR_FAIL` honours an older thread id), **5** (how the rest of
R15's bound is enforced and proven), **12** (R19's provenance check) and **13**
(how Ralph notices the requester's answer). Unknowns **3**, **10** and **11**
were already closed before this half: 3 and 10 by the code, which the merged PRD
records as "Answered by the code" for both (`prd.md:865`, `prd.md:895`), and 11
by your D6.

---

## 7. Acceptance examples: the tracker-side obligation *(budget: 70 words)*

Only the part of each example this half owns; the reply half is the chat
design's.

| # | What this half must do | Test |
|---|---|---|
| E1 | one item exists, with requester, ask, kind, state and next action, **before** the acknowledgement | `test_chat_reply`: a non-zero `--record` sends nothing. `test_tracker_sync`: the fields are set |
| E2 | a repeated message yields one item | `test_tracker_sync`: a second `--record` exits 0 and creates nothing |
| E3 | a refusal creates **no** item | `test_chat_reply`: a decline calls `tracker-sync` not at all |
| E5, E13 | thread-keyed lookup first, else a new item with a `related to` relation, never a wait on the owner | `test_tracker_sync` against a fake tracker; and a forced double create converges per §3.3 — the loser loses its thread key and is `Duplicate`, the marker holds the canonical id |
| E8, E10 | the final reply's settle pass closes the item once; an `update` is acknowledged and never closed | `test_tracker_sync`: the close is written once and stays retryable |
| E11 | `Resolved` only with a non-empty outcome, otherwise `Rejected`/`Closed` | `test_tracker_sync` |
| E12 | every open item reads cold, with no chat open | `test_tracker` for the fields |
| E15-E18 | the pending record lands after the question, one item per thread, nothing sweeps it | `test_tracker_sync`: a clarification marker yields the pending record once |

---

## 8. Non-goals *(budget: 140 words)*

- **How a chat message arrives.** Intake, eligibility, the read window, Ralph's
  own messages, the approved-room gate: the chat design owns all of it, and this
  half records what it admits.
- **How replies are bounded.** Reply kinds, per-thread ceilings, the R19 content
  refusals, `REPLY_MESSAGE_OR_FAIL`, the per-thread lock and the send switch are
  the chat half's. The lock is named here only as a dependency (§3.3), and a
  withheld reply is an input.
- **A tick of its own.** `tracker-sync` is invoked, never scheduled.
- **A closure producer or any tracker poll.** The orchestrator's final card is
  the trigger (`profiles/orchestrator/SOUL.md:25`).
- **A second sender.** `tracker-sync` contains no send call and may never
  acquire one.
- **A new datastore** (R14), and **any tracker field duplicating a marker file**.
- **Any requester-facing tracker surface:** no link, identifier, account or
  email reaches a requester (R1).
- **Choosing the tracker project or the issue layout beyond §4's fields.** R2
  makes routine project choices Ralph's, and you may correct any by hand.
- **Amending the PRD** (card `t_73bf3c94`), **enabling the REST API or touching
  your instance** (that is D1), and **implementing any of this** — the first
  pull request starts after merge.

---

## 9. Assumptions and observations *(budget: 180 words)*

Every inference the PRD, the card and the verified source did not state.

1. **Redmine's capabilities are as generally documented** — a REST toggle, a
   per-user key, custom fields, issue relations, and custom-field *creation*
   through the admin interface rather than over REST. Not inspected and must not
   be: the first eval runs against a fake, and the first live call is manual.
2. **An API key in a compose volume mounted into one service is an acceptable
   home for it.** Your D1 approves the account; where its key sits is my
   inference, and it keeps the two credentials apart (§2).
3. **The private overlay producer writes the layout the public sender
   enforces.** I cannot read `chat_inbox.py`; I rely only on what `chat-reply`
   refuses if it differs (§3.3).
4. **The sender will hold a per-thread lock across `--record`.** It does not
   today. §3.3 states the consequence, the guard that converges the map without
   it, and the one cost the guard does not remove.
5. **The thread is the request**, for lookup and counting. A requester who
   restates the ask in a new thread is a new request that R12 then links.
6. **An item's lifetime is an acceptable retention period** for the stored chat
   fields, given R1 keeps the tracker private to two parties.

### Observations, out of scope

- `docs/workflows.md:84` names `chat_inbox.py` as the intake owner and the file
  is not in this repository, so a reader of the public source cannot check how
  `sender_trusted` is set — the whole eligibility gate after D6.
- `chat-route` refuses any decision metadata key outside its allow-list
  (`chat-route:55-56`) while `chat-reply` ignores extra keys
  (`chat-reply:92-93`). Adding `kind` for T1(b) must widen that allow-list in
  the same change, or the acknowledgement retries every scan and never sends.
- The sender has no block, alert or log beyond stderr today (§3.4), and the
  overlay producer makes that gap live rather than theoretical.
- No chat eval runs in CI: `.github/workflows/` holds `scope-check.yml` and
  `skills-public.yml`, and the second runs two named evals only.
