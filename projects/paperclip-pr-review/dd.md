---
id: dd-paperclip-pr-review
title: Submit one APPROVE or COMMENT summary through native Paperclip tasks
status: in review
owner: Director
card: none
revision: v3
date: 2026-11-27
prs:
  - Zhachory1/docs#39
implements: prd-paperclip-pr-review
issue: Zhachory1/ralph#339
supersedes: none
---

# Submit one APPROVE or COMMENT summary through native Paperclip tasks

**Director-confirmed amendment:** Replace pending drafts and inline threads with
one submitted **APPROVE or COMMENT** review. Its summary contains labeled
findings, file/line references and actual checks. The Director chose this format
on 2026-11-27 after the native API compatibility gap was explained.

**Approved base:** [docs#39][base] approved native routines, existing
expert/reviewer agents, a trusted helper, local checks and the shared GitHub
gateway. This amendment changes the publication protocol, not the author policy
or launch gate. No new service, provider, MCP server or security plugin.

**Scope:** PRs assigned to `Zhachory1`, authored by `zhachbot`, `Zhachory1` or
`zhach1`, within one configured company. The optional repository filter is empty
here: no repository restriction within the connection's access. Local checks
run with the OS user's access. Spend-limit setup remains deferred.

Record the format clarification in [prd-paperclip-pr-review][prd] alongside this
design. Continue local implementation against the Director's explicit choice.
Publishing the amendment and implementation PRs still needs authorization;
enabling the workflow and its real acceptance review still needs launch approval.

## Native capability and the corrected protocol

Paperclip evidence uses the previously inspected source at
`e716c6cea0e3672d02fd97df20804743be4d8308`. GitHub MCP evidence uses `v2.3.0`.
These establish source contracts, not live acceptance by the configured company.

| Capability | Verified source | Consequence |
| --- | --- | --- |
| Native scheduling | [Routine contract][routines] creates execution issues and supports concurrency/catch-up policies. | Reuse routines and heartbeats, not another polling service. |
| Ownership and documents | [Issue checkout][checkout], [document revision checks][documents], [unique document creation][document-create]. | Use native tasks/documents for provenance, send reservation and receipt. |
| Creation keys expire | [Issue creation][issue-create]. | Issue-creation idempotency is not a permanent publication ledger. |
| Gateway identity/idempotency | [Gateway][gateway], [explicit invocation keys][invocations], [run-specific default keys][default-key]. | Use the configured connection and an explicit cross-run key. |
| Direct review creation | [Review write handler][review-write] accepts `create`, explicit event, `commitID` and body; the returned message is not a review ID. | Submit once, then confirm the review through authoritative reads. |
| Repository/review identity | [Repository search][repo-search] exposes ID/full name; [review reads][review-types] expose review ID, account identity, commit, state and body. | Match exact repository and publishing account; locate the submitted marker. |

The former protocol assumed pending creation preserved its body marker and
inline readback exposed each comment's parent review. The inspected native
contract does neither reliably enough for that protocol. A direct submitted
summary avoids those operations instead of inferring their missing receipts.

## Alternatives

| Option | Scope/effort | Trade-off and reversibility |
| --- | --- | --- |
| **A: One submitted summary** | Small native helper patch; no pending/inline state machine. | Selected by the Director. Keeps classified findings, source references and checks. Pause native intake/tasks to stop; retain receipts. |
| B: Extend the native MCP provider for inline recovery | Provider changes, pending-visible scoped comment reads and deployment qualification. | More maintenance and outside this change. Revisit if inline threads become required. |
| C: Another service, MCP server or direct-GitHub fallback | New transport, credentials or infrastructure. | Reject: duplicates/bypasses the chosen native path. |

## Discovery and handoff

1. Private configuration supplies company/project/connection references, an
   unassigned ledger task, coordinator/default expert/reviewer IDs, optional
   repository routes, assignment/publishing identities and filters. Public
   examples contain placeholders and an empty author list that denies all.
2. Render a **paused** native routine and a disabled three-minute schedule
   trigger. Use `coalesce_if_active`, `skip_missed` and activity gate `always`:
   a GitHub assignment need not create Paperclip activity.
3. Search open PRs assigned to the configured account and inspect authoritative
   metadata. Before reading PR contents, routing or executing checks, require
   non-draft state and an allowlisted author. Do not use `@me`, CI status or a
   separate review request as eligibility gates.
4. Empty repository filters admit any accessible repository; non-empty filters
   admit only listed repositories. Empty author lists admit nobody. Malformed
   configuration fails closed. Resolve account/repository identities from
   GitHub metadata, not claims inside PR descriptions.
5. Create keyed expert tasks as explicit standalone tasks, linked from the
   ledger/routine record. Experts delegate to a distinct reviewer as a direct
   child and use native dependencies to resume. This avoids unsupported
   third-level nesting. Use repository overrides or the configured default
   route; missing routing/authority is a visible blocked result.
   Dispatch keys distinguish head/base analysis versions; publication K remains
   head-based. A base change before any send can schedule fresh analysis without
   granting another publication attempt.
6. The reviewer saves its structured result on its own task and completes
   through native lifecycle. The expert publishes only after that task and
   native reviewer run have completed. Do not require a delegate to write a
   parent-task comment.

```mermaid
flowchart LR
    Routine["Paused native routine until launch"] --> Discover["Assignment, author and optional repo checks"]
    Discover --> Expert["Domain expert task"]
    Expert --> Reviewer["Independent reviewer and local checks"]
    Reviewer --> Result["Pinned result and command evidence"]
    Result --> Reserve["Recheck and reserve one send"]
    Reserve --> Gateway["Existing native GitHub gateway"]
    Gateway --> Summary["One APPROVE or COMMENT summary"]
    Summary --> Receipt["Confirm account, head, marker and review ID"]
```

The native agents own review delivery. The helper validates their result and
performs one review write. It does not create a private draft, manage inline
comments or reserve the account's pending-review UI slot.

## Result and verdict contract

Record repository ID/name, PR number, head/base SHAs, expert/reviewer task and
agent/run references, coverage, findings and check evidence. The helper loads
the exact native result revision and verifies the server-recorded writer/run,
assigned reviewer, delegated task relationship and completed run. Caller text
alone cannot establish provenance.
The reviewer verifies the pinned checkout/comparison and reports
`reviewedHead` and `reviewedBase`; mismatches are rejected rather than stamping
planned SHAs onto an unpinned result.

Publish these summary sections:

- **Blocking findings:** PR-caused must-fix defects, each with evidence,
  correction and an applicable file/line reference. Say `None` when empty.
- **Non-blocking suggestions:** Optional improvements/preferences and labeled
  observations. These may accompany approval.
- **Checks:** Actual commands, outcomes and checks not run with reasons.
  Report CI state without treating green CI as a review prerequisite.

Include reviewed head/base, coverage limits and explicit automated attribution.
Do not publish credentials, private host paths or raw transcripts. A blocking
label is a recommendation, not a GitHub change-request block.

APPROVE requires sufficient evidence and no blockers. COMMENT covers blockers,
other concerns, insufficient evidence and self-authorship restrictions.
The helper rejects REQUEST_CHANGES, missing/unknown events and APPROVE with
blockers before transport. Configure `Zhachory1` for assignment and publishing
per the Director's answer; verify the actual gateway account. Its own PRs
receive COMMENT, not self-approval. The reviewer must not approve its own work.

## One-write publication and reconciliation

Define a versioned key `K` from unambiguously encoded company, repository ID,
PR number, head SHA and publishing account ID. Base SHA is evidence and part
of the request hash, not part of K: a base change must not permit a second
review for the same head/account.

Use the existing run-scoped native API/gateway authority. Validate the ledger's
company/project and unassigned state; native access denial is a block, not a
reason to introduce a board token or a direct API bypass. Verify account ID/login
through `get_me` on the same connection used for writing.

Before sending:

1. Read and paginate prior reviews completely. Any submitted review by the
   configured account for this head, including a manual one, suppresses another
   review. Adopt it as an automated receipt only with the matching marker,
   durable send intent and native gateway send record. Otherwise record
   `already_reviewed_elsewhere` or a provenance conflict without claiming
   automated acceptance.
2. Leave unrelated human pending reviews untouched. If pending state prevents
   direct creation, report the conflict. Never switch to `submit_pending`,
   modify a draft or downgrade a failed APPROVE into another write.
3. Recheck active native run/task authority, publishing identity, open state,
   assignment, readiness, author/repository filters and exact head/base.
   Changed evidence invalidates the prepared result before any send.
4. Create one immutable native `send-<digest(K)>` document, at most 64 characters
   in key length, without `baseRevisionId`. Store exact payload hash, result
   revision, event, revisions, owning task/run and a fresh invocation nonce.
   Only an acknowledged new creation at the expected key/body/nonce permits
   sending. Existing intent, uncertain acknowledgement, conflict, replayed
   creation or a locked-document fork never grants send authority.
5. Recheck authority/revisions after reservation, then call
   `pull_request_review_write` with `method: create`, explicit APPROVE/COMMENT,
   explicit `commitID` and the summary body plus marker derived from K and
   a pre-marker content hash. Store the full on-wire payload hash in the intent
   after appending that marker, avoiding a self-referential hash. Supply K as
   the gateway idempotency key. Never omit
   `event`; that would create the unsupported pending-draft path.
6. Read GitHub to confirm the submitted account ID/login, commit, state, body
   marker and review ID/link. Record the matching native invocation and remote
   receipt before reporting delivery. A successful process, message-only tool
   response or idempotent replay is not a confirmed GitHub receipt.

Two concurrent workers cannot both obtain the same unique send reservation.
No per-PR pending slot, extra publisher process, client-token rotation or
generation-reset machinery is required. Keep the intent across restarts and
do not expire it with issue-creation keys.

| State | Action |
| --- | --- |
| No prior review or intent | Eligible for final validation and one reservation. |
| Matching submitted review and native send evidence | Confirm/adopt its receipt; no write. |
| Manual/other submitted account/head review | Record external review and skip; not automatic acceptance. |
| Intent exists, response/receipt missing | Read GitHub and native invocation state before doing anything else. |
| Remote result or native provenance remains uncertain | Keep `submission_unknown` and the intent; no resend, alternate key or automatic reset. |
| Drift after transport | Retain the receipt, record staleness and notify the operator; no second review, dismissal or merge. |

A crash after reservation but before sending can leave a head requiring manual
reconciliation. This deliberately favors duplicate prevention. The operator
must prove non-delivery and settle prior work before approving recovery;
elapsed time or an empty read alone is insufficient.

GitHub pins the reviewed head commit but does not atomically condition the
write on base SHA/assignment. Recheck immediately before sending and afterward;
keep this known race visible rather than promise cross-system atomicity.
The one-send guarantee covers this workflow, not independent human actions:
a concurrent manual review after the final history check can still add another
review. Direct creation does not target or modify the human's pending draft.

## Local checks and operational gates

Use separate local checkouts at the recorded revisions, not the operator's
working tree. Recheck author/repository policy before contents or execution.
PR scripts have the OS user's access, including reachable files, credentials
and network; the Director accepted that trust model. The helper does not
execute arbitrary strings from review-result JSON as shell commands.

No sandbox/provider setup, managed low-trust runtime, publishing plugin or
spend-limit setup. Keep secrets in native storage and out of examples/reviews.
Unknown tools, failed commands or unavailable checks remain explicit omissions.

Stop new work by pausing the native routine. Cancel active tasks through the
existing stack and reconcile in-flight review writes; do not assume pausing
ticks cancelled them. Preserve send intents and receipts during rollback.

## Validation and delivery

| Acceptance | Required evidence |
| --- | --- |
| Filters and routing | Allowed/unknown authors; empty authors deny all; empty/non-empty repository filter semantics; malformed config; draft/closed/unassigned negatives; default and overridden routes. |
| Normal handoff/provenance | Standalone expert and direct-child reviewer; native dependency wake; exact completed reviewer run/result writer; wrong task/writer/revision and unfinished-run rejection. |
| Summary verdict | Labeled sections and file/line references; actual commands/omissions; APPROVE with suggestions; COMMENT with blockers/insufficient evidence/self-authorship; no REQUEST_CHANGES or pending/inline write. |
| One send and uncertainty | Concurrent reservations, locked forks, lost acknowledgements, restarts, expired creation keys, message-only/replayed responses, lost submit response, failed receipt save, manual prior review and incomplete history. No blind resend. |
| Revision safety | Head/base/eligibility/filter drift before and after reservation; post-send drift visible; new eligible head obtains a new K. |
| Real acceptance, after launch approval | One allowed real PR traverses routine, expert, reviewer, local checks, gateway and confirmed summary; repeat scan/restart does not duplicate it. |

Use at most two implementation PRs against Ralph's `clean-slate` base:
helpers/tests, then paused wiring and qualification. Combine an indivisible
slice rather than add a service or split artificial scaffolding PRs.
Passing local tests is not the real acceptance journey.

## Non-goals

- Inline threads, pending drafts and manual submission of staged reviews.
- PR maintenance, changing reviewed code, merges or deployments.
- Another scheduler/queue/datastore/MCP server, publisher plugin or provider fix.
- New company/agent provisioning, memory capture or a broad #343 access rollout.
- Universal test support, sandboxing or mandatory spend-limit setup.

## Assumptions and open qualification

- Three-minute polling and the configured existing expert/reviewer route fit
  the trusted-author scope.
- `Zhachory1` names both instance identities; the actual connection must match.
- Native actors can use the unassigned ledger under their ordinary authority;
  denied access is not worked around.
- The connected review listing supplies complete, unfiltered history or
  reliable completion evidence. Unsupported filtering/truncation blocks
  publication; qualify the actual tool contract before enabling it. Record
  that qualification in private `reviewHistoryUnfiltered` configuration,
  false by default; it is not evidence by itself.
- Trusted agents follow the helper contract. It is not adversarial containment
  of a compromised author, account, local process or agent.

## Observations

- Issue #339 retains the former sandbox/authorship wording; align its brief
  with the Director's approved clarifications before launch.
- The deployed native identity, tool contract and trigger-to-review journey
  remain unverified. No live configuration or publishing action is authorized
  by this amendment.

## Next gate

Review this Director-confirmed amendment alongside the linked implementation
PR. Publishing these changes does not merge or launch them.
Keep the workflow disabled until Director launch approval and normal-path
qualification; keep #339 open until the revised acceptance cases pass.

## Sources

[base]: https://github.com/Zhachory1/docs/pull/39
[prd]: https://github.com/Zhachory1/docs/pull/38
[routines]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/skills/paperclip/references/routines.md#L1-L107
[checkout]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/server/src/services/issues.ts#L12904-L12964
[issue-create]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/server/src/services/issues.ts#L10730-L10772
[documents]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/server/src/services/documents.ts#L410-L425
[document-create]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/server/src/services/documents.ts#L487-L584
[gateway]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/doc/MCP-ACCESS-GOVERNANCE.md#L55-L157
[invocations]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/server/src/services/tool-access-policy.ts#L1863-L1878
[default-key]: https://github.com/zhachbot/paperclip/blob/e716c6cea0e3672d02fd97df20804743be4d8308/server/src/services/tool-access-policy.ts#L316-L327
[review-write]: https://github.com/github/github-mcp-server/blob/v2.3.0/pkg/github/pullrequests.go#L1973-L2200
[repo-search]: https://github.com/github/github-mcp-server/blob/v2.3.0/pkg/github/search.go#L23-L138
[review-types]: https://github.com/github/github-mcp-server/blob/v2.3.0/pkg/github/minimal_types.go#L782-L811
