---
id: dec-hermes-webui-evaluation
title: Keep the built-in dashboard; do not adopt hermes-webui
status: in review
owner: Zhachory1
card: t_d0c0db4f
source: Zhachory1/ralph#2
revision: v2
date: 2026-10-20
prs:
  - Zhachory1/docs#1
---

# Keep the built-in dashboard; do not adopt hermes-webui

Status: **in review** — proposed, awaiting human ratification at the decision gate.
Decision: **Keep the built-in dashboard. Do not adopt hermes-webui.**
Source: Zhachory1/ralph issue #2, "Evaluate hermes-webui as the interface, in place of the built-in dashboard".
Revision: v2 (2026-10-20). v1 is superseded in its reasoning, not in its answer — see "Changed since v1".

---

## Review header

**Decisions needed from you (3).**

1. **Ratify or overturn the recommendation.** It is unchanged from v1 — keep the built-in
   dashboard — but it is now a **weak** preference resting on an absence of demonstrated benefit,
   where v1 claimed a safety case. That safety case did not survive your review and has been
   withdrawn. A reasonable person could decide the other way on this evidence; I am recording my
   reading, not asserting the question is closed. Merging ratifies; a comment overturns.
2. **Settle it on this evidence, or test the thing that was never tested.** Issue #2's own
   strongest complaint is that the built-in Kanban view *"has been least useful during this work"*
   — and the comparison the issue commissioned tested one narrow job (find a blocked card, read
   its reason, clear it), not daily usefulness. So the evaluation never examined the grievance
   that motivated it. A bounded trial would settle that properly; see "How to overturn this
   cheaply". Whether that is worth your time, or whether "no demonstrated benefit" is enough to
   close the question, is a call about your own time and I should not make it for you.
3. **Where this file lives.** Filed as `projects/hermes-webui-evaluation/decisions.md` because the
   card directed it there. This repository's own rule moves a decision that outlives its feature up
   to the owning area, and "which interface runs the board" is an ongoing responsibility rather
   than work with an end. If you agree, move it to `areas/` on merge. Flagged rather than silently
   chosen. (Unchanged from v1; you did not comment on it.)

**Council dissent.**

- **You, in review on PR #1, against v1 — upheld on every point, and v1 was wrong.** Your summary
  was that the record *"has made a lot of assumptions on what I want/need for a service and over
  provisioned on safety even though it's not exposing ports at all."* I agree. Specifically:
  blast radius is not a property of the UI (the same agent capability sits behind the current
  dashboard and is mitigated in base Hermes by sandboxes and approved tools); exposure risk is
  moot while nothing binds past localhost; third-party and always-on are already true of what runs
  today; an unpinned image is a thing to pin, not a thing to refuse; and the attention banner was
  far too thin to be reason #1. Four of v1's five pillars are withdrawn below under "What this
  decision no longer rests on". **Resolution: conceded in full.** The answer survives only because
  the one argument you did not contest — nothing here shows the replacement is *better* — was
  always the load-bearing one, and v1 buried it under four arguments that were not.
- **Agent reviewer (t_b493c10f) against v1 — one finding resolved by your review, one promoted.**
  It flagged that the "full CLI toolset" blast-radius claim rested on an observed config grant
  rather than an executed tool call, and called it *the* load-bearing assumption. That is now
  resolved by subtraction, not by evidence: blast radius no longer differentiates the two UIs, so
  the strength of that evidence no longer matters to the decision. Its second finding is promoted
  to load-bearing: version coupling was sandboxed only on the non-Docker install path while this
  stack deploys via Docker Compose — and version coupling is now the main surviving argument. See
  Assumption 1.
- **Research note vs sandbox report, internal to the evidence — unchanged from v1, and it cuts
  against this recommendation.** The research note predicted no dedicated kanban REST endpoint;
  the sandbox found a first-class kanban surface including `POST /api/kanban/tasks/<id>/unblock`
  and an SSE event stream. **Resolved in favour of the sandbox**, which watched real traffic. This
  is evidence that the kanban integration is more mature than its documentation suggests, and it
  is kept here rather than dropped.

**Scope: S.** One PR, documentation only: this file. No code, no compose change, no port, no
service.

**Changed since v1.**

- The reasoning is rewritten, not softened. Four of five arguments are **withdrawn outright** and
  named as withdrawn, so this record cannot be cited later for claims it no longer makes.
- The attention banner is demoted from reason #1 to a parenthetical. The named job is now recorded
  as a **wash**, full stop.
- "What this decision costs" is **deleted**: its central claim — that the off-desk reply problem
  stays unsolved — was wrong. The notifier plus a poller already covers that, and off-host access
  is a non-starter regardless.
- v1's human-decision #2 ("accept the unmet need, or commission a separate exposure record") is
  **deleted** for the same reason. Replaced with decision 2 above, which is a real open question.
- §3 is rewritten from "Risk axes" into "What the risk axes came to", recording for three of the
  four axes that they **do not differentiate the two options** — and why.
- §5 (exposure) is cut from two paragraphs to three sentences: one factual statement that issue #2
  requires, with the sermon removed.
- Assumptions is restructured: six items are moved out of the list into "No longer assumptions"
  because you have now stated the answers, and the remaining list is reordered by how much weight
  the decision actually puts on each.
- The decision itself is **unchanged**, and is now explicitly recorded as weakly held.

---

## Decision

**Keep the built-in dashboard. Do not adopt hermes-webui as the interface.**

### The reason, in one paragraph

On the one job issue #2 named, the two tools are a wash — neither is better at finding a blocked
card, reading its reason, and clearing it. No other tested property favours hermes-webui, and
adopting it adds a standing upgrade constraint: because it runs Hermes in-process against internal
module layout rather than a versioned API, a pinned hermes-webui is pinned to a *particular*
Hermes, so every Hermes bump needs a hermes-webui that matches it. The built-in dashboard has that
constraint satisfied by construction, since it ships inside the image you already pin. Changing the
control surface of the system for no measured gain, in exchange for a permanent coupling, is not a
trade worth making on the evidence collected. That is the whole case.

### What this decision no longer rests on

v1 gave five reasons. Your review retired four of them, and this record withdraws them explicitly
so that nobody — including a future me — re-litigates this from the old text:

1. **Withdrawn: "its blast radius is the profile's, not a dashboard's."** You are right that this
   is not a property of the UI. The same agent capability is reachable behind the current
   dashboard, and the mitigation lives in base Hermes — sandboxes and approved tools — not in the
   choice of front end. Whatever is true of hermes-webui's reach here is already true today. It is
   not a differentiator and it should not have been argued as one.
2. **Withdrawn: "a password is not safe exposure."** Correct as a general statement and irrelevant
   here: nothing binds beyond 127.0.0.1, you have said that is not going to change, and this is
   your machine. An argument about what *would* be true after a hypothetical exposure is not an
   argument about the decision in front of you. §5 now states the localhost fact once, because
   issue #2 requires it, and stops.
3. **Withdrawn: "third-party, and one more always-on service."** The current UI is already a
   third-party always-on service. The marginal cost of adopting hermes-webui is one more
   lifecycle, which is a real but small operational cost, not a category change — and nothing like
   the notifier's 4.5GB→451MB problem, which was about size, and size is not the issue here.
4. **Withdrawn as a *reason*, retained as a *requirement*: "it does not pin a Hermes version."**
   Your answer — if it is not a pinned image, it should be — is the right one. An unpinned
   dependency is a thing you fix on adoption, not a reason to decline. What survives is strictly
   narrower and is stated in the paragraph above: the in-process coupling means pinning must now
   be done in matched pairs, and that constraint does not go away after you fix the pin.
5. **Demoted, not withdrawn: the attention banner.** You are right that a banner is a very small
   thing to put at the top of a list. It is a genuine ergonomic difference the sandbox observed,
   and it is worth about one sentence, which is what it now gets in §2. The named job is a wash.

### What this decision is weak against

Recorded plainly, because it is the strongest argument on the other side and it came from you, in
issue #2: the built-in dashboard's Kanban view is *"the thing Ralph uses Hermes for most and the
part of the built-in dashboard that has been least useful during this work."* The comparison this
evaluation ran does not speak to that. It tested one narrow interaction and found parity; it did
not test whether hermes-webui's board is nicer to live with across a working week, which is the
actual grievance. So this decision says "no measured benefit", and the honest footnote is that the
benefit most likely to exist was never measured. See decision 2 and "How to overturn this cheaply".

---

## 1. What hermes-webui can reach (sandbox-verified)

Recorded because issue #2's acceptance criteria require it. Read it as a capability description:
after the concessions above, none of it is an argument against adoption, because the same
capability is already reachable today. All observed by running it, not read from documentation.
[SANDBOX]

**Hermes APIs.** hermes-webui is not an HTTP client of a stable Hermes API. It imports Hermes
Python modules in-process — `run_agent.AIAgent`, and the same `hermes_cli.kanban_db` the CLI and
dispatcher use. [SANDBOX][RESEARCH] Roughly 100 distinct `/api/*` calls were captured in one
ordinary session, including `/api/settings`, `/api/profiles`, `/api/profile/active`,
`/api/sessions*`, `/api/models`, `/api/workspaces`, `/api/projects`, `/api/updates/check`, and a
complete kanban surface: `/api/kanban/boards`, `/board`, `/config`, `/assignees`, `/stats`,
`/events/stream` (long-lived SSE), `/tasks/<id>`, `/tasks/<id>/log`, and
`POST /api/kanban/tasks/<id>/unblock`. [SANDBOX]

**Can it act as a profile?** Yes — confirmed by action. The unblock call actually flipped a task
from `blocked` to `ready` in the underlying SQLite board and recorded a canonical `unblocked`
event, cross-checked from a separate shell with `hermes kanban show`. [SANDBOX] Its chat runs with
`toolsets: ["hermes-cli"]`, read from the running config rather than the README — the same bundle a
terminal `hermes chat` session gets: terminal, code execution, file read/write, cron, delegation.
[SANDBOX][RESEARCH] A profile switcher means whoever is authenticated acts as the selected profile.
[RESEARCH] This is the same capability that already sits behind the current dashboard and is
mitigated at the Hermes layer, so it is recorded here as fact and carries no weight in the
decision.

**Files.** The *workspace browser panel* is scoped to a configured directory and never listed
outside it. [SANDBOX] The *chat agent's* file and code-execution tools are not scoped to it at all.
[SANDBOX] Worth knowing if you ever expected the panel's scoping to be a containment boundary; it
is not one.

**One finding that is specific to this tool and does survive.** hermes-webui resolves which board
to show from `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD` environment pins, *not* from `HERMES_HOME`,
and its documentation does not mention those variables. Launched from a shell that already had
those pins set, it came up **labelled "Board: Default" while actually displaying the live board**.
No writes occurred and it was corrected immediately. [SANDBOX] This is not a safety-posture
argument: it is a correctness hazard about acting on the wrong board while the UI tells you
otherwise, it was hit in practice rather than hypothesised, and following the tool's own documented
isolation recipe would not have prevented it. It is mitigable by setting the variables explicitly
on launch. It also is not purely hermes-webui's problem — see Observations.

**Secondary observation.** Opening hermes-webui's page makes the *browser* fetch seven assets from
`cdn.jsdelivr.net` (xterm, prism, katex), all 200 OK; the server itself only ever talked to
127.0.0.1. The built-in dashboard made zero non-localhost requests because its assets are bundled.
[SANDBOX] Recorded as a real difference, weighted as minor.

## 2. The comparison issue #2 asked for: find a blocked card, read its reason, clear it

Run independently on each UI against a throwaway board with realistically-worded blocked cards.
[SANDBOX]

| | hermes-webui | Built-in dashboard |
|---|---|---|
| Finding it | Open Kanban tab, scan the Blocked column for a ⊘/BLOCKED marker | "⚠ 1 task needs attention" banner, clickable, jumps straight to it |
| Reading the reason | One click; reason appears as the summary line and verbatim in Comments | One click; reason in a dedicated "Final Result (run summary)" field, separate from the event history |
| Clearing it | One "Unblock" button, no confirmation, immediate (`POST .../unblock`) | One "Unblock" button, no confirmation, immediate (`PATCH ... status=ready`) |
| Audit trail | Canonical `unblocked` event | Same canonical `unblocked` event |

**Verdict: a wash.** Reading and clearing are indistinguishable from the operator's seat; the
different endpoints underneath are a maintenance detail, not a user-facing one. The dashboard's
attention banner is a small genuine convenience on finding, and that is all it is — not a reason to
prefer the incumbent, and corrected here from v1, which over-weighted it. [SANDBOX] Screenshots of
both card-detail and post-unblock states were captured as evidence.

**What this comparison does not cover**, and should be read as a limit on the whole evaluation: one
interaction on a small throwaway board, by an operator who had just built both boards. It says
nothing about either Kanban view across a working week on a busy real board, which is the use issue
#2 actually cares about.

## 3. What the risk axes came to

Issue #2 named four axes. After review, three of them do not distinguish the two options. Recording
that explicitly is the point of this section — a retired axis should be retired in writing, not
quietly dropped.

**Third-party project holding the board, profiles, and a credentialed chat — does not
differentiate.** The facts stand: ~19,018 stars (GitHub API, confirming the issue's "~19k"), MIT,
2,747 forks, 951 open issues/PRs, 326+ contributors. [RESEARCH] But the current dashboard is also
third-party code with board-mutation reach, and the agent capability behind it is the same one,
mitigated in base Hermes rather than in the front end. The honest residual is narrow: adopting
hermes-webui means a *second* independent codebase with that reach instead of one, so its supply
chain is additional attack surface rather than substituted attack surface. That is a real but small
increment, not the step change v1 described. One unresolved gap, kept in proportion and not
load-bearing: commit recency verified only to 2026-08-26, a reported `pushed_at` of 2026-10-18 that
did not reconcile with the commits API, and issue-response latency unmeasurable because the API
rate-limited. [RESEARCH]

**Password versus exposure — moot.** Nothing in this stack binds beyond 127.0.0.1 and you have said
that is not changing. The posture difference is real in the abstract — hermes-webui's
`docs/remote-access.md` positions its password as the enabler for `HERMES_WEBUI_HOST=0.0.0.0`
[RESEARCH], where the built-in dashboard's unauthenticated-loopback stance makes no such offer —
but an argument about a configuration nobody is going to adopt is not an input to this decision.
Withdrawn as a reason. §5 keeps the one factual sentence issue #2 requires.

**Hermes version coupling — this is the surviving argument.** hermes-webui pins no Hermes Agent
version on either install path: `bootstrap.py` installs hermes-agent's `main`, and its
two-container compose uses the floating `nousresearch/hermes-agent:latest`. [RESEARCH] As you said,
that part is fixable by pinning and is a requirement, not an objection. What is not fixable by
pinning is why the pin has to be exact: chat runs against Hermes internals in-process — module
layout, the `AIAgent(...)` constructor signature, `run_conversation(..., task_id=...)`, the
`state.db` schema — so an upstream refactor can break it while Hermes's documented API is untouched.
This has already happened once: `api/agent_compat.py` exists as a "compatibility-only" shim for
Hermes names that moved, and the project's own RFC calls the arrangement *"a compatibility bridge,
not the long-term API contract."* [RESEARCH] The sandbox saw the coupling mechanically: bootstrap
probed for an interpreter that could `import run_agent.AIAgent` before it would start, and
succeeded only because the host's install happened to be shaped the way hermes-webui expected.
[SANDBOX] **Concretely, the cost you would be taking on:** every Hermes image bump becomes a
two-sided upgrade — find a hermes-webui revision compatible with the new Hermes, or hold the bump —
and the canary has to cover both. The built-in dashboard has no such step, because it ships inside
the image already being pinned. **Caveat, and it matters:** this was sandboxed on the non-Docker
path only; the Docker-path evidence is read from compose files, not run, and this stack is Docker
Compose. See Assumption 1.

**One more service — real, small.** A second always-on HTTP server with its own state directory,
session and attachment storage, PID and log files, `ctl.sh` lifecycle, and an update check firing
on every page load. [SANDBOX] Its footprint is modest: pure Python and vanilla JS, no bundler or
runtime build step. [SANDBOX] Size is not the complaint and the notifier comparison does not apply.
The marginal cost is one more lifecycle to run, upgrade, back up, and reason about during an
incident. One line of cost, weighted as one line.

## 4. Non-goals

- **Replacing the notifier.** Chat stays the alert path. Out of scope per issue #2 — and per your
  review, the notifier plus a poller is already how you communicate with this stack.
- **Any exposure beyond localhost.** See §5.
- **A general feature comparison.** Issue #2 named one job; this record evaluates it and declines
  to accumulate a feature matrix.
- **Judging hermes-webui as a project.** It looks competent and well-built, and its kanban
  integration is more mature than its documentation suggests. This record is about fit here.
- **Implementing anything.** No code, compose, or port changes accompany this record.
- **Re-litigating the base system's agent-capability model.** Sandboxes and approved tools in base
  Hermes are the mitigation layer and are not this record's business.

## 5. Exposure

Nothing in this evaluation was exposed beyond 127.0.0.1: both tools stayed loopback-only for the
entire sandbox run, verified via `/proc/net/tcp` and their startup banners, with no password on
either. [SANDBOX] **This record does not authorize exposing anything beyond localhost**, which
issue #2 places out of scope; any future change there would be its own decision, for either tool.
Keeping the built-in dashboard is not an argument that its loopback binding is now safe to relax.

## How to overturn this cheaply

This decision is weakly held, and the cheapest thing that would overturn it is not more analysis —
it is a week of use. With the risk case withdrawn, a trial costs little: hermes-webui installs
without touching the existing dashboard, both can run at once against the same Hermes home, and
nothing needs to bind off the host.

A bounded trial that would actually settle decision 2: run it on localhost against the live board
for a working week alongside the dashboard, launched with `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD`
set explicitly so the board-labelling hazard in §1 cannot bite, with its Hermes revision pinned so
the coupling cost is observed rather than argued. If its Kanban view is meaningfully better to live
with, that is the measured benefit this record says is missing, and this decision should be
overturned on that evidence. If it is not, this is settled.

Otherwise, re-open only if one of these changes — "someone found the project again" is not a
reason:

1. It talks to Hermes over a versioned, documented API instead of in-process imports — its own
   RFC's "long-term API contract" — removing the matched-pair pinning constraint.
2. The built-in dashboard's Kanban view regresses or is removed upstream.
3. The constraint that only the local machine runs this changes. That would reopen the interface
   question legitimately, and should be decided on its own merits first.

## Assumptions

Every inference the brief and the sources did not state, ordered by how much this decision leans on
each.

1. **Load-bearing: the Docker install path was never exercised.** hermes-webui was installed via
   `python3 bootstrap.py` against an existing agent directory; the Docker paths were read from
   compose files, not run. [RESEARCH] This matters more in v2 than it did in v1, because version
   coupling is now the main surviving argument and this stack deploys via Docker Compose. If you
   are inclined to overturn, this is the gap to close first: the matched-pair pinning cost is
   inferred for the path you would actually use. Flagged by the agent reviewer against v1 and
   promoted here.
2. **Low weight now: toolset reach was observed as a grant, not exercised.** The sandbox read
   `toolsets: ["hermes-cli"]` from the running config but did not execute a tool call through a
   live chat turn, because the scratch profile was deliberately given no model credentials.
   [SANDBOX] In v1 this was the load-bearing assumption; it no longer is, because blast radius no
   longer differentiates the two options.
3. **Low weight: the comparison is of the software, not of this deployment.** The sandbox ran its
   own instances on 127.0.0.1:8788 and 127.0.0.1:9129 against a throwaway board, not this stack's
   dashboard on 127.0.0.1:9219. [SANDBOX]
4. **Low weight: maintenance health is partly unverified.** Commit recency confirmed only to
   2026-08-26; issue-response latency not measured (API rate-limited); no CODEOWNERS found.
   [RESEARCH] Not load-bearing — but if you overturn to "adopt", it becomes so and should be
   re-checked.
5. **Low weight: footprint is characterized, not measured.** No published image size; "modest"
   rests on the observed absence of a runtime build step and bundled ML dependencies. [SANDBOX]
   Your review makes this near-irrelevant: size was never the complaint here.
6. **Low weight: the CDN egress was not traced further.** Seven jsdelivr assets observed loading;
   whether they can be vendored or disabled by configuration was not investigated. [SANDBOX]
7. **My reading of your comment on exposure, which I may have wrong.** You wrote, against v1's
   claim that exposure should be decided first: *"We already have the decisions right? It's
   whatever is available now, the notifier, the kanban board, and the API."* I read that as: there
   is no pending exposure decision for this record to wait on — what is reachable is what is
   running now, and it stays local. v1's "decide exposure first, then pick an interface"
   sequencing is withdrawn on that reading. If I have misread you, that withdrawal is the thing to
   correct.
8. **This pull request comes from a fork.** The credential available to me authenticates as
   `zhach1`, which has `push: false` on `Zhachory1/docs`, so the branch lives on a fork and PR #1
   is cross-fork. Nothing about the content depends on it; if these documents are meant to land on
   branches of `Zhachory1/docs` itself, that account needs write access.

### No longer assumptions — stated by you in review

v1 guessed at these. It should not have, and that is the substance of your "a lot of assumptions
on what I want/need" — recorded here so they are never re-assumed:

- **Off-desk replies are already handled.** The notifier is in the stack, and with a poller
  watching messages you can communicate with it. v1 assumed this need was open and motivating; it
  is not.
- **Off-host access is a non-starter.** Only the local machine runs this. v1 treated remote access
  as a deferred benefit; it is a closed question.
- **The risk appetite is yours, on your machine.** v1 applied a conservative posture to a service
  exposing no ports. Withdrawn.
- **Blast radius is mitigated in base Hermes**, by sandboxes and approved tools, and does not
  depend on the UI. v1 treated it as a UI property. That was an error, not merely an assumption.
- **The current UI is already a third-party always-on service.** v1 counted both as new costs.
- **An unpinned image is a thing to pin.** Your words: if it is not a pinned image, it should be.
  v1 treated unpinned-ness as a reason to decline rather than a task on adoption.

## Observations — out of scope, not acted on

- **The board-resolution hazard is not hermes-webui's alone.** Any tool resolving the board from
  ambient `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD` pins can display the live board while labelled
  otherwise. The sandbox hit this. It may deserve its own hardening card regardless of what is
  decided here.
- **The two tools are not mutually exclusive.** hermes-webui ships a compose layout running both
  against the same Hermes home, with a documented history of lock-file races between them.
  [RESEARCH] Not relevant under this decision; directly relevant to the trial above.
- **`GH_TOKEN` is not forwarded into dispatcher-spawned workers** (only `GIT_AUTHOR_NAME`/`EMAIL`
  are), so an authoring worker has to recover it from elsewhere to push. Worth fixing if authoring
  workers are expected to open pull requests routinely.

## Sources

- **[SANDBOX]** — Empirical sandbox run, 2026-10-20: hermes-webui installed from `master` HEAD via
  `python3 bootstrap.py --host 127.0.0.1`, run against a purpose-built throwaway kanban board
  alongside the built-in `hermes dashboard`, both loopback-only; both UIs driven with
  Playwright + Chromium with full network capture; board effects cross-checked independently via
  `hermes kanban show`; port bindings confirmed via `/proc/net/tcp`; torn down afterwards.
  Recorded on the ralph board as card `t_18025034`, with four screenshots attached.
- **[RESEARCH]** — Read-only source and documentation review, 2026-10-20, of
  `github.com/nesquena/hermes-webui` at `master`: README, ARCHITECTURE.md, docs/docker.md,
  docs/remote-access.md, Dockerfile, requirements.txt, LICENSE, `.env.docker.example`, both
  compose files, plus the GitHub repository and commits APIs. Recorded on the ralph board as card
  `t_2e0a9333`.
- **Review of v1** — six line-anchored review comments and a summary comment by `zhach1` on
  `Zhachory1/docs#1`, 2026-10-20, against the v1 blob; plus the agent review recorded on card
  `t_b493c10f`.
- **Issue** — Zhachory1/ralph#2, "Evaluate hermes-webui as the interface, in place of the built-in
  dashboard", read at its current body, including its acceptance criteria and "Not in scope".
