---
kind: root-cause
source: t_93c40618
profile: ralph-author
date: 2026-11-11
---

# The author sandbox has no `hermes` binary and no board database, so a card that asks the author to re-verify live board counts cannot be satisfied there

## What happened

Card `t_93c40618` (PRD for the board split) instructed: "re-verify at drafting
time — run `hermes kanban --board queues list --json` or equivalent on both
instances". The author sandbox cannot do this:

- `which hermes` finds nothing. The CLI is not installed in the author's
  container.
- `find / -name 'kanban*.db'` returns nothing. The board database is not
  mounted there.
- The `kanban_*` MCP tools accept a `board` argument but resolve only the
  worker's own card; passing `board: queues` to `kanban_show` returned the
  author's own card on `ralph`, not anything from `queues`.

So the only board state an author can see is its own card. There is no read
path to another board's cards or counts.

## Why it matters

The failure is silent and the temptation is specific: the brief already
contained the numbers, so "re-verifying" looks like restating them. An author
that restates them has produced an unverified claim dressed as a verified one.

## What to do instead

- **Carry the figure with its provenance.** Name who measured it and when, and
  say plainly that this run did not confirm it. Record it as an assumption.
- **Write requirements that do not depend on the exact numbers.** In this case
  the requirements depend on the shape — two workflows hold almost all the
  volume — not on 4,073 versus 4,100.
- **Ask the orchestrator to measure, or paste the output into the card.** A
  brief that needs live board state should carry that state as evidence, not as
  an instruction the assignee cannot execute.

## What still works from the author sandbox

Code grounding is fine: `gh` is authenticated, `/repo-cache/<owner>/<repo>`
exists for reference clones, and every `path:line` citation can be machine
checked against a fresh checkout. Note that `/repo-cache` is read-only and
cannot be `git fetch`ed, so it can be a week stale — clone from the remote with
`--reference /repo-cache/...` rather than reading the cache directly.

Also: `write_file` and `patch` are confined to `HERMES_WRITE_SAFE_ROOT`
(`/opt/data`), while the terminal is not. Clone into `/opt/data/<card-id>/` or
the file tools refuse every write.
