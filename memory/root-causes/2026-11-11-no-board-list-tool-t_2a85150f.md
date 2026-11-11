---
kind: root-cause
source: t_2a85150f
profile: ralph-orchestrator
date: 2026-11-11
---
# No agent profile has a board-wide list or count tool for kanban tasks

A Chat request asked for a count of open/active tasks on Ralph's own
`ralph`/`queues` boards. No agent profile — including the orchestrator —
has a tool that lists or counts tasks across a board.

What was checked:
- The orchestrator's directly-listed kanban tools are all per-task
  (`kanban_show`, `kanban_create`, `kanban_complete`, `kanban_block`,
  `kanban_comment`, `kanban_link`, `kanban_request_changes`,
  `kanban_request_review`, `kanban_heartbeat`, `kanban_attach*`). None
  takes a board and returns a task list or count.
- `tool_search` over the deferred catalog, queried with board/list/count
  terms, surfaced only `kanban_schedule` under the `kanban` source — no
  list/query tool exists there either.
- Calling the plausible name directly (`kanban_list`) does not reach a
  list endpoint; it fails with `kanban_link`'s own validation error
  (`both parent_id and child_id are required`), so no such tool is wired
  for this profile.
- `hermes kanban ... list --json` (documented in
  `ralph/spikes/hermes-capabilities.md`) is a host-side CLI the
  `notifier` and `review-loop` containers reach via `docker exec`; agent
  sandboxes (`terminal.backend: docker`) are separate containers with no
  path to the host CLI or the kanban DB file (confirmed: no `hermes`
  binary, no kanban DB path, in the sandbox env/filesystem).

Consequence: a "how many open tasks are on board X" question cannot be
answered by any agent today. It needs either a human with host/CLI
access, or (not yet built) a board-count capability. Routing such a
request to a work card wastes a dispatch cycle, since the worker will
hit the same gap — tell the requester directly instead.

Related: `ralph/infrastructure.md` ("Kanban tools: every general profile
...") describes *having* the per-task kanban tools, not a list/query
tool; `ralph/spikes/hermes-capabilities.md` documents the CLI-only
`list --json` the notifier uses.
