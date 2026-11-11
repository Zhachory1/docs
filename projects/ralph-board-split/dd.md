---
id: dd-ralph-board-split
title: Review graphs move to `lenses`, maintenance to `maintain`, and the notifier stops being told which boards exist
status: draft
owner: Zhachory1
card: t_7898132f
revision: v1
date: 2026-11-11
implements: prd-ralph-board-split
prs:
  - Zhachory1/docs#28   # this design
issue: Zhachory1/ralph#159
supersedes: none
---

# Review graphs move to `lenses`, maintenance to `maintain`, and the notifier stops being told which boards exist

> **This repository is public.** No employer, person, space, message content or
> private repository appears here. The two deployments are "the work instance"
> and "the personal instance". The PRD's banner applies unchanged.

Status: **draft**. This is the design step only. The implementation plan follows
as its own pull request. Nothing is built, no board was created, and no card was
moved.

**Grounding.** Every claim about Ralph is read from a fresh clone of
`Zhachory1/ralph` at `8e0ea5f` (`main`, 2026-11-11, merge of #174). The PRD was
written against `7939bb0`; line numbers moved, so the citations below replace the
card's list. The PRD is `Zhachory1/docs#27`, merged at `054c25c`.

Three things the PRD does not name turned up in that re-read. They change the
design, so they are stated before anything else:

- **The review loop creates a third kind of card.** `run-review-loop:134-160`
  creates the librarian's weekly card on the same `$BOARD` as review and
  maintenance. It is neither. See **D1**.
- **The review-loop container is never told which board to use.**
  `compose/docker-compose.yml:248-281` passes no board variable at all, so
  `run-review-loop:14` falls back to its own `:-queues` default. The notifier
  (`:219-220`) is told. They agree today only because both defaults say
  `queues`. §6 closes that gap.
- **`bootstrap/stop:93` reports in-flight cards from `$RALPH_BOARD` only.** It
  misses review and maintenance cards today, and will keep missing them. The
  report is cosmetic; the container stop at `:108` is what actually kills work.
  Out of scope, **O3**.

---

## Review header

### Decisions needed from you: three *(budget: 350 words)*

| # | Decision | Why it is yours | Default if you say nothing |
|---|---|---|---|
| **D1** | **Where the librarian's weekly card goes.** `run-review-loop:153` creates it on the review variable's board, so this split must send it somewhere. Options: keep it on `queues` (display name **Intake**), or move it to `maintain`. `dd-ralph-librarian` put it on `queues` because "the notifier polls `RALPH_BOARD` only, so a card on `queues` never pages you". That reason stopped being true at #145, and §5 ends it completely. | It is a taste call about what **Intake** means. One card a week either way. | **Keep it on `queues`**, through `RALPH_QUEUES_BOARD`. It is the only option that changes no behaviour: the weekly idempotency key and the "a librarian run is still open" guard both read one board, and both keep their history. |
| **D2** | **The notifier will watch every board the instance has, not a list it is given** (§5). On the personal instance that adds `ralph-evals` and `default`. A blocked card on either would page you. Both are believed empty of live gates (`ralph#163`: 116 archived cards on `ralph-evals`, `default` empty on both instances). | Alerts go to you. Nobody else can say whether an unexpected page from a test board is acceptable. | **Accept it.** Enumerating is the only shape that cannot silently miss a new board, which is the failure this split would otherwise create. |
| **D3** | **The cutover needs a short drain** (§7). Switching boards while a review graph is half-finished strands that graph's verdict: the publish step looks only at the new board. The drain is three commands and about fifteen minutes of no new reviews. | It is an operations window on your instances, run by you. | **Drain.** The alternative, cutting over hot, costs one duplicate review graph per in-flight pull request and one stranded verdict, recoverable only by a manual catch-up command. |

### Council dissent *(budget: 60 words)*

None yet. This is v1 by a single author. The agent review round follows, and its
findings land in v2.

### Scope *(budget: 40 words)*

**M. Five build pull requests, about fifteen files**, plus regenerated diagrams.
The largest is the notifier change, around 150 changed lines. Nothing here needs
splitting below 400 lines. The plan card does the breakdown; this document does
not.

---

## 1. What changes *(budget: 200 words)*

Four boards, each with one owner that creates cards on it.

| Board | Display name | Who creates cards on it | Variable |
|---|---|---|---|
| `ralph` | Ralph | the orchestrator, `chat-route:84-86` | `RALPH_BOARD` |
| `queues` | Intake | `chat-producer:211`, `inbox-producer:95`, `inbox-writer:67`, and the librarian card (**D1**) | `RALPH_QUEUES_BOARD` |
| `lenses` | Lenses | `build-review-graph:161-175`, through `discover-reviews` | `RALPH_LENS_BOARD` (new) |
| `maintain` | Maintain | `maintain-pr:393` | `RALPH_MAINTAIN_BOARD` (new) |

Everything else follows from that table. Three code paths read the new
variables: the review loop (§3), the synthesis safety check (§4), and
installation (§6). One code path stops reading board variables altogether: the
notifier (§5).

`RALPH_REVIEW_BOARD` is **retired**. It names one board for two workflows, and
"review" is a card status, not a board (PRD R2). A leftover
`RALPH_REVIEW_BOARD=queues` in an operator's `.env` would otherwise cancel the
split silently, so the two scripts that used to read it print one warning line
when they still see it, then ignore it.

---

## 2. The configuration surface *(budget: 250 words)*

| Variable | Default | Read by | Set in |
|---|---|---|---|
| `RALPH_BOARD` | `ralph` | `install:11`, `stop:33`, `run-notifier` (state rename only, §5) | `.env.example:157`, compose `init`, `notifier` |
| `RALPH_QUEUES_BOARD` | `queues` | `install:348`, `run-review-loop` (librarian, **D1**) | `.env.example`, compose `init`, `review-loop` |
| `RALPH_LENS_BOARD` | `lenses` | `install`, `run-review-loop`, `lens-tools` (§4) | `.env.example`, compose `init`, `review-loop`, `hermes` |
| `RALPH_MAINTAIN_BOARD` | `maintain` | `install`, `run-review-loop` | `.env.example`, compose `init`, `review-loop` |
| `RALPH_REVIEW_BOARD` | — | nothing. Warned about and ignored | removed from compose |

**One place decides each board.** The loop passes `--board` explicitly on every
call, so the Python scripts' own defaults are only for a human running them by
hand. Those defaults move with the board they serve and stay literals, not
environment reads: `discover-reviews:371`, `build-review-graph:111` and
`publish-review:227` become `lenses`; `maintain-pr:326` becomes `maintain`. Two
readers of one value cannot drift apart if only one of them is configurable.

**Unchanged defaults:** `inbox-producer:95`, `inbox-writer:67`,
`chat-route:103`, `chat-reply:671`, `chat-producer:211` and
`tracker-sync:203,217,229` all keep `queues` (PRD R8, R9). `scope-check:176`
keeps `ralph` (PRD O4).

---

## 3. The review loop names three boards *(budget: 350 words)*

`run-review-loop:14` is one `BOARD`, used by four stages. It becomes three
variables, read once at the top:

```sh
LENS_BOARD=${RALPH_LENS_BOARD:-lenses}
MAINTAIN_BOARD=${RALPH_MAINTAIN_BOARD:-maintain}
QUEUES_BOARD=${RALPH_QUEUES_BOARD:-queues}
```

| Stage | Line today | Board after |
|---|---|---|
| list finished synthesis cards | `:62` | `$LENS_BOARD` |
| `publish-review --board` | `:73` | `$LENS_BOARD` |
| `discover-reviews --board` | `:82` | `$LENS_BOARD` |
| `maintain-pr --board` | `:99` | `$MAINTAIN_BOARD` |
| librarian card: list and create | `:148`, `:153` | `$QUEUES_BOARD` (**D1**) |

The startup line at `:56` prints all three, so a misconfigured container says so
in its first log line.

**One pass per board, not one pass that routes by profile.** The card asked
which. Neither is needed: these are already four separate processes with a
`--board` argument each. Discovery, publishing and maintenance never share a
card, a query or a lock. Routing by the pull request's assigned profile would
add a lookup, a failure mode, and a second place that knows the
board-to-workflow mapping, and would buy nothing, because no stage ever needs to
see two boards at once. A per-pass loop over boards would be worse again:
`discover-reviews` has a per-pass admission budget (`:417`), and running it twice
per pass would double spend.

**Why publishing and discovery must keep one variable.** `run-review-loop:62`
lists `ralph-review-synthesis` cards that are `done` on `$BOARD`, and discovery
refuses a head whose graph already exists on `$BOARD`
(`discover-reviews:286-299, 447-452`). Two different values would review the same
head twice and post nothing. One variable for both makes that impossible to
configure wrong. Maintenance shares no query with either, which is why it can
have its own.

**Nothing else in the loop changes.** Allowlists, admission budgets, the
`.published` file, sandbox cleanup, the knowledge refresh and every idempotency
key keep their current shape (PRD N6).

---

## 4. The synthesis safety check follows the board *(budget: 400 words)*

`lens-tools:431` refuses parent handoffs unless the calling card is
`ralph-review-synthesis` **and** sits on `queues`. The board half must now say
`lenses`, without becoming "any board" (PRD R10).

**Where the two values come from.** `aliases["board"]` is `LENS_BOARD`, rendered
from `${HERMES_KANBAN_BOARD}` in `profiles/review-synthesis/config.yaml:34` —
the board the dispatcher actually claimed the card on. It is a fact about the
card. The constant it is compared against is a fact about the instance. The
check works because those two facts come from different places; keeping that
property matters more than where the constant lives.

**Resolution, in order, inside `parent_handoffs()` only:**

1. `LENS_EXPECTED_BOARD`, added to the synthesis profile's `env:` block as
   `"${RALPH_LENS_BOARD:-lenses}"` (same form as `LENS_TOOLS_PYTHON` at
   `config.yaml:40`).
2. `RALPH_LENS_BOARD` from the process environment.
3. The built-in constant `lenses`.

A value that is set but is not a valid slug **fails the call**; it does not fall
through to the next source. Otherwise, setting the variable to junk would be a
way to reach the default.

**Not in `LENS_ALIASES`.** Every name in that table is required for every lens
tool call (`lens-tools:141-146`). A missing expected-board value would break the
four lens profiles, which never call `parent_handoffs()`. Resolving it inside the
one function that needs it keeps the blast radius at one profile.

**Belt and braces, because one step is unverified.** I could not confirm from
this sandbox whether Hermes expands `${RALPH_LENS_BOARD}` in a profile `env:`
block from the container's environment or from the profile's own `.env`
(`install:466` says a dispatched worker sees its profile's `.env`, not the
container's; `config.yaml:40` suggests container-level variables do expand).
Assumption **A2**. The design does not need the answer: `install` writes
`RALPH_LENS_BOARD=<value>` into `profiles/ralph-review-synthesis/.env` — the path
proven by the token-writing block at `install:477-500` — **and** compose passes
the variable to the `hermes` service. Either path alone is enough, and the third
fallback covers both failing.

**It fails closed.** If no source resolves and the instance uses a non-default
slug, synthesis is refused its handoffs and the card fails loudly. It never
succeeds on the wrong board.

**Also updated:** the stale `queues` mentions in `lens-tools:6` and
`profiles/review-synthesis/config.yaml:44`.

---

## 5. The notifier stops being told which boards exist *(budget: 450 words)*

`run-notifier:11` builds its board list from two variables. Two boards, maximum,
by construction. With four boards and a fifth possible, that list is a thing
someone must remember to update, and forgetting it means a gate never reaches
you.

**It enumerates instead**, the way `cleanup-sandboxes:59-68` already does:
`hermes kanban boards list --json` through `RALPH_HERMES_CONTAINER`, slugs
sorted, one poll each.

| Option | For | Against |
|---|---|---|
| **Enumerate (chosen)** | Cannot miss a board. No list to maintain. Matches the one reader that already does this, and PRD R16 | Polls boards you may not care about (**D2**). One extra command per pass |
| Explicit list variable | Predictable cost | A new fixed list, which is the bug being fixed. PRD R16 forbids it |
| Keep two variables | No change | Cannot express four boards |

**Per-board state, with no first-board exception.** Today the first board keeps
the bare `$RALPH_STATE_FILE` and every other board gets a derived name
(`run-notifier:22-24, 38, 46-48, 58`). That makes alert history depend on list
order. With an enumerated, sorted list the order can change, and a changed first
board would make a whole board's gates announce again. So every board gets
`${STATE%.json}-<slug>.json` and `${DIGEST_STAMP}-<slug>`, and the loop does a
one-time rename at startup: if the bare file exists and the per-board file for
`${RALPH_BOARD:-ralph}` does not, move it. The old `queues` file already carries
its slug and needs nothing. Nothing is announced twice, and `RALPH_BOARD` is read
for that one purpose only. This also closes PRD O5.

**When the board list cannot be read.** The pass is skipped, not guessed at, and
the last good list is cached at `/var/lib/ralph/notifier-boards.json` on the
existing `notifier-state` volume. A failed enumeration uses the cache; with no
cache, the pass logs one line and the next poll retries. No state file is written
or pruned on a skipped pass, so nothing is announced again afterwards.
`run-notifier` runs under `set -euo pipefail` (`:7`), so the enumeration must be
guarded; an unguarded `$( )` failure would kill the container.

**Archived boards.** `cleanup-sandboxes` deliberately includes them, because
their cards' checkouts are still on disk. A gate on an archived board cannot be
worked, so the notifier skips archived boards *if* `boards list --json` exposes
that flag. I could not check the field name against the installed Hermes version
from this sandbox (assumption **A3**): the implementation confirms it and says so
in its pull request, and polls every board if there is no such field.

**Cost.** Two boards become four or five: one extra `docker exec` for the
enumeration, plus two or three more per 900-second pass. Negligible against a
pass that already reads a 4,000-card board.

**Unchanged:** `notifier/approvals.py` keeps its `--board` argument and has no
board-specific behaviour (PRD R13). Only the list of boards it is called with
changes.

---

## 6. Installation and compose wiring *(budget: 300 words)*

**Four boards, one function.** `install:330-356` holds two copies of the same
"list, grep, create" block. A third and fourth copy would be the point at which
this becomes a function (`STANDARDS.md`, rule of three):

```sh
ensure_board() {  # slug, display name, description
    if hermes kanban boards list 2>/dev/null | grep -qE "^[● ]+$1\b"; then
        echo "  board $1 already exists"
    else
        hermes kanban boards create "$1" --name "$2" --description "$3" \
            --default-workdir /workspaces >/dev/null
        echo "  created board $1"
    fi
}
```

| Slug | Display name | Description |
|---|---|---|
| `$BOARD` | Ralph | Feature pipeline: PRD, design docs, PRs |
| `$QUEUES_BOARD` | Intake | Chat and email intake, and the record of review and maintenance work created before the board split |
| `$LENS_BOARD` | Lenses | Review graphs: lens cards and their synthesis. Machine-created; no human cards |
| `$MAINTAIN_BOARD` | Maintain | Pull-request maintenance rounds. Machine-created; no human cards |

**An existing board is never renamed by `install`.** The existence check skips
it, which is what preserves the **Intake** name you set by hand (PRD R3, R14).
The `--name` above applies to a fresh instance only. A second run creates nothing
and exits 0.

**Compose.** `review-loop` gets `RALPH_LENS_BOARD`, `RALPH_MAINTAIN_BOARD` and
`RALPH_QUEUES_BOARD` — it has no board variable at all today. `init` gets the
same three, so `install` creates the boards the loop will use. `hermes` gets
`RALPH_LENS_BOARD` for §4. `notifier` loses `RALPH_REVIEW_BOARD` and keeps
`RALPH_BOARD`. `.env.example` lists all four board variables with their defaults,
beside `RALPH_BOARD` at `:157` (PRD R15).

**Two wrong comments go with it** (PRD R18): `install:344-347` and
`compose/README.md:229-231` both still say the notifier polls `RALPH_BOARD` only.
It has polled more since #145, and after §5 it polls everything.

---

## 7. Cutover *(budget: 350 words)*

The review loop and the notifier run continuously. The scripts are mounted
read-only from the host checkout, so a `git pull` plus a restart moves every
container at once.

**What breaks if you cut over hot.** Board queries are per board. A graph
half-finished on `queues` is invisible to the new board, so:

- `discover-reviews:447-452` sees no graph for that head on `lenses` and builds a
  second one. Cost: one duplicate review graph per in-flight pull request.
- The old graph's verdict is never posted: `run-review-loop:62` lists finished
  synthesis cards on `$LENS_BOARD` only.
- `maintain-pr:375-382` loses its round history for in-flight pull requests, so
  round numbering restarts and a round already run can run again.

Nothing double-posts to GitHub: `publish-review`'s marker is a comment on the
pull request, checked against any account at the exact head
(`publish-review:160-191, 301-312`), so a duplicate graph's verdict is a no-op.

**The drain, per instance (D3).**

1. `docker compose stop review-loop`. Discovery stops; no new graph starts.
2. Wait for lens and synthesis cards on `queues` to finish:
   `hermes kanban --board queues list --assignee ralph-review-synthesis --status running --json`
   returns empty. A synthesis run is budgeted at 480 seconds
   (`profiles/review-synthesis/config.yaml`), so this is minutes, not hours.
3. Publish what finished during the drain, from the old board:
   `/ralph/bootstrap/publish-review --task <id> --board queues --post` for each
   `done` synthesis card not already listed in `.published`.
4. `git pull`, then `docker compose up -d` for the whole instance.

**Rollback.** Put `RALPH_LENS_BOARD=queues` and `RALPH_MAINTAIN_BOARD=queues` in
the instance `.env` and restart. New work goes back to the old board; cards
created on `lenses` and `maintain` in between stay there and are read by hand.
There is no undo for cards, by PRD N2 and A7.

**Order of deployment.** §4 and §3 must land together: the board where graphs are
created and the board `lens-tools` expects are the same value, from the same
variable. The plan puts them next to each other, and the cutover is one restart.

---

## 8. What does not change *(budget: 150 words)*

- **Chat and email.** `chat-producer:211`'s literal, `chat-reply:671`'s
  `choices=("queues", "ralph")` enum, `chat-route:103`'s defaults and
  `tracker-sync:203-229`'s lookups all stay (PRD R8, R9). The card asked for them
  to change; the merged PRD keeps intake on `queues`, so changing them would be
  work with no requirement behind it. They remain a standing constraint: two
  readers treat *which board a card is on* as the fact that decides behaviour
  (`chat-reply:520, 657`), so the `queues` slug must never be reused for
  something else. Hermes slugs are immutable, which helps.
- **Profiles.** No prompt, body, budget, retry, workspace or assignee changes
  (PRD N5).
- **The dispatcher.** It enumerates every board each tick, and
  `kanban.max_in_progress` is a host-wide cap (`install:341-344`), so two more
  boards add no concurrency and need no wiring.
- **`cleanup-sandboxes`.** Already correct (PRD R16).

---

## 9. Documentation and diagrams *(budget: 150 words)*

In scope, in the same work (PRD R17):

| File | What is wrong after the split |
|---|---|
| `profiles/PRINCIPLES.md:11` | "Board `queues` holds machine-created work". This text is prepended to every profile's instructions, so a stale copy files cards on the wrong board by instruction |
| `docs/workflows.md:54` | Sequence diagram participant "Hermes board (queues)" |
| `docs/workflows.md:142` | Librarian entry: "a card … on `queues`" (**D1**) |
| `docs/operations.md:34-35` | Blocked-card lookup, `--board queues` |
| `docs/operations.md:90` | Librarian runbook, `--board queues` (**D1**) |
| `docs/operations.md:136` | Board glossary row: two boards |
| `compose/README.md:229-231` | Two boards, and the stale notifier claim |
| `docs/diagrams/system.dot:45,128-129`, `system-flow.dot:22,53-54` | One `queues` cylinder fed by discovery and maintenance |

**The diagrams are regenerated, not left stale.** `docs/infrastructure.md:15`
documents the command, and the generated `.svg` and `.png` are committed beside
their `.dot` sources today. A stale architecture diagram is worse than none, and
the boxes are the thing that changed.

---

## 10. Tests *(budget: 250 words)*

`AGENTS.md` requires a failing test before a behaviour change. Per area:

- **Review loop.** Nothing asserts which board publishing, discovery or
  maintenance use today; the one board assertion,
  `evals/test_run_review_loop.py:276`, is about the librarian card. New tests:
  publish and discover are called with `--board lenses`, maintenance with
  `--board maintain`, and a set `RALPH_REVIEW_BOARD` produces the warning and
  changes no board. `:276` keeps passing unchanged under **D1**'s default, and is
  the test that fails first if the librarian card moves.
- **Notifier** (`notifier/test_approvals.py:631-671`): with a stubbed `docker`
  returning four boards, each board is polled once by each of the three scripts;
  each board announces from its own state file; the bare state file is renamed
  once and nothing announces twice; an enumeration failure polls nothing and
  writes nothing; a cached list is used when enumeration fails. The existing
  harness stubs `python3` and `sleep` on `PATH` and must now stub `docker` too.
- **`lens-tools`**: a new `evals/test_lens_tools.py`. Synthesis on the expected
  board is allowed; on any other board it is refused; an invalid expected-board
  value is refused, not defaulted. No eval covers `parent_handoffs()` today.
- **Install**: all four boards are created on a fresh instance, a second run
  creates nothing and exits 0, and an existing board is not renamed.
- **Profile config** (`evals/test_profile_config.py`): the synthesis profile
  carries `LENS_EXPECTED_BOARD`.

Acceptance examples 1-12 in the PRD are the integration pass, run by hand on one
instance at cutover.

---

## 11. Requirements, mapped *(budget: 120 words)*

| PRD | Where it is met |
|---|---|
| R1, R2 | §1, §6: `lenses` and `maintain` created by `install` |
| R3, R4 | §6: existing boards never renamed, `ralph` untouched |
| R5 | §7: new boards start empty; no move operation exists |
| R6 | §3: one variable for discovery, graph building and publishing |
| R7 | §3: `RALPH_MAINTAIN_BOARD`, separate from the lens board |
| R8, R9 | §2, §8: intake literals and enums unchanged |
| R10 | §4: single-board equality, fails closed |
| R11, R12, R13 | §5: enumeration, per-board state, one-time rename |
| R14 | §6: `ensure_board`, idempotent |
| R15 | §6: `.env.example` and compose |
| R16 | §5: no new fixed list anywhere |
| R17, R18 | §9, §6 |

---

## 12. Rejected *(budget: 150 words)*

- **Reusing `RALPH_REVIEW_BOARD` for the lens board.** One rename, no new
  variable. Rejected: an instance with `RALPH_REVIEW_BOARD=queues` already in its
  `.env` would keep every review card on `queues`, and the split would look done
  while nothing moved.
- **One board for review and maintenance.** Cheaper. Rejected by PRD R7: they are
  two workflows with different readers, and you asked to tell them apart.
- **A `boards.yml` holding the mapping.** Rejected by Occam: four variables in
  the two files an operator already edits, against a new file, a parser and a
  failure mode.
- **Widening `lens-tools` to a set of boards.** Rejected by PRD R10. A set makes
  the check a filter that grows; equality against one configured value does not.
- **Renaming the `queues` slug.** Hermes slugs are immutable, and three readers
  in the chat path use the slug as a decision (§8).

---

## Assumptions

- **A1.** The merged PRD's decisions are final: `lenses`, `maintain`, intake and
  email stay on `queues`, no card moves, the `ralph-evals` board is out of scope.
- **A2.** Either compose's `hermes` environment or the profile `.env` reaches the
  synthesis profile's `env:` expansion. I could not verify which from this
  sandbox, so §4 writes both and falls back to a constant.
- **A3.** `hermes kanban boards list --json` returns one object per board with a
  `slug`, as `cleanup-sandboxes:66` relies on. Whether it exposes an archived
  flag is unverified; §5 handles both answers.
- **A4.** The personal instance runs the same scripts as the work instance and
  differs only in volume (PRD A3).
- **A5.** The lens cards and their synthesis card always belong to one board,
  because `build-review-graph:161-175` creates them in one pass with the lens
  cards as parents (PRD A5).
- **A6.** A board that does not exist is not created implicitly by
  `hermes kanban create`. The cutover therefore runs `install` (or the `init`
  service) before the first card is created on a new board.
- **A7.** Nothing outside `Zhachory1/ralph` hardcodes these board names. I
  searched `bootstrap/`, `notifier/`, `compose/`, `profiles/`, `evals/` and
  `docs/` in that repository only (PRD A6).

## Observations, out of scope

- **O1.** `run-review-loop` creates the librarian card on the review board. That
  coupling is what makes **D1** necessary; it is not part of this split's
  requirements.
- **O2.** The review-loop container receives no board variable today
  (`docker-compose.yml:248-281`). It works by default agreement only. §6 fixes it
  as a side effect; it is a latent bug independent of this work.
- **O3.** `bootstrap/stop:93` reports in-flight cards from `$RALPH_BOARD` only,
  so its report misses review and maintenance work. The container stop that
  follows is unaffected. Worth its own card.
- **O4.** `run-notifier:38`'s derived state-file naming disappears with §5, which
  closes PRD O5.
- **O5.** `evals/test_chat_reply.py` and three other eval files use `"queues"` as
  an arbitrary board argument. They stay correct and are not touched.
