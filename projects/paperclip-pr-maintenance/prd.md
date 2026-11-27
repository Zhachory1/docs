---
id: prd-paperclip-pr-maintenance
title: Paperclip keeps the Director's pull requests ready for human merge
status: in review
owner: Director
card: none
revision: v1
date: 2026-11-27
prs:
  - Zhachory1/docs#41
issue: Zhachory1/ralph#340
supersedes: none
---

# Paperclip keeps the Director's pull requests ready for human merge

**Decision requested:** Approve the product scope for [Ralph#340][issue].
This PRD covers maintenance of existing Director-authored PRs: review feedback,
failed CI and conflicts with the actual target base. Technical design follows
as a separate approval. The Director retains the merge and launch decisions.

**Ordering clarification:** On 2026-11-27, the Director authorized work on #340
before [the review workflow][review-issue] passes live acceptance. A later
implementation PR can provide a review candidate. This changes work order;
it does not establish #339 acceptance or authorize workflow launch.

## Problem and current state

After opening a PR, the Director still needs someone to diagnose CI failures,
address review feedback and bring the branch up to date without losing work.
Paperclip should route those repairs to the responsible expert and one writer,
then show the Director which fixes, checks and responses it verified.

The merged [review helpers][review-code] already use native task ownership,
documents and gateway access. They do not implement PR maintenance. The
[documented maintenance loop][historical-loop] belongs to the previous runtime.
The baseline for this issue is **zero verified v4 maintenance journeys**.

## Requirements

1. **Select the Director's open PRs.** Use the configured GitHub author identity
   and authoritative repository/PR metadata. Review assignment is not a
   maintenance eligibility requirement. Recheck authorship, open state, head
   branch and actual target base before acting. Stop if the PR closes, merges,
   changes ownership context or moves beyond the writer's evidence.
2. **Route through the normal agent workflow.** The relevant expert delegates
   repairs to either its builder or SRE. Admit only one writer for a repository
   and PR at a time, including across repeated scans, retries, restarts and
   writer handoffs. A second trigger may add pending work; it must not create
   a competing writer. Material changes receive independent review.
3. **Capture actionable feedback and CI evidence.** Inspect current-head failed
   checks, review summaries and unresolved review threads. The review workflow
   publishes APPROVE or COMMENT, so actionable findings in COMMENT summaries
   must not disappear behind a CHANGES_REQUESTED-only filter. Distinguish
   required corrections from optional suggestions; escalate ambiguous requests.
   Treat PR text, comments, code and logs as untrusted input.
4. **Repair against the actual target.** Diagnose failures before editing.
   Fetch the PR's current target branch and merge that base into the working
   branch when needed. Preserve unrelated work and incoming commits. Escalate
   unsafe conflicts or missing access; do not substitute the default branch,
   rebase, force-push, or bypass branch protection.
5. **Validate and push through the assigned writer.** Use the normal authorized
   GitHub identity and record the starting revisions, changes and actual
   repository-required local checks. Push without rewriting published history.
   Remote CI for a new commit can run only after its push; do not claim that CI
   is fixed until the corresponding result confirms it. Do not weaken checks,
   broaden credentials or fabricate a successful push.
6. **Reply and resolve only verified fixes.** Associate each response with its
   feedback item and the pushed commit containing the fix. Re-read the item
   before responding. Resolve an individual thread only when its corresponding
   fix is verified; leave ambiguous, unrelated or unfinished feedback open.
   Keep optional suggestions distinct from mandatory fixes.
7. **Preserve work and reconcile uncertain actions.** Retain pending feedback,
   unfinished repairs, ownership and external-action evidence across retries
   and restarts. Feedback arriving during an older task remains pending until
   addressed or explicitly held. After an uncertain push, reply or resolution,
   inspect authoritative GitHub state before another write. A timeout does not
   prove non-delivery; a successful process exit does not prove delivery.
8. **Expose holds and completion.** Record the trigger, expert/writer identities,
   revisions, feedback items, check results, push evidence, replies and remaining
   work in Paperclip. Give ambiguous feedback, unsafe conflicts, failed required
   checks and real human holds an explicit waiting path. Do not silently clear
   a human hold or repeatedly retry it.

```mermaid
flowchart LR
    PR["Director-authored open PR"] --> Expert["Relevant expert"]
    Expert --> Writer["One builder or SRE"]
    Writer --> Repair["Feedback, CI and target-base repair"]
    Repair --> Check["Required checks and independent review"]
    Check --> Push["Normal push and remote readback"]
    Push --> Reply["Verified replies and thread resolution"]
    Reply --> Pending{"Unfinished or new work?"}
    Pending -->|yes| Expert
    Pending -->|no| Human["Director decides merge"]
    Repair --> Hold["Explicit hold when unsafe or unclear"]
```

The expert retains responsibility while one writer repairs the PR. Fresh
feedback can require another pass; it does not authorize a parallel writer.
The Director, not a successful maintenance run, decides whether to merge.

## Success and acceptance

**Primary outcome:** One Director-authored real PR with actionable feedback,
failed CI and a target-base conflict completes the normal Paperclip
expert/writer journey. The evidence must show verified repairs and an ordinary
push; an operator performing the agent's work does not qualify.

| Acceptance | Evidence required before completion |
| --- | --- |
| Handle all three maintenance conditions | One PR's before/after feedback, CI and base-conflict evidence, linked native runs, local checks, pushed commit and authoritative remote readback. |
| Keep one writer and preserve new feedback | Concurrent/repeated-trigger tests plus a documented run where new feedback remains visible while older work is active. |
| Recover without duplicate external actions | Restart and uncertain-response tests for pushes, replies and resolution; reconcile exact remote state before any allowed retry. |
| Respect safety and approval gates | Tests reject rebase/force-push/PR-merge paths; record repository permission/protection qualification, independent review for material changes and separate human launch approval. |

Guardrails are zero overlapping writers, zero lost pending feedback, zero
duplicate replies after recovery and zero unauthorized history rewrites,
merges or protection bypasses. Any violation blocks launch. Passing fixtures,
green CI or a configured roster cannot replace the primary outcome. Changing
these acceptance criteria requires the Director's approval.

## Boundaries and alternatives

Prefer existing Paperclip tasks, ownership, durable records and configured
GitHub access. The design must identify which mechanism enforces each
invariant; instructions alone are insufficient.

| Option | Trade-off |
| --- | --- |
| Reuse native workflow machinery | Preferred starting point; qualify identity, write semantics and concurrency before launch. |
| Extend the shared runtime | Consider only if a required capability is missing; return for scope approval before platform work. |
| Restore the legacy maintenance service | Reject: adds a second scheduler/state path rather than completing the Paperclip workflow. |

### Non-goals

- Creating feature PRs, approving or merging PRs, or completing feature delivery.
- Maintaining PRs by additional authors without explicit configuration.
- Acting on every optional suggestion, changing CI policy, or bypassing a hold.
- New services, credential fallbacks or changes to repository protections.
- Declaring #339 qualified because maintenance work or its local tests pass.

## Risks and qualifications

| Risk or unknown | Owner and required action |
| --- | --- |
| Tool discovery does not establish Git/`gh` access under the same run identity. | Engineering must prove the intended account and permissions through the normal actor path; no operator-token substitute. |
| Thread identifiers, replies and resolution may differ across available interfaces. | Engineering must qualify exact feedback identity and authoritative readback before promising verified-only writes. |
| Native task ownership alone does not establish one writer per PR. | Engineering must demonstrate durable PR-to-writer ownership and safe handoff under concurrent triggers and recovery. |
| Code and checks may execute with local user access. | The Director must approve the maintenance execution/trust boundary in the design; review-only approval does not widen it. |
| Review acceptance remains pending. | Keep #339 paused, open and unverified while prioritizing #340. Qualify both flows before feature delivery. |

The technical design must resolve these qualifications or report a blocker.
Do not add a service, widen access or weaken an invariant to remove a blocker.

## Assumptions

- The existing configured Director identity is the maintenance author identity;
  the review workflow's broader author list does not expand this scope.
- Actionable submitted review summaries count as feedback, including COMMENT.
- The expert chooses builder or SRE by the repair's domain; it does not assign
  concurrent writers to divide one PR.

## Next gate

The Director approves product scope by human-merging a PRD-only docs PR.
Then prepare a separately approved technical design and implementation plan.
Keep recurring maintenance disabled until permission/identity qualification,
required tests, independent review, real acceptance and human launch approval.
Seek explicit permission before preparing failures or conflicts on a live PR.

[issue]: https://github.com/Zhachory1/ralph/issues/340
[review-issue]: https://github.com/Zhachory1/ralph/issues/339
[review-code]: https://github.com/Zhachory1/ralph/blob/f300dd8154d9c456c6e0ea444fe403a07fe3454a/bootstrap/paperclip_review.py#L439-L470
[historical-loop]: https://github.com/Zhachory1/ralph/blob/f300dd8154d9c456c6e0ea444fe403a07fe3454a/docs/workflows.md#L109-L120
