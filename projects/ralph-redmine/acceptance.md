---
id: redmine-issue-tracking-acceptance
status: in review
owner: Zhachory1
kanban_card: t_021bba46
pull_requests:
  - https://github.com/Zhachory1/docs/pull/5
---

# Acceptance tests: Redmine as the canonical issue tracker

Status: in review, v3. Nothing here has been run. No build authorised.
Covers [the PRD](prd.md) and [design](design.md). Card: t_021bba46.

## Review header

**Decisions needed from you (2).**
1. A1 to A8 run against a scratch Redmine project, not the real one. Confirm you will
   create a `Work Requests (test)` project for this, or accept that nothing is proven
   before first live use.
2. A4 needs a human in the loop by definition. Confirm you will answer one test prompt.

v2's other two were your round-1 comments, and your round-2 answers closed both. They are
tests again rather than questions: A1 asserts the acknowledgement path you named, A6
asserts the server refusing instead of the client.

**Council dissent.** Independent review card t_1f1c1c9a reported no blockers. All three
of your round-1 comments landed on this file and all three are now answered by you, not
deferred by me. One of them cost this file a test it did not have: A8 exists because
deleting the client-side allowlist moves the whole boundary onto Redmine roles, and a
boundary nothing tests is a claim.

**Scope rating.** Eight tests. A1 to A3 are one end-to-end run. A5 to A7 are tool-layer
tests and need no Redmine round trip beyond a stub. A8 needs a live instance, because
what it tests is a Redmine role.

**Changes since last revision (v2 to v3).** A1 drops the open question and asserts the
issue URL in the intake card's metadata, which is what a human needs to write the
acknowledgement (prd.md D6). A3 asserts `In Progress` while the execution card runs, set
by the orchestrator, and its "Open" note is gone. A6 is rewritten: no client-side
allowlist survives, so it asserts Redmine's 403. A8 is new, covering the orchestrator's
Redmine role.

## How to read these

Each test names what is set up, what is done, and what must be true after. "Must not"
lines are the ones worth writing first; they are the failures that would look like
success from the outside.

A test that needs a live Redmine runs against a scratch project. None of them run
against the real Work Requests project.

---

## A1. One request creates one issue

**Setup.** Empty scratch project. No issues.

**Do.** Feed one request event, thread `T1`, message `M1`.

**Must be true.**
- Exactly one issue exists.
- Its status is New.
- `ralph_source_threads` contains `T1`.
- `ralph_create_op` equals the operation id for `T1`/`M1`.
- `ralph_next_action` is not empty.
- The intake card completed with metadata carrying the issue id **and the issue URL**.
  The URL is the acknowledgement path: you settled that a human comment is the
  acknowledgement (prd.md D6), and a human should not have to search for the issue to
  write one.

**Must not.**
- No second issue.
- No Chat message and no mail sent by us. Nothing in this system can send either (prd.md,
  "Non-goals"), and nothing should: the acknowledgement is a person writing a comment.
  A test that passes while something automated acknowledges the requester is a failure.

---

## A2. A later follow-up in a different thread lands on the same issue

This is the test the whole design exists to pass.

**Setup.** The issue from A1.

**Do.** Feed a second event about the same request, from a different thread `T2`,
message `M2`, whose text does not repeat the first one word for word.

**Must be true.**
- Still exactly one issue.
- `ralph_source_threads` contains both `T1` and `T2`.
- The issue has a new note naming `T2`.
- The note's first line is `ralph-op: <operation id of T2/M2>`.

**Must not.**
- No second issue.
- No `duplicates` relation, because there is no second issue to relate.
- The subject and description are byte-identical to A1. The follow-up appends; it does
  not rewrite.

---

## A3. A Hermes execution card references the issue id

**Setup.** The issue from A1, intake card done.

**Do.** Let the orchestrator fan out one execution card with `parents=[intake card]`.

**Must be true.**
- The execution card is promoted only after the intake card is done.
- Its worker context contains the intake handoff, including the issue id.
- Its completion metadata names the issue id.
- **The issue reads `In Progress` while the execution card runs**, set by the
  orchestrator in the same run that created the card, and attributed in the journal to
  the orchestrator's Redmine account rather than the maintainer's. v2 could assert no
  status here at all; your answer gave the status a writer.
- The report card, child of the execution card, runs after it and adds exactly one note
  to the issue.

**Must not.**
- The execution card holds no Redmine credential and calls no Redmine tool. Check its
  resolved toolset, not its behaviour on one run. It reports through its completion
  metadata, which the orchestrator and the report card read. design.md, "Why the
  orchestrator may, and the executing cards may not", argues where that line falls: the
  orchestrator reads a brief two profiles have filtered, a builder reads a repository
  nobody has.
- The orchestrator must not create, relate, or resolve anything. A8 tests that properly,
  against its Redmine role; here it is enough that the only journal entry it leaves is
  the status change.
- No poller, cron entry, or watcher process is involved. If one is needed to pass this
  test, the design failed.

---

## A4. An ambiguous match waits for a human

**Setup.** Two issues in the project that are plausibly the same request.

**Do.** Feed an event that matches both.

**Must be true.**
- The maintainer's card is blocked, with the candidate issue ids in its reason or
  metadata.
- A human answer naming one issue, followed by an unblock, results in the note landing
  on that issue only.

**Must not.**
- No third issue created.
- Neither candidate is modified while blocked: no note, no status change, no field
  change, nothing in either journal.
- The maintainer does not pick one and proceed.

---

## A5. A retry cannot duplicate

**Setup.** Empty scratch project.

**Do.** Three things, each separately:
1. Run the same intake operation twice in sequence.
2. Run it twice concurrently.
3. Kill the tool process after the Redmine create returns but before the card
   completes, then re-run.

**Must be true.**
- After each, exactly one issue exists and it carries exactly one note per operation.
- The second run reports a no-op, not an error.
- The kanban idempotency key means the second card create returns the first card.

**Must not.**
- No duplicate issue, no duplicate note, in any of the three.
- Dedupe must not depend on a local state file. Delete any state the tool wrote, re-run,
  and the result must be unchanged.

---

## A6. A project the account cannot write to is refused by Redmine

Rewritten in v3. You deleted the client-side allowlist (prd.md D7), so this test asserts
that the server refuses, not that the client refuses first. It needs a live instance now;
a stub transport cannot prove a permission that lives in Redmine.

**Setup.** Two projects exist. The maintainer's account is a member of one of them.

**Do.** Drive the tool at the project it is not a member of, through the `project`
argument on `issue_create`, and by a direct call with the project id if the model cannot
express it.

**Must be true.**
- The call fails on Redmine's 403.
- It surfaces to the model as a permission error and is not retried: a permission error
  is a configuration bug, not a transient (design.md, "Failure and audit").
- The refusal is logged.
- The non-member project has no new issue afterwards. Assert on Redmine, not on the
  tool's return value.

**Must not.**
- The error must not include the API key or the base URL with credentials.
- No fallback: not to another credential, not to another project, and above all not to
  the project the account *is* a member of. Writing the issue somewhere it is allowed is
  the failure this test exists to catch, because it looks like success from the card.

---

## A7. No raw transcript reaches an issue

**Setup.** An event whose source text is long, contains a quoted message block, and
contains a token-shaped string.

**Do.** Run intake.

**Must be true.**
- The issue description is a summary under the byte cap.
- It contains the thread id.
- A create or note over the cap is refused by the tool, not trimmed silently.

**Must not.**
- No verbatim run of source text over the configured length appears in the description,
  any note, or any custom field.
- No token-shaped string appears anywhere in the issue.
- This must fail if the model is instructed to paste the transcript. Write that test:
  the boundary is the tool, and a test that only checks the well-behaved path proves
  nothing about the boundary.

---

## A8. The orchestrator can set a status and nothing else

New in v3. The orchestrator is the second holder of a Redmine key, by your answer, and
with the client-side allowlist gone its Redmine role is the only thing that bounds it.
A boundary nothing tests is a claim.

**Setup.** The issue from A1. The orchestrator's own Redmine account and key.

**Do.** Three things:
1. From the orchestrator profile, set the issue to `In Progress`.
2. From the same profile, attempt to create an issue and to add a relation.
3. With the orchestrator's key directly, outside the tool, POST a new issue to the
   project it can read.

**Must be true.**
- Step 1 succeeds, and the journal attributes it to the orchestrator's account, not the
  maintainer's. Two writers, two names in the history, or the audit trail is a guess.
- Step 2 fails because `issue_create` and `issue_relation_add` are not in its resolved
  toolset at all. Assert on the toolset, not on a refusal at call time.
- Step 3 fails with Redmine's 403. This is the one that matters: it tests the role rather
  than our code, and the role is what survives a bug in our code.

**Must not.**
- The orchestrator must not be able to set `Resolved`, `Closed`, `Rejected`, or
  `Needs Owner`. `In Progress` is its whole status budget (design.md, "Who can update an
  issue").
- It must not share the maintainer's key. If step 3 passes, the two profiles are one
  account and the separation in the design does not exist.

---

## Not tested here, and why

- **Redmine's own correctness.** We test our use of it.
- **Backup and restore.** A drill, not a test, and it needs a scratch database. Listed
  as a task, not an acceptance test.
- **Match quality.** Whether the maintainer's judgment on a borderline case is good is
  not mechanically checkable. A4 tests that it asks when unsure, which is the property
  that matters.
