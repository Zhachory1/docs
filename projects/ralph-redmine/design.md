---
id: redmine-issue-tracking-design
status: in review
owner: Zhachory1
kanban_card: t_021bba46
pull_requests:
  - https://github.com/Zhachory1/docs/pull/5
---

# Design: Redmine as the canonical issue tracker

Status: in review, v3. No build authorised. Covers [the PRD](prd.md) and nothing else.
Card: t_021bba46. Research: [public research notes](research-notes.md).
Public Ralph source read for current behaviour: /ralph/bootstrap/{inbox-producer,
inbox-writer,install,install-overlay-jobs,lens-tools}, /ralph/evals/test_profile_config.py.

## Review header

**Decisions needed from you (3, plus the 5 in prd.md).**
1. The maintainer writes to Redmine from inside the container, rather than a host-side
   writer holding the key the way inbox-writer holds the Gmail token. Justification is
   in "Why not a host-side writer". Accept or reject.
2. The orchestrator links execution cards under the intake card, and in v3 it also holds
   a narrow Redmine tool and its own Redmine account. That makes it the second key
   holder. Accept or reject; the alternative to the linking half is a poller, which this
   design refuses.
3. Three custom fields get added to the project by hand (source threads, next action,
   create op). Confirm you are willing to maintain them.

v2's decision 4, "nothing sets `In Progress`", is closed. Your comment on A3 is what
found the hole and your answer is what picked the writer: the orchestrator.

**Council dissent.** Independent review card t_1f1c1c9a reported no blockers. Your
round-1 review asked why an execution card may not hold the Redmine tool, and v2 argued
untrusted input. Your round-2 answer rejected half of that argument: a request reaches
the orchestrator only after the chat maintainer and the issue maintainer have handled it,
and "not having any agent talk to the issue tracker is stupid because no work will get
tracked." v3 takes it. The argument survives only where it is still true, which is the
executing cards, and "Why the orchestrator may, and the executing cards may not" says
where the line now falls.

**Scope rating.** Small-to-medium. Three slices, one PR each. Slice 0 is yours and
blocks everything.

**Changes since last revision (v2 to v3).** Two of your answers land here. The
orchestrator holds `redmine-tools` narrowed to read, status and note, under its own
Redmine account, and sets `In Progress` when it fans out. The client-side
`REDMINE_PROJECT_ALLOWLIST` check is deleted and `issue_create` takes a project argument
(prd.md D7). Shape, tool surface, account, flows, failure handling and install options
moved with those two; nothing else changed.

## Shape

```
Chat thread
   │  host producer (existing pattern)
   ▼
intake card ──► ralph-issue-maintainer ──► redmine-tools (full surface, maintainer key)
   │                                              │  creates the issue
   │                                              ▼
   ▼                                     Redmine issue #NNN  (canonical record)
orchestrator card ──► redmine-tools               ▲
   │                  (get, status, note,         │
   │                   orchestrator key) ─────────┤  sets In Progress
   ▼                                              │
execution cards (children)                        │
   │  completion metadata only, no Redmine access │
   ▼                                              │
report card (child) ──► ralph-issue-maintainer ───┘  writes one note
```

Hermes cards execute. The Redmine issue records. Neither drives the other.

## Where the Redmine tool runs, and why

**Decision: a stdio MCP server, `redmine-tools`, running inside the existing Hermes
container, launched from the `mcp_servers` block of each profile that holds it.**

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

Two profiles hold the tool, with different allowlists. The maintainer gets the full
surface below. The orchestrator gets three calls — `issue_get`, `issue_status_set`
restricted to `In Progress`, and `issue_note_add` — and its Redmine role cannot do the
rest either, so the narrowing is enforced twice.

- `issue_search(text?, thread_id?, status?, limit)` — read. Returns id, subject, status,
  updated_on, source threads, next action. Never returns journals in bulk.
- `issue_get(issue_id)` — read, one issue, including relations and journals.
- `issue_create(tracker, project, subject, summary, source_thread_id)` — write. `project`
  became an argument in v3, when you deleted the allowlist (prd.md D7); what bounds it is
  the account's Redmine membership, nothing in this process.
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

## Accounts and permissions

One Redmine user, `ralph-issue-maintainer`, not an administrator. Its API key is its
own per-user key, so every write is attributed to it in the journal
(research-notes.md §1).

Its role grants, on the projects it is a member of: `view_issues`, `add_issues`,
`edit_issues`, `add_issue_notes`, `manage_issue_relations`. Nothing else. Membership is
what bounds where it can write, and Redmine enforces that independent of anything in our
code.

**There is no client-side project allowlist.** v2 had one, from
`REDMINE_PROJECT_ALLOWLIST`, checked before any HTTP request left the process; you asked
for it deleted (prd.md D7) and v3 deletes it from the tool, the install options and the
overlay job. What is left is the server-side layer, and it is the better of the two: it
is enforced where the data is, it survives a bug in our code, and you administer it in
the web UI where you can see it. What it costs is the thing the second layer was for —
widening the account's membership now widens what the agent can write and nothing in our
code will notice. Membership changes become the review point.

**The orchestrator gets its own Redmine user**, `ralph-orchestrator`, whose role grants
`view_issues`, `edit_issues` and `add_issue_notes` and nothing more: no `add_issues`, no
`manage_issue_relations`. Its tool allowlist says the same thing, but the role is what
makes it true. That matters more in v3 than it did in v2: with the client-side check
gone, the server is the boundary, so a profile that may not create issues should be
unable to rather than merely disallowed. It costs a second account and a second key to
rotate, both of them slice 0 work.

Each key reaches its MCP process through that profile's `.env`, written at install time
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
bar, creates the issue in the project it judges right, with status New, writes the
summary and the thread id, sets `ralph_next_action`, and completes its card with metadata
`{redmine_issue: NNN, redmine_url: ..., op_id: ...}`. It then creates the orchestrator
card as a child of its own, which is what starts the work. The URL is in the metadata
because the acknowledgement to the requester is a human comment (prd.md D6) and a human
needs one click to get to the issue.

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

In the same run, before the first execution card can report anything, the orchestrator
sets the issue to `In Progress`. That is its whole write budget and it is the state the
rest of this flow could not express.

The orchestrator also creates one report card assigned to `ralph-issue-maintainer`, with
`parents=[the execution cards]`. The kernel promotes it when they finish. The maintainer
reads their completion handoffs, writes one note summarising what happened, updates
`ralph_next_action`, and sets Resolved if the work is finished.

Executing profiles get no Redmine tool, no Redmine key, and no Redmine account. They
write their results where they already write them: the card completion metadata the
kernel carries for free, which the orchestrator and the report card read. Nothing polls,
nothing watches a directory, nothing runs on a timer, and there is no new daemon to keep
alive.

## Who can update an issue

- `ralph-issue-maintainer`, through `redmine-tools`, within the allowlist above.
- `ralph-orchestrator`, through the same tool narrowed to `issue_get`,
  `issue_status_set` for `In Progress`, and `issue_note_add`, under its own account.
- Humans, through the Redmine web UI, with their own accounts.

That is the whole list. No executing profile and no other service.

Because humans edit the same issues, the maintainer must never assume it is the last
writer. Two rules follow, and they are enforceable in the tool, not just advice:
- It appends notes; it does not rewrite history.
- It overwrites only the fields it owns: `ralph_next_action`, `ralph_source_threads`,
  and status within its allowed transitions. The subject and description are written
  once at create and never edited again, so a human's correction survives.

**Why the orchestrator may, and the executing cards may not.** v2 refused both, on the
grounds that a card which reads untrusted text and can also write the canonical record
can be talked by that text into writing anything. You rejected that for the orchestrator,
and you were right about where the chain of custody ends. A request reaches the
orchestrator only after the chat maintainer has read the untrusted thread and the issue
maintainer has turned it into a summary. What the orchestrator reads is a card brief two
profiles have already filtered. Refusing it the tracker buys nothing and costs what you
said it costs: no work gets tracked while it runs.

The executing cards sit on the other side of that line, and the line is not seniority, it
is input. A builder opens the repository, the dependency tree, the web page named in the
issue. None of that has passed a filter, because filtering it is not possible — reading
it is the job. Containing exactly that is why `lens-tools` exists and why this design puts
the key behind a process boundary at all ("Where the Redmine tool runs, and why"). The
maintainer's surface, and the orchestrator's three calls, are small enough to review line
by line. A builder's is not.

So an execution card that needs the issue updated still does not get its own access. It
writes what it learned into its completion metadata, and the orchestrator or the report
card — which runs as `ralph-issue-maintainer` — puts it on the issue. That hop costs one
card promotion the kernel already performs.

The weak reason v2 gave, blast radius, is still true and is why this stops at two holders
rather than becoming a general permission. Each holder is another key to rotate and
another `config.yaml` whose allowlist must be re-read on every change.

**`In Progress` has an owner now.** prd.md defines it as "a Hermes card for this issue is
running", and in v2 nothing set it: the issue read New for as long as the work ran, which
is the one state a reader would most want the tracker to be honest about. The orchestrator
sets it, in the same run that creates the execution cards, and that is the only status it
sets. Resolved stays with the report card, Needs Owner with the maintainer, Closed and
Rejected with you. Of v2's three options this is the third, which your answer picked; it
is cheaper than option 1 by one card per request, and it is the second key holder option 3
warned about, now authorised.

This still depends on prd.md D2. If the maintainer may not set status at all, the
orchestrator may not either, and the issue reads New until the report card writes.

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
- A project the account is not a member of: Redmine answers 403 and it fails like any
  other permission error. There is no client-side pre-check to refuse it first; you
  deleted it (prd.md D7).
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
- The Redmine base URL, the two API keys (maintainer and orchestrator), and the numeric
  ids of the trackers, statuses, and custom fields. Numeric ids are instance-specific and
  must be resolved at startup or supplied as options, not hard-coded. No project
  allowlist: it is gone as of v3.

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
   the configured workflow, and treat a rejected transition as a configuration bug. v3
   adds a second role to this: the orchestrator's role needs the New to In Progress row,
   and nothing else, or it cannot do the one thing it holds the tool for.
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
