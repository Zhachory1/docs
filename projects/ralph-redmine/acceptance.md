---
id: redmine-issue-tracking-acceptance
status: in review
owner: Zhachory1
kanban_card: t_021bba46
pull_requests:
  - https://github.com/Zhachory1/docs/pull/5
---

# Acceptance tests: Redmine as the canonical issue tracker

Status: in review, v2. Nothing here has been run. No build authorised.
Covers [the PRD](prd.md) and [design](design.md). Card: t_021bba46.

## Review header

**Decisions needed from you (4).**
1. A1 to A7 run against a scratch Redmine project, not the real one. Confirm you will
   create a `Work Requests (test)` project for this, or accept that nothing is proven
   before first live use.
2. A4 needs a human in the loop by definition. Confirm you will answer one test prompt.
3. A1 gives the requester no acknowledgement, by design. Your comment asked for one, and
   providing one means lifting a non-goal, so it is **prd.md D6**, costed there.
4. A6 tests a client-side `REDMINE_PROJECT_ALLOWLIST`. Your comment said that check is
   not required; it was cut off mid-word, and the two readings differ in size, so it is
   **prd.md D7**, with both readings there.

**Council dissent.** Independent review card t_1f1c1c9a reported no blockers. Your
round-1 review requested changes, and all three comments landed on this file. All three
are answered: two as prd.md D6 and D7, one as design.md decision 4.

**Scope rating.** Seven tests. A1 to A3 are one end-to-end run. A5 to A7 are tool-layer
tests and need no Redmine round trip beyond a stub.

**Changes since last revision (v1 to v2).** A1, A3 and A6 gained the notes your three
comments asked for. The full text of two of those decisions was written here first; it
now lives in prd.md as D6 and D7, with only a pointer left behind, because answering
either changes the PRD rather than the test. A3 now points at the design's argument for
why an execution card holds no Redmine tool, and carries the gap that argument exposed.

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
- The intake card completed with metadata carrying the issue id.

**Must not.**
- No second issue.
- No Chat message sent. Nothing in this system can send one (prd.md, "Non-goals"). The
  only trace of this request outside Redmine is the intake card's completion metadata,
  which carries the issue id and which no requester reads. Whether that is enough, and
  what should replace it if not, is prd.md D6.

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
- The report card, child of the execution card, runs after it and adds exactly one note
  to the issue.

**Must not.**
- The execution card holds no Redmine credential and calls no Redmine tool. Check its
  resolved toolset, not its behaviour on one run. An execution card that needs the
  issue updated does not get its own Redmine access: it reports through its completion
  metadata, and the report card — the only other card besides the intake card that
  runs as `ralph-issue-maintainer` and holds `redmine-tools` — reads that metadata and
  writes the note. design.md, "Who can update an issue", argues why it is kept that way.
- No poller, cron entry, or watcher process is involved. If one is needed to pass this
  test, the design failed.

**Open.** This test asserts no status at all between A1's New and the report card's
note, because nothing in the design sets `In Progress` while the execution cards run.
See design.md decision 4; whichever writer you pick, a status assertion belongs here.

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

## A6. An out-of-scope project is rejected

Written as the design currently stands, and open: prd.md D7 asks whether the client-side
allowlist survives at all. Under its narrow reading this test still exists but asserts
that Redmine refuses, not that the tool refuses first.

**Setup.** Two projects exist in Redmine. Only one is in `REDMINE_PROJECT_ALLOWLIST`.

**Do.** Drive the tool toward the project that is not allowlisted, by whatever argument
reaches it, and by a direct call with the id if the model cannot express it.

**Must be true.**
- The call fails.
- No HTTP request leaves the process. Assert on the stub transport, not on Redmine's
  response, because the point is that the client refuses first.
- The refusal is logged.

**Must not.**
- The error must not include the API key, the base URL with credentials, or the
  allowlist contents.
- Removing the allowlist check must make this test fail. If the Redmine membership alone
  still passes it, the second layer is untested and must be tested separately.

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

## Not tested here, and why

- **Redmine's own correctness.** We test our use of it.
- **Backup and restore.** A drill, not a test, and it needs a scratch database. Listed
  as a task, not an acceptance test.
- **Match quality.** Whether the maintainer's judgment on a borderline case is good is
  not mechanically checkable. A4 tests that it asks when unsure, which is the property
  that matters.
