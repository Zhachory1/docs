---
kind: root-cause
source: t_f0a3aa2b
profile: ralph-author
date: 2026-11-09
---
# The review loop's eval harness stubs `docker` as `exit 0`, so every periodic block guarded by a container check is silently untested

## What

`evals/test_run_review_loop.py` runs the real `bootstrap/run-review-loop` with
stub `gh`, `python3`, `docker` and `sleep` on `PATH`. Two properties of that
harness mean its periodic blocks never execute, and nothing reports it:

- The `docker` stub is `#!/bin/sh\nexit 0\n`. The knowledge-refresh block gates
  on `[ "$(docker inspect -f '{{.State.Running}}' "$kc")" = true ]`, and the
  stub prints nothing, so the comparison is always false. The block is skipped
  in every test in the file.
- No test sets `RALPH_REVIEW_WORK`, so `$WORK` keeps its default under
  `/workspaces/…`. Stamp files (`.cleanup-stamp`, `.knowledge-stamp`) are
  written and read outside the test's temp directory. The script runs with
  `set -uo pipefail` and no `-e`, so a failed `touch` does not fail the test.

Together these are why the file's seven passing tests all cover startup
(allowlist parsing, identity refusal, the maintenance search) and none covers a
periodic block.

## Why it matters

Any new work in a periodic block of that loop — a stamp, a container call, a
head comparison — needs three harness changes before a test of it can fail for
the right reason: stub `docker inspect` to print `true`, point
`RALPH_REVIEW_WORK` at the temp directory, and record the `docker exec` call to
a file. Writing the test without them produces a green test that asserts
nothing, which is worse than no test.

Found while planning new blocks for that loop; not a bug in shipped behaviour,
and the blocks themselves work in production. The gap is in the test harness
only.

## Links

- `bootstrap/run-review-loop`, the periodic blocks after the maintenance search
- `evals/test_run_review_loop.py`, `StartupTests.run_loop`
- `plan-ralph-librarian`, PR 2 and PR 3 test plans, which carry the three fixes
