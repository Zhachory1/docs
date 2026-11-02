---
id: redmine-issue-tracking-design
status: in review
owner: Zhachory1
kanban_card: t_021bba46
pull_requests:
  - https://github.com/Zhachory1/docs/pull/5
---

# Design: Redmine as the canonical issue tracker

Status: in review, v2. No build authorised. Covers [the PRD](prd.md) and nothing else.
Card: t_021bba46. Research: [public research notes](research-notes.md).
Public Ralph source read for current behaviour: /ralph/bootstrap/{inbox-producer,
inbox-writer,install,install-overlay-jobs,lens-tools}, /ralph/evals/test_profile_config.py.

## Review header

**Decisions needed from you (4, plus the 7 in prd.md).**
1. The maintainer writes to Redmine from inside the container, rather than a host-side
   writer holding the key the way inbox-writer holds the Gmail token. Justification is
   in "Why not a host-side writer". Accept or reject.
2. The orchestrator links execution cards under the intake card. That makes the
   orchestrator part of this flow. If you want the orchestrator untouched, say so and
   the alternative is a poller, which this design refuses.
3. Three custom fields get added to the project by hand (source threads, next action,
   create op). Confirm you are willing to maintain them.
4. Nothing in this design sets the `In Progress` status that prd.md defines, so an issue
   reads New for as long as the work runs. Your review comment on A3 is what found it.
   Pick a writer for that status; the three options are in "Who can update an issue".

**Council dissent.** Independent review card t_1f1c1c9a reported no blockers. Your
round-1 review requested changes. The comment that lands on this document asked why an
execution card may not hold the Redmine tool; the answer was in here only by implication,
and chasing it down surfaced decision 4 above, which is a real hole rather than a missing
cross-reference.

**Scope rating.** Small-to-medium. Three slices, one PR each. Slice 0 is yours and
blocks everything.

**Changes since last revision (v1 to v2).** "Who can update an issue" gained the argument
for why neither the orchestrator nor an executing card holds a Redmine tool, and names
`In Progress` as unowned. Nothing else changed; your other two comments became D6 and D7
in prd.md.

## Shape

```
Chat thread ──(host producer, existing pattern)──► intake card ──► ralph-issue-maintainer
                                                        │               │
                                                        │               └─ redmine-tools (MCP stdio,
                                                        │                   holds the API key)
                                                        │                        │
                                                        ▼                        ▼
                                          orchestrator fans out          Redmine issue #NNN
                                          execution cards as children         (canonical)
                                                        │                        ▲
                                                        ▼                        │
                                              report card (child) ──► maintainer writes one note
```

Hermes cards execute. The Redmine issue records. Neither drives the other.

## Where the Redmine tool runs, and why

**Decision: a stdio MCP server, `redmine-tools`, running inside the existing Hermes
container, launched by the maintainer profile's `mcp_servers` block.**

This is not a new pattern for this stack. `/ralph/bootstrap/lens-tools` is already
exactly this: a stdio MCP server in the existing container that stands between an agent
reading untrusted content and a resource it must not be able to abuse, with every
native toolset disabled and an explicit tool allowlist
(/ralph/evals/test_profile_config.py, `test_the_chat_maintainer_reads_untrusted_chat_
and_holds_nothing_else`). `redmine-tools` is a sibling of it, not a new idea.

The four candidates, scored on one question: can the model see or use the API key?

| Option | Key visible to model | Key isolated from in-process code | New container |
|---|---|---|---|
| Direct HTTP from the model's shell | Yes, by construction | No | No |
| Hermes plugin | No | **No** | No |
| **MCP stdio in the existing container** | **No** | **Yes** | **No** |
| Separate sidecar service | No | Yes | Yes |

Direct HTTP is out: the key must be passed into the sandbox for the model's own code to
use it, and Hermes' published threat model says that is the weakest isolation it offers
(research-notes.md §3, SECURITY.md §2.3).

A plugin keeps the key out of the model's context, but Hermes' SECURITY.md is explicit
that "any component running inside the agent process (skills, plugins, hook handlers)
can read whatever the agent itself can read, including in-memory credentials", and that
the mitigation is operator review, not isolation (research-notes.md §3). A plugin is a
context boundary, not a process boundary.

A stdio MCP server is a separate OS process. Hermes passes it only the `env:` entries
its config declares; the model sees the tool's JSON return value and nothing else
(research-notes.md §3). That is the boundary we want, and it costs no new container,
because the subprocess runs inside the Hermes container that already exists.

A sidecar buys nothing beyond this and adds a service to deploy, monitor, back up, and
upgrade. Rejected.

**Why not a host-side writer.** `/ralph/bootstrap/inbox-writer` puts the Gmail token on
the host and performs the write there, so the agent that read untrusted mail never holds
it. That is the right call for Gmail, because a send is irreversible, externally visible,
and no OAuth scope can forbid it; the guarantee has to live in code the agent cannot
reach. Redmine is different on all three counts: it is local, every write is reversible,
and every write is recorded in the issue journal under the service account's name. The
scoping that Gmail cannot give us, Redmine gives us directly through a per-project role
(research-notes.md §1). So the key can sit one boundary closer to the agent, in a
process the agent still cannot read, and we keep the operational simplicity.

If you reject this, the fallback is the inbox-writer shape: the maintainer emits a
decision in card metadata and a host-side `redmine-writer` applies it. Same flows, one
extra hop, a host cron to run it, and nothing else changes.

## Tool surface

The allowlist is the permission. Anything not listed does not exist for this profile.

- `issue_search(text?, thread_id?, status?, limit)` — read. Returns id, subject, status,
  updated_on, source threads, next action. Never returns journals in bulk.
- `issue_get(issue_id)` — read, one issue, including relations and journals.
- `issue_create(tracker, subject, summary, source_thread_id)` — write. Project is not an
  argument; see "Account and allowlist".
- `issue_note_add(issue_id, note, next_action?)` — write, append only.
- `issue_status_set(issue_id, status)` — write, restricted to the transitions in
  prd.md. `Closed` and `Rejected` are not accepted values.
- `issue_relation_add(issue_id, other_issue_id, relation_type)` — write, restricted to
  `relates` and `duplicates`.
- `issue_reference_add(issue_id, kind, value)` — write, `kind` in {card, thread, github}.

Not present, and therefore not possible: delete, project create, membership change, user
admin, attachment upload, setting `is_private`, editing a subject or description after
create, any endpoint outside `/issues` and `/search`.

**The operation id is not a tool argument.** It comes from a workspace binding file,
`.redmine-tools.json`, exactly as lens-tools reads `.lens-tools.json` and takes its task
id, run id, and claim lock from the environment rather than from the model. The model
cannot forge, reuse, or omit it.

## Account and allowlist

One Redmine user, `ralph-issue-maintainer`, not an administrator. Its API key is its
own per-user key, so every write is attributed to it in the journal
(research-notes.md §1).

Its role grants, on the allowlisted project only: `view_issues`, `add_issues`,
`edit_issues`, `add_issue_notes`, `manage_issue_relations`. Nothing else. It is a member
of no other project, so Redmine itself rejects a write anywhere else, independent of
anything in our code.

The project allowlist is also enforced in the tool layer, from
`REDMINE_PROJECT_ALLOWLIST`, before any HTTP request leaves the process. Two layers,
because the server-side one depends on a membership a human could widen by accident,
and the client-side one depends on code review. Neither is sufficient alone.

The key reaches the MCP process through the profile's `.env`, written at install time
with mode 600 by the existing overlay-job mechanism (`/ralph/bootstrap/install-overlay-
jobs`, which writes named options into one profile's `.env` and no other's). It is never
on a command line, never in a card body, never in a log line.

## Identity and idempotency

Three custom fields on the project (created by hand; Redmine custom fields are
admin-managed, research-notes.md §2):

- `ralph_source_threads` — string, multiple, filterable. Every thread that has fed this
  issue.
- `ralph_next_action` — string. What happens next, in one line.
- `ralph_create_op` — string, filterable. The operation id of the event that created
  this issue.

**The operation id** has the shape the inbox producer already uses: thread id plus the
id of the message that triggered the event (`/ralph/bootstrap/inbox-producer`). It pins
one event. A new message in the same thread is a new operation; a rerun over an
unchanged thread is the same operation.

**Card duplication** is already prevented: the producer passes the operation id as the
card's `--idempotency-key`, so a rerun returns the existing card instead of making a
second one (same mechanism as inbox-producer).

**Issue duplication** is prevented in the destination's own state, not in a local ledger
that can be lost. This copies inbox-writer, which checks Gmail for a draft carrying the
operation id rather than keeping a record of what it sent:

1. The tool takes an exclusive file lock on the operation id, making check-then-write
   atomic against a second run.
2. `issue_create` first queries `cf_<ralph_create_op>` for this operation id, with
   `status_id=*` so closed issues are included, and compares the returned value exactly
   before accepting the match. A hit returns that issue and performs no write.
3. `issue_note_add` writes every note with a first line of `ralph-op: <operation id>`.
   Before writing, it reads the issue's journals and refuses if that marker is already
   present. A retry after a lost response is a no-op, not a second note.

No local state file. Nothing to resync after a restore.

## Flows

**New request.** Producer stages the event and creates the intake card. The maintainer
searches by thread id, finds nothing, searches by text, finds nothing above the match
bar, creates the issue with status New, writes the summary and the thread id, sets
`ralph_next_action`, and completes its card with metadata `{redmine_issue: NNN,
op_id: ...}`.

**Same thread, repeat message.** New operation id, new intake card. The maintainer
queries `cf_ralph_source_threads` for the thread id, gets exactly one issue, and appends
a note. No new issue, no relation. The thread id is already on the issue, so nothing is
added to that field.

**Different thread, same request.** The thread id query returns nothing. The maintainer
falls back to text search across the project, open and closed (the Issues resource
supports `status_id=*`; research-notes.md §1). One candidate clears the match bar: the
maintainer appends the thread id to `ralph_source_threads`, appends a note naming the
new thread, and does not create an issue. This is the behaviour the whole design exists
for, and acceptance test A2 is exactly it.

**Ambiguous.** Two or more candidates clear the bar, or one is close but the maintainer
cannot tell. It writes nothing to any candidate. It sets no status on any candidate. It
completes its card with `{ambiguous: true, candidates: [...]}` and blocks for a human.
You answer by naming the issue; the maintainer's next run applies it under the same
operation id, so the pending work is not lost and cannot be applied twice.

The match bar is a judgment, and the design does not pretend otherwise. The rule is: a
candidate clears only if the maintainer can name the shared request in one sentence
without using the words "probably" or "maybe". Everything else is ambiguous. Erring
toward ambiguous costs one question; erring toward confident costs a split history.

## How other profiles' results reach the issue

**No poller.** The dispatcher already does this job.

The orchestrator creates execution cards with `parents=[intake card]`. The kernel holds
each one in `todo` until the intake card is done, then promotes it with the intake
handoff attached, which is where the execution card learns `#NNN`. That is how it can
reference the issue id in its own work, and acceptance test A3 checks it.

The orchestrator also creates one report card assigned to `ralph-issue-maintainer`, with
`parents=[the execution cards]`. The kernel promotes it when they finish. The maintainer
reads their completion handoffs, writes one note summarising what happened, updates
`ralph_next_action`, and sets Resolved if the work is finished.

Executing profiles get no Redmine tool, no Redmine key, and no Redmine account. They
write their results where they already write them: the card completion metadata the
kernel carries for free. Nothing polls, nothing watches a directory, nothing runs on a
timer, and there is no new daemon to keep alive.

## Who can update an issue

- `ralph-issue-maintainer`, through `redmine-tools`, within the allowlist above.
- Humans, through the Redmine web UI, with their own accounts.

That is the whole list. No other profile and no other service.

Because humans edit the same issues, the maintainer must never assume it is the last
writer. Two rules follow, and they are enforceable in the tool, not just advice:
- It appends notes; it does not rewrite history.
- It overwrites only the fields it owns: `ralph_next_action`, `ralph_source_threads`,
  and status within its allowed transitions. The subject and description are written
  once at create and never edited again, so a human's correction survives.

**Why not the orchestrator, and why not the executing cards.** Two reasons; the second
is the one that decides it.

The weak reason is blast radius. Every profile holding `redmine-tools` is another process
the key is trusted to and another `config.yaml` whose allowlist must be re-reviewed on
every change. That argues for few holders, not for exactly one.

The strong reason is that executing cards read untrusted input: repositories, web pages,
and the text of the request itself. A card that can both read untrusted text and write
the canonical record can be talked by that text into writing whatever it says. Containing
exactly that is why `lens-tools` exists, and why this design puts the key behind a
process boundary at all ("Where the Redmine tool runs, and why"). The maintainer's
surface is small enough to review line by line. A builder's is not.

So an execution card that needs the issue updated does not get its own access. It writes
what it learned into its completion metadata, and the report card — which runs as
`ralph-issue-maintainer` and does hold `redmine-tools` — reads that and writes one note.
The orchestrator gets nothing for the same reason, plus one of its own: it has no
judgment to add about an issue it never read.

**What that argument leaves unowned: `In Progress`.** prd.md defines the status as "a
Hermes card for this issue is running", and nothing in this design sets it. The intake
card creates the issue as New. The report card writes the result and may set Resolved.
Between the two, the issue reads New while the work runs, which is the one state a reader
would most want the tracker to be honest about. Three ways to close it, none chosen here:

1. The orchestrator creates one short card assigned to `ralph-issue-maintainer` alongside
   the execution cards, which sets `In Progress` and does nothing else. One extra card
   per request, no new holder of the key, and the single-writer rule above survives
   intact. It is the only option consistent with the rest of this document.
2. Drop `In Progress` from the agent flow. The issue stays New until the report card
   writes, and "is this running" is answered by the board, which already knows. Costs the
   status filter that prd.md D2 calls most of the point of using a tracker.
3. Give the orchestrator `redmine-tools` with `issue_status_set` and nothing else. Fewest
   cards, but a second key holder, a second profile config to re-review on every change,
   and it lands on the one profile that touches every card on the board.

This depends on prd.md D2. If the maintainer may not set status at all, option 2 is the
only one left standing.

## Failure and audit

Every tool call writes one audit line inside the container: timestamp, tool name,
operation id, issue id, project, HTTP status, outcome. Arguments are **not** logged;
the note body can contain the summary of a private request, and an audit log is a
second copy of it. The log answers "what did the agent do", not "what did it say".

Failure handling:
- HTTP 401/403: fail the tool call with a clear message, do not retry, do not fall back
  to another credential. A permission error is a configuration bug, not a transient.
- HTTP 422: return Redmine's validation errors verbatim to the model. These are
  actionable, for example a required custom field that is missing.
- 5xx or connection refused: retry three times with backoff, then fail the card. The
  intake card retries under the same operation id, so no duplicate is possible.
- Project outside the allowlist: refuse before the request is made, and log the refusal.
- Redmine API disabled: every call fails identically; the card blocks with that reason.

The durable audit trail is Redmine's own journal. Every write is attributed to the
service account, timestamped, and diffable in the web UI, with no cooperation from our
code required.

## Backup and recovery

Two parts, because one is not enough (research-notes.md §4):

1. `pg_dump -Fc` of the Redmine database, restored with `pg_restore`.
2. A file copy of Redmine's `files/` directory, where attachments live. `pg_dump` does
   not cover it, and a database-only restore leaves issues pointing at attachments that
   no longer exist.

Also copy `config/configuration.yml`, because it records a non-default
`attachments_storage_path`, and without it a restore can put the files back in the wrong
place.

Recovery has no agent-side step. There is no local ledger to reconcile, by design: the
idempotency anchors are the issue's own custom field and journals, so a restored
database restores the dedupe state with it. A card that was mid-flight at the time of
the dump re-runs under its old operation id and finds its own work already present.

Restore is not proven until it is drilled. One drill into a scratch database, once,
before this is relied on.

## Public code versus per-instance setup

Public, lives in the repository, no secrets, reviewable by anyone:
- `bootstrap/redmine-tools` (the MCP server), its eval file, the maintainer profile's
  `SOUL.md` and `config.yaml`, and the documented names of the three custom fields.

Per-instance, never committed:
- The Redmine base URL, the API key, the project allowlist, and the numeric ids of the
  trackers, statuses, and custom fields. Numeric ids are instance-specific and must be
  resolved at startup or supplied as options, not hard-coded.

These travel through the mechanism that already exists for this: a private overlay job
naming its `OPTIONS`, whose values are read from the instance env file and written into
one profile's `.env` at mode 600 (`/ralph/bootstrap/install-overlay-jobs`). Option names
are printed; values never are.

## Unproven Redmine behaviour

Call these out before building. Each one is cheap to settle once slice 0 lands, and
expensive to discover in slice 2.

1. **Exact-match filtering on a string custom field.** The documented operator for
   custom fields is `~` substring matching (research-notes.md §2). Whether `cf_N=value`
   is an exact match is not established. Mitigated by comparing the returned field value
   in the tool, so a substring hit cannot be accepted as a match, but the query shape
   needs confirming.
2. **Filtering on a `multiple=true` custom field** (`ralph_source_threads`). Not
   documented in the pages read. If it does not filter, the fallback is a single-value
   field holding only the originating thread and keeping the rest in notes, which costs
   one extra search.
3. **No uniqueness constraint on custom field values.** Redmine is not documented to
   reject a second issue carrying the same `ralph_create_op`. Our dedupe therefore rests
   on the file lock plus the pre-check, not on the database. Verify the lock holds under
   two concurrent runs.
4. **Workflow transitions.** The workflow matrix is per role and tracker, and the error
   shape when the API attempts a disallowed transition was not established
   (research-notes.md, "What I could not establish"). Build the status allowlist to match
   the configured workflow, and treat a rejected transition as a configuration bug.
5. **The Search resource is marked Alpha** (research-notes.md §1). Cross-thread matching
   should prefer the Issues resource with filters, which is marked Stable, and use
   `/search` only as a fallback.
6. **Issue Relations are marked Alpha-era**, and there is no batch relation endpoint
   (open request #29272, not shipped). One relation per call.
7. **Personal access tokens versus the classic API key.** Which Redmine version
   introduced tokens was not established. Design for the classic per-user key, which is
   documented back to 1.1.0, and switch if the instance turns out to support tokens.
8. **REST is disabled on the instance today.** The owner created `Work Requests` and
   one manual issue, but nothing in this API design has been exercised against it.
   Everything requiring REST remains unproven in practice.
