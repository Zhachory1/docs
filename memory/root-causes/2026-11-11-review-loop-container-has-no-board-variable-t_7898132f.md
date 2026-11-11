---
kind: root-cause
source: t_7898132f
profile: ralph-author
date: 2026-11-11
---

# The review-loop container is passed no board variable, so it and the notifier agree only because their two defaults happen to match

Found while grounding the board-split design against `Zhachory1/ralph` at
`8e0ea5f`.

**What.** `compose/docker-compose.yml`'s `review-loop` service sets no board
variable of any kind, and has no `env_file`. `bootstrap/run-review-loop:14`
therefore always falls back to its own `${RALPH_REVIEW_BOARD:-queues}` default.
The `notifier` service *is* given `RALPH_REVIEW_BOARD: ${RALPH_REVIEW_BOARD:-queues}`.
Setting `RALPH_REVIEW_BOARD` in an instance `.env` moves the notifier's watch
list but not the loop's cards, unless the same file is also read by the loop —
which it is not.

**Why it matters.** A board setting that looks instance-wide is honoured by one
of the two containers that use it. The two stay consistent only while both
defaults say `queues`. Any change to one default splits them, and the symptom is
a gate that reaches nobody rather than an error.

**Where it is handled.** The board-split design closes it as a side effect
(`projects/ralph-board-split/dd.md`, §6 and observation O2): every board variable
the loop reads is passed to the `review-loop` service explicitly. It is a latent
bug independent of that work, and is worth checking for in any other service that
reads a `RALPH_*` default inside a script.

Links: `projects/ralph-board-split/dd.md`, `projects/ralph-board-split/prd.md`
(`Zhachory1/docs#27`), `Zhachory1/ralph#159`.
