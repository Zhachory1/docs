---
id: redmine-issue-tracking-tasks
status: in review
owner: Zhachory1
kanban_card: t_021bba46
pull_requests: []
---

# Task list: staged build slices

Status: in review, v1. Design only.

> ## HUMAN APPROVAL REQUIRED
>
> Nothing in this file may start. No build card exists for any of it, and none should be
> created until the owner approves the design on card t_ea5dfb5f. Slice 0 is a human
> action on a live instance and no agent performs any part of it.

Card: t_021bba46. Covers prd.md, design.md, acceptance.md.

## Review header

**Decisions needed from you (2).**
1. Slice order. Slice 0 is yours and blocks the rest. Confirm you want to do it before
   any code is written, rather than building against a stub first.
2. Whether slice 3 (status transitions) ships at all. It depends on decision D2 in
   prd.md.

**Council dissent.** None. No critic lens ran.

**Scope rating.** Three code slices after one human slice. Each is one PR. Slice 1 is
the only one that is more than a day's work.

**Changes since last revision.** First revision.

---

## Slice 0 — instance setup (human only, no code, no PR)

Blocks everything below. None of it is reviewable code and none of it is agent work.

- Enable the REST API in Administration, Settings, API. This is instance-wide; see
  prd.md decision D4.
- ~~Create the project `Work Requests`.~~ Done manually; private project exists with one document-migration issue.
- Create the three custom fields: `ralph_source_threads` (string, multiple, filterable),
  `ralph_next_action` (string), `ralph_create_op` (string, filterable).
- Create the statuses from prd.md that do not already exist, and the workflow rows for
  the maintainer's role.
- Create the user `ralph-issue-maintainer`, a role with exactly `view_issues`,
  `add_issues`, `edit_issues`, `add_issue_notes`, `manage_issue_relations`, and a
  membership on `Work Requests` only.
- Create a scratch project `Work Requests (test)` for the acceptance tests.
- Record the numeric ids of trackers, statuses, and custom fields. They are
  instance-specific and are install options, not code.

**Done when** a `curl` with the service key lists issues in `Work Requests` and fails
on any other project.

---

## Slice 1 — the tool

One PR. No profile uses it yet, so merging it changes no behaviour.

Plausible files:
- `bootstrap/redmine-tools` — the stdio MCP server. Modelled on `bootstrap/lens-tools`:
  same bounds, same binding-file shape (`.redmine-tools.json`), same refusal to take
  identity from the model.
- `evals/test_redmine_tools.py` — the allowlist, the project refusal, the byte cap, the
  op-id lock, the journal pre-check. Against a stub transport, no live instance.

Covers acceptance A5, A6, A7.

Settle the unproven API behaviour here, against the scratch project, before the shape
is locked: items 1, 2 and 3 in design.md, "Unproven Redmine behaviour".

---

## Slice 2 — the maintainer profile and intake

One PR. This is where behaviour changes.

Plausible files:
- `profiles/issue-maintainer/SOUL.md`
- `profiles/issue-maintainer/config.yaml` — toolsets `[file]`, everything else in
  `disabled_toolsets`, `mcp_servers` naming `redmine-tools` and `lens-tools` with an
  explicit `tools.include` allowlist. The shape `profiles/chat-maintainer/config.yaml`
  already uses.
- `evals/test_profile_config.py` — extend: the maintainer holds no shell, no web, no
  board write, and its config text contains no credential name.
- The producer change that creates the intake card with the operation id as its
  idempotency key.
- A private overlay job carrying `REDMINE_URL`, `REDMINE_API_KEY`,
  `REDMINE_PROJECT_ALLOWLIST` into that profile's `.env` and no other's.

Covers acceptance A1, A2, A4.

---

## Slice 3 — results flow back

One PR, smallest of the three.

Plausible files:
- The orchestrator change: execution cards get `parents=[intake card]`, and one report
  card gets `parents=[execution cards]`.
- `evals/test_orchestration.py` — extend: the execution card sees the issue id in its
  parent handoff, and the report card does not promote early.

Covers acceptance A3.

Depends on prd.md decision D2 for whether the report run may set Resolved.

---

## After the slices, before relying on it

- One restore drill: `pg_dump -Fc` and `pg_restore` into a scratch database, plus the
  `files/` copy, plus `config/configuration.yml`. Confirm an issue with an attachment
  survives. Not an acceptance test; a drill.
- One live request end to end, watched.

---

## Explicitly not in any slice

- Chat writing. Out of scope in prd.md and it stays out.
- A second project. prd.md decision D1 defers it; reopen only on the triggers named
  there.
- A poller, watcher, or cron job for results. design.md refuses it.
- A sidecar container. design.md rejects it.
- Migrating history into Redmine.
- Giving any other profile a Redmine credential.
