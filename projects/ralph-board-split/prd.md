---
id: prd-ralph-board-split
title: Split `queues` into one board per workflow — review and maintenance get their own boards, chat intake keeps `queues`, and no existing card moves
status: draft
owner: Zhachory1
card: t_93c40618
revision: v1
date: 2026-11-11
prs:
  - Zhachory1/docs#27   # this PRD
issue: Zhachory1/ralph#159
supersedes: none
---

# Split `queues` into one board per workflow — review and maintenance get their own boards, chat intake keeps `queues`, and no existing card moves

> **This repository is public.** No employer, person, space, message content or
> private repository appears here. The two deployments are "the work instance"
> and "the personal instance".

Status: **draft**. This is the PRD step only. The design follows as its own
pull request, and the implementation plan after that. Nothing is built yet.

## Review header

**Decisions needed from you**

1. **The review board's name.** `review` is already the name of a per-card
   *status* — the column `kanban_request_review` moves a card into
   (`docs/spikes/hermes-capabilities.md:85`: `list --status review`, called out
   there as "a separate status!"). Naming a board `review` makes "a card in
   review on the review board" a sentence this system will produce. `lenses` is
   one alternative. **I did not pick.** Every requirement below is written
   against "the review-workflow board", so your answer changes one name and no
   requirement (R2).
2. **Whether `queues` keeps one live workflow.** Chat intake (6 cards) is the
   only producer I recommend leaving on `queues` (R8), because its board
   identity is load-bearing in four scripts. The cost: `queues` is frozen for
   review and maintenance, not frozen absolutely. If you want it strictly
   frozen, say so and the design gives chat intake its own board instead —
   that is a bigger change, for reasons in R8.
3. **Whether the email-inbox path is still live.** `bootstrap/inbox-producer`
   and `bootstrap/inbox-writer` default to `--board queues`, but no
   `ralph-inbox-drafter` lane appears in the measured counts. Retire it, or
   route it like the others? I could not tell from the code.

**Council dissent:** none yet. This is v1; the agent review round follows.

**Scope:** M. Roughly four to six build pull requests in the Ralph repository,
across about fifteen files. The design sizes them; the plan splits them.

## Who this is for

- **The owner**, who reads a board to see what needs a decision, and today
  reads one board holding four thousand cards to find six that do.
- **The producers and loops** that create and watch cards:
  `bootstrap/discover-reviews`, `bootstrap/build-review-graph`,
  `bootstrap/maintain-pr`, `bootstrap/publish-review`,
  `bootstrap/run-review-loop`, `bootstrap/chat-producer`,
  `bootstrap/chat-route`, `bootstrap/chat-reply`, `bootstrap/tracker-sync`,
  `bootstrap/run-notifier`, `bootstrap/cleanup-sandboxes`, `bootstrap/install`.

**Not for:** the lens, synthesis, maintainer and chat-maintainer profiles
themselves. Their card bodies, assignees, statuses, workspaces, budgets and
prompts do not change. Only the board a card is created on changes.

## Terms, settled by the owner (ralph#159)

- **Board**: a real Hermes board (`hermes kanban boards`), a separate card
  pool. This is the only grouping primitive Hermes has; there is no
  board-of-boards.
- **Assignee lane**: the WebUI's grouping of a board's cards by `assignee`
  (e.g. "Ralph author"). Not a separate board — a display of the existing
  `assignee` field.
- **Status**: the existing per-card lifecycle (`todo`, `ready`, `blocked`,
  `running`, `done`, `triage`, `archived`, plus the `review` column
  `kanban_request_review` moves a card into).

So **board = workflow**. There is no "queue" concept distinct from a board;
the owner rejected that framing in ralph#159.

## Current state, verified

Code cited from Ralph `main` at `7939bb0` (2026-11-10, merge of #158) and the
docs repository `main` at `01d01bc` (2026-11-10). I read every file and line
named below in a fresh checkout.

**I could not re-verify the live card counts.** The author sandbox has no
`hermes` binary and no board database on disk, and the board tools available to
me resolve only my own card. The numbers below are the owner's measurement of
2026-11-10, carried from ralph#159 and **not** independently confirmed by me.
See assumption A1 — nothing in this PRD depends on the exact figures, only on
the shape (two workflows hold almost everything).

**The density, as measured by the owner on 2026-11-10.** Work instance,
`queues`: 4,073 cards across 11 assignee lanes.

| Workflow | Assignee lanes | Cards | Share |
|---|---|---|---|
| Review | `ralph-lens-reliability` 850, `ralph-lens-mvp` 849, `ralph-lens-generalist` 848, `ralph-review-synthesis` 834, `ralph-lens-security` 79 | 3,460 | 85.0% |
| Maintenance | `ralph-maintainer` 597 | 597 | 14.7% |
| Chat intake | `ralph-chat-maintainer` 6 | 6 | 0.1% |
| Everything else | `ralph-orchestrator` 4, `ralph-author` 3, `ralph-builder` 1 | 8 | 0.2% |

Two workflows hold 4,057 of 4,073 cards — 99.6%. The named lanes sum to 4,071,
two short of the stated total; two cards sit in a lane the measurement did not
name (observation O2). Personal instance: `queues` at 610 cards, same shape at
smaller scale. The `ralph` board (feature work) sits at 27 (work) / 74
(personal) and is not part of this problem. `default` is empty on both.
`ralph-evals` exists on both with no real cards.

**Two boards exist, and `bootstrap/install` creates them.** `BOARD` at
`install:11` (`${RALPH_BOARD:-ralph}`), created at `:301-309`; `QUEUES_BOARD`
at `:319` (`${RALPH_QUEUES_BOARD:-queues}`), created at `:320-328`. Both
creations are already guarded by a `boards list | grep` existence check, so
both are idempotent today. Only `RALPH_BOARD` appears in
`compose/.env.example` (`:157`); `RALPH_REVIEW_BOARD` is defaulted inline in
`compose/docker-compose.yml:219` and `RALPH_QUEUES_BOARD` nowhere outside
`install`.

**One variable drives three different workflows.** `bootstrap/run-review-loop:14`
sets `BOARD=${RALPH_REVIEW_BOARD:-queues}` once, and passes that same value to
publishing (`:73`), discovery (`:82`) **and** maintenance (`:99`). Discovery
passes it on to `build-review-graph` (`discover-reviews:495`), which creates
every lens card and then the synthesis card on it, with the lens cards as the
synthesis card's parents (`build-review-graph:161-175`). The security lens is
created only when the staged evidence marks the change security-relevant
(`:141, 152`) — which is why its lane holds 79 cards against roughly 850 for
each of the other three.
`maintain-pr` creates maintainer cards on it (`maintain-pr:393`) and lists by
assignee on it (`:274`). One variable cannot name two boards, so review and
maintenance cannot be separated by configuration as the code stands.

**`queues` is a hardcoded default or literal in at least ten places.**
`discover-reviews:371`, `maintain-pr:326`, `publish-review:227`,
`build-review-graph:111`, `inbox-producer:95`, `inbox-writer:67`,
`chat-route:103`, `run-review-loop:14`, `run-notifier:11` (as the
`RALPH_REVIEW_BOARD` default), `install:319`. Three more are harder than a
default:

- `chat-producer:211` writes `"--board", "queues"` as a plain string literal.
  There is no variable to set.
- `chat-reply:527` declares `choices=("queues", "ralph")`. A third board name
  is rejected by argument parsing before any logic runs.
- `lens-tools:431` refuses parent handoffs unless
  `aliases["board"] == "queues"`. This is a **safety check**, not routing: on
  a new board, review synthesis would lose its parent handoffs and fail, not
  misfile.

**The notifier watches exactly two boards.** `run-notifier:11` builds its board
list from `RALPH_BOARD` and `RALPH_REVIEW_BOARD` and deduplicates — two boards,
maximum, by construction. It was one until #145 (`notifier/test_approvals.py:632-633`:
"work-instance features run on the review board (`queues`), and the notifier
only polled RALPH_BOARD, so their gates were never seen"). Per-board isolation
is deliberate and tested: the first board keeps the original state file and
digest stamp, each other board gets its own (`run-notifier:22-24, 38, 46-48, 58`),
so adding a board neither resends nor loses alerts on the boards already there.

**Two comments still describe the pre-#145 world.** `install:315-318` and
`compose/README.md:230` both say the notifier polls only `RALPH_BOARD`, "so
queue cards stay off its radar by construction". `run-notifier:11` has polled
both since #145. The comments are wrong today, independently of this work.

**Chat intake sits on a two-sided board boundary.** Intake cards are created on
`queues` (`chat-producer:206-215`), routed work is created on `ralph`
(`chat-route:84-86`, assignee `ralph-orchestrator`, from `--source-board queues`
to `--target-board ralph`, `:103-104`). Three readers then use *which board a
card is on* as the fact that decides behaviour: `chat-reply:493` scans both,
`:511` treats a `ralph` card as owing a `final` reply, `:513` treats a `queues`
card as owing `ack`/`reply`/`clarify`/`decline`; `tracker-sync:203, 217, 229`
finds the intake card with `find_card("queues", op_id)` and `:230` finds the
report card with `find_card("ralph", op_id)`.

**One reader already does the right thing.** `cleanup-sandboxes:59-65`
enumerates every board Hermes knows rather than a fixed list, archived boards
included, and `:172` deletes nothing if any board is unreadable. That is the
pattern the rest should follow.

**`ralph-evals` is a leftover, and the test that made it no longer can.**
`evals/test_orchestration.py:30-34` now builds its `Board("ralph-evals", ...)`
against a private temporary database, explicitly "a private DB the gateway
never ticks", and `evals/board.py:15-26` omits `--board` entirely when a
`db_path` is pinned. So the live `ralph-evals` board is residue from an older
fixture, and deleting it will not recreate it.

**Hermes has no move-a-card-between-boards operation.** No such command appears
in `docs/spikes/hermes-capabilities.md` or in any board call across
`bootstrap/` and `notifier/`. This is why the split is a fresh start, not a
migration.

## The outcome

Opening a board shows one workflow. The review-workflow board holds review
graphs; the maintenance board holds maintenance cards; `ralph` holds feature
work; `queues` holds chat intake and the historical record of everything the
old shared board ever carried. Gates reach the owner from whichever board they
occur on. A fresh instance comes up with the same set of boards without anyone
editing code, and no card that exists today is moved, archived or lost.

## Requirements

### Boards

- **R1.** Each instance has, in addition to `ralph` and `queues`, exactly two
  new boards: a **review-workflow board** carrying `ralph-lens-generalist`,
  `ralph-lens-reliability`, `ralph-lens-mvp`, `ralph-lens-security` and
  `ralph-review-synthesis`; and a **maintenance-workflow board** carrying
  `ralph-maintainer`. This confirms the split proposed in ralph#159: those two
  workflows are 99.6% of the measured volume, and they are the only two that
  grow without a human asking for anything.
- **R2.** The review-workflow board's slug is the owner's choice at approval of
  this PRD — `review`, `lenses`, or another name (decision 1). The maintenance
  board's slug is `maintain`; it collides with nothing. Every requirement here
  is written against the role, not the slug, so the answer changes one name in
  the design and no requirement in this document.
- **R3.** `queues` is **not** deleted, archived, emptied or renamed. Every card
  on it keeps its id, status, history, comments and attachments, and stays
  readable by `list` and `show` for historical lookups.
- **R4.** The `ralph` board's structure, assignees and purpose do not change.
- **R5.** No card is moved between boards. The new boards start empty and fill
  from new work only. Hermes has no move operation, so this is not a choice
  about effort — it is the only available shape.

### Routing: a card lands on the board that matches its workflow

- **R6.** A review graph created from this point lands on the review-workflow
  board: all five lens cards and the synthesis card, together, on one board.
  Discovery, graph building and publishing must agree on it — a verdict must be
  published from the same board the graph was created on, or reviews go
  unposted.
- **R7.** A maintenance card created from this point lands on the
  maintenance-workflow board. Review and maintenance must be separately
  configurable: one board variable can no longer drive both
  (`run-review-loop:14` feeding `:82` and `:99`).
- **R8.** **Chat intake stays on `queues`** and routed chat work stays on
  `ralph`. Not because of volume — six cards do not matter either way — but
  because three readers use the board itself as a fact:
  `chat-reply:511` and `:513` decide which reply kind an operation still owes
  from whether the card is on `ralph` or `queues`, and `tracker-sync:203-230`
  tells the intake card from the report card the same way. Moving intake to
  `ralph` collapses both sides onto one board and makes that distinction
  unexpressible; it would also put cards whose bodies are untrusted chat
  content (`chat-route:80`) on the human feature board. The honest consequence:
  `queues` is frozen for review and maintenance, and remains the live chat
  intake board. It stops growing by thousands; it does not stop growing.
- **R9.** Every board any producer writes to is settable per instance by the
  same kind of mechanism `ralph` and `queues` use today, so two instances can
  differ without a code change. Specifically, no board name may remain a bare
  string literal (`chat-producer:211`) or a closed enumeration that rejects a
  valid board (`chat-reply:527`).
- **R10.** Review synthesis keeps its parent-handoff restriction, and that
  restriction follows the review-workflow board. `lens-tools:431` must permit
  synthesis on the new board and must still refuse it everywhere else. This is
  a security boundary; widening it to "any board" is not an acceptable fix.

### Gates and notification

- **R11.** The notifier raises every gate it raises today. Its board list is
  every board that can carry a human-facing gate — the number is not fixed at
  two (`run-notifier:11`). Concretely: a lens or synthesis card blocked for a
  human decision on the review-workflow board, and a maintenance card blocked
  on the maintenance board (`maintain-pr:414`), both reach the owner.
- **R12.** Adding a board must not disturb the boards already watched. The
  existing per-board state file and digest stamp isolation
  (`run-notifier:22-24, 38, 46-48, 58`) holds: no alert is sent twice, none is
  dropped, and no board's daily digest is resent because another board's failed.
- **R13.** The GitHub-decision path (`notifier/approvals.py`: unblock, comment,
  `reopen-review`, reassign, complete, block) works on every board the notifier
  watches, with no board-specific behaviour.

### Installation

- **R14.** A fresh install creates every board the instance needs, idempotently,
  in the same shape as the existing checks at `install:301-309` and `:320-328`:
  a second run creates nothing and fails nothing. This holds on both instance
  shapes, work and personal.
- **R15.** The new board names are visible where an operator configures an
  instance — `compose/.env.example` and `compose/docker-compose.yml` — not only
  as inline defaults inside scripts. `compose/.env.example:157` lists only
  `RALPH_BOARD` today; `RALPH_REVIEW_BOARD` and `RALPH_QUEUES_BOARD` are
  invisible there.
- **R16.** Anything that reads across boards keeps reading across **all** of
  them. `cleanup-sandboxes:59-65` already enumerates every board rather than a
  list, and must keep doing so; no new fixed board list may be introduced
  anywhere. A new board must not make `cleanup-sandboxes` delete a live card's
  checkout.

### Documentation

- **R17.** Every as-built statement of "there are two boards" is corrected in
  the same work: `profiles/PRINCIPLES.md:11`, `docs/operations.md:86`,
  `docs/workflows.md:54`, `compose/README.md:229-230, 289`, and the two
  diagrams `docs/diagrams/system.dot:45` and `docs/diagrams/system-flow.dot:22`
  with their generated SVGs. An agent reads `PRINCIPLES.md` to decide where to
  file a card; leaving it stale puts cards on the wrong board by instruction.
- **R18.** The two stale notifier comments (`install:315-318`,
  `compose/README.md:230`) are corrected to match `run-notifier:11`. They are
  already wrong; this work touches both files.

## Acceptance examples

1. **Fresh install, both shapes.** On a clean instance, `bootstrap/install`
   runs and `hermes kanban boards list` shows `ralph`, `queues`, the
   review-workflow board and `maintain`. Running `bootstrap/install` a second
   time prints "already exists" for all four, creates nothing, and exits 0.
   Verified on both the work and personal instance shapes.
2. **A review graph lands on the review board.** With the review loop running,
   a newly discovered pull request produces its whole graph on the
   review-workflow board: three or four lens cards (the security lens only when
   the change is security-relevant, `build-review-graph:141, 152`) plus one
   `ralph-review-synthesis` card holding them as parents.
   `hermes kanban --board queues list --assignee ralph-lens-generalist --json`
   shows no card created after the cutover; its count is unchanged from the
   moment of the change.
3. **Synthesis completes and publishes from the new board.** The synthesis card
   on the review-workflow board reads its five parents' handoffs successfully
   (`lens-tools:431` permits it there), reaches `done`, and
   `run-review-loop:62-73` finds it and `publish-review` posts the verdict to
   the pull request. No verdict is left unposted because discovery and
   publishing disagreed about the board.
4. **Synthesis is still refused elsewhere.** A `ralph-review-synthesis` card
   created on `ralph` or on `maintain` is refused parent handoffs by
   `lens-tools`, exactly as a card on any non-review board is today.
5. **A maintenance card lands on the maintenance board.** A new maintenance
   run creates its `ralph-maintainer` card on `maintain`.
   `hermes kanban --board queues list --assignee ralph-maintainer --json` count
   is unchanged from the cutover.
6. **A gate on the new review board reaches the owner.** A lens card blocked
   for a human decision on the review-workflow board appears in the notifier's
   alert and in the next daily digest, the same as a blocked card on `queues`
   does today. Likewise for a blocked maintenance card on `maintain`.
7. **Gates on existing boards are undisturbed.** After the new boards are
   added, a blocked card on `ralph` produces exactly one alert, not two, and
   the `ralph` daily digest still goes out once. No previously announced gate
   is re-announced by the change.
8. **An owner decision from GitHub still moves a card.** Approving a gate
   through the GitHub path moves the card on the review-workflow board from
   `blocked` to `ready`, as `notifier/approvals.py` does on `queues` today.
9. **Nothing on `queues` moves.** Before and after, `hermes kanban --board
   queues list --archived --json` returns the same card ids with the same
   statuses, except for new chat intake cards (R8). A card id from the old
   review backlog still resolves with
   `hermes kanban --board queues show <id> --json`, with its comments and
   events intact.
10. **Chat intake is unchanged end to end.** A new chat message creates one
    intake card on `queues` with one `operation_id:` line, assignee
    `ralph-chat-maintainer`, idempotency key the operation id
    (`chat-producer:206-215`); `chat-route` creates the orchestrator card on
    `ralph`; `chat-reply --scan` finds both and sends the reply kind each side
    owes; `tracker-sync` resolves the intake card on `queues` and the report
    card on `ralph`. The six existing chat cards stay where they are.
11. **A board name can be changed without editing code.** Setting the review
    board's variable to a different value on one instance routes new review
    graphs there, and the other instance is unaffected. No board name remains a
    bare literal.
12. **`ralph-evals` is gone.** The owner (or ops) deletes the `ralph-evals`
    board on both instances; `hermes kanban boards list` no longer shows it,
    and a full `evals/` run afterwards passes without recreating it
    (`evals/test_orchestration.py:30-34` uses a private temporary database).
    **This is an owner/ops action, not a build card** — deleting a board is not
    a code change.
13. **Sandbox cleanup is still safe.** With four boards live, a running card's
    checkout on the new review board is not deleted by `cleanup-sandboxes`, and
    a cleanup run that cannot read one board deletes nothing (`:172`).

## Non-goals

- **N1.** No new Hermes concept for "a collection of boards". There is not one,
  and inventing a name for it would describe something that does not exist.
- **N2.** No migration of existing cards between boards.
- **N3.** No change to assignee lanes or status within a board.
- **N4.** No change to the `ralph` board's own structure.
- **N5.** No change to what any profile does on a card: prompts, bodies,
  budgets, retries, workspaces and handoffs are untouched. Only the board a
  card is created on changes.
- **N6.** No change to the review loop's discovery rules, allowlists, cost
  bounds or publishing idempotency.

## Open questions for the owner

- **The `review` name** (decision 1). `review` is already a card status. Keep
  the collision, or pick another slug.
- **Whether `queues` stays live for chat intake** (decision 2). R8 recommends
  yes, with the two-sided boundary as the reason. If you want `queues` strictly
  frozen, the only other shape is a fourth board for chat intake — which means
  changing `chat-reply:511-513` and `tracker-sync:203-230` from "which board"
  to some other discriminator. That is real design work, and I am not assuming
  you want it.
- **The email-inbox path** (decision 3). `inbox-producer:95` and
  `inbox-writer:67` default to `queues`; no `ralph-inbox-drafter` lane appears
  in the measured counts. Retire it, or route it like the rest?

## Unknowns to settle at design time

- **How the review loop names two boards instead of one.** `run-review-loop:14`
  is a single `BOARD` used for publish, discover and maintain. Two variables, a
  per-stage flag, or a small config read — the design picks.
- **Whether `chat-reply`'s enum becomes an open argument or a wider enum**, and
  what rejects an unknown board if the enum goes.
- **How `lens-tools:431` expresses "the review board"** without widening to any
  board: an environment variable the lens sandbox already trusts, or a value
  derived from the alias binding it already validates (`:436-437`).
- **How the notifier gets its board list**: enumerate every board the way
  `cleanup-sandboxes:59-65` does, or an explicit list of gate-carrying boards.
  Enumerating is self-maintaining; an explicit list is predictable in cost.
- **Cutover order on a live instance.** The review loop and notifier run
  continuously. The design says what happens to a graph half-created when the
  board changes, and whether anything must be stopped first.
- **Board display names and descriptions** for the two new boards, matching the
  shape of `install:302-305` and `:321-324`.

## Assumptions

- **A1.** The owner's 2026-11-10 counts are accurate. I could not re-verify
  them: this sandbox has no `hermes` binary and no board database. Nothing here
  depends on the exact figures, only on two workflows holding almost everything.
- **A2.** Deleting a board is an operation the owner can perform. ralph#159
  states `ralph-evals` "can be deleted"; I did not verify a command exists, and
  this PRD does not depend on one beyond acceptance 12.
- **A3.** The personal instance has the same board-touching scripts as the work
  instance, differing only in volume. Both run from the same Ralph repository.
- **A4.** The WebUI density problem is card count on one board, not total cards
  in the instance. Splitting 4,073 across three boards helps only if the view
  is per board — which the owner's own term, "the WebUI's grouping of **a
  board's** cards by `assignee`", states.
- **A5.** The lens assignees and `ralph-review-synthesis` always belong to the
  same graph and therefore the same board. `build-review-graph:161-175` creates
  them in one pass, with the lens cards as the synthesis card's parents;
  splitting them across boards would break `lens-tools`' parent handoffs.
- **A6.** No tool, dashboard or saved query outside the Ralph repository
  hardcodes `queues`. I searched `bootstrap/`, `notifier/`, `compose/`,
  `profiles/`, `evals/` and `docs/` only.
- **A7.** The cutover happens once per instance and is not reversible without a
  second cutover, since cards cannot move back either.

## Observations, out of scope

- **O1.** `install:315-318` and `compose/README.md:230` describe a notifier that
  polls only `RALPH_BOARD`. That stopped being true at #145. R18 folds the
  correction into this work because it touches both files, but the error exists
  today without it.
- **O2.** The measured lanes sum to 4,071 against a stated total of 4,073. Two
  cards sit in a lane the measurement did not name — possibly unassigned. Worth
  a look before the design assumes the lane list is complete.
- **O3.** `inbox-producer` and `inbox-writer` default to `queues` and appear in
  `compose/README.md:279-293` as a manual procedure, but no
  `ralph-inbox-drafter` cards were measured. The path may be dead code
  (decision 3).
- **O4.** `bootstrap/scope-check:176` defaults to `--board ralph` — correct for
  build pull requests, and unaffected by this split. Noted so the design does
  not "fix" it.
- **O5.** `run-notifier:38` derives the second board's state file from the
  first's (`${STATE%.json}-$board.json`). With four boards this still works,
  but the first board's file keeps a name that mentions no board. Harmless,
  mildly confusing to operate.
