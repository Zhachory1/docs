---
id: dec-hermes-webui-evaluation
title: Keep the built-in dashboard; do not adopt hermes-webui
status: in review
owner: Zhachory1
card: t_2f2c6131
source: Zhachory1/ralph#2
revision: v1
date: 2026-10-20
prs:
  - Zhachory1/docs (this pull request)
---

# Keep the built-in dashboard; do not adopt hermes-webui

Status: **in review** — proposed, awaiting human ratification at the decision gate.
Decision: **Keep the built-in dashboard. Do not adopt hermes-webui.**
Source: Zhachory1/ralph issue #2, "Evaluate hermes-webui as the interface, in place of the built-in dashboard".
Revision: v1 (2026-10-20).

---

## Review header

**Decisions needed from a human (4).**

1. **Ratify or overturn the recorded decision.** The evaluation recommends keeping the built-in
   dashboard. Merging this record is the ratification; rejecting it with a comment is the
   overturn. This record does not merge itself.
2. **Accept the unmet need, or commission a separate path for it.** The issue's actual
   motivation — answering a blocked card away from a desk — is *not* satisfied by this decision,
   and would not have been satisfied by adopting hermes-webui either, because the only thing that
   satisfies it is exposure beyond localhost, which issue #2 places out of scope. Someone has to
   decide whether that need is accepted as open, or gets its own exposure decision record. I
   cannot decide that; it is a risk-appetite call.
3. **Set the re-evaluation trigger.** Section "When to re-open this" proposes conditions. Whether
   those are the right conditions, and whether anyone is obliged to watch for them, is a human
   call.
4. **Whether this decision outlives the project folder.** It is filed as
   `projects/hermes-webui-evaluation/decisions.md`, matching the repository's project-folder
   shape. But "which interface runs the board" is an ongoing responsibility rather than work with
   an end, and this repository's own rule is that a decision outliving its feature moves up to the
   owning area. If you agree, this should move to `areas/` on merge. I filed it under `projects/`
   because that is where the card directed it, and flag the tension rather than silently choosing.

**Dissent.** No independent critic has reviewed v1 yet; the agent review is the next card in this
chain, and a human gate follows it. What I can record honestly is the disagreement *between the
two source investigations*, since inventing a council would be worse than saying so:

- The research note predicted hermes-webui exposes **no dedicated kanban REST endpoint**, and that
  board actions would run through agent tool-calls like the CLI does. The sandbox found the
  opposite: a full first-class kanban REST surface including a dedicated
  `POST /api/kanban/tasks/<id>/unblock`. **Resolved in favour of the sandbox**, which observed
  running traffic; the research note was reading a docs table that is incomplete. This cuts
  *against* my own recommendation — a dedicated unblock endpoint is a sign of a more mature
  kanban integration than the docs suggested — and it is recorded here rather than dropped.
- The research note and the sandbox agree, independently and by different methods, on the finding
  that actually drives the decision: the chat surface is not read-only, it carries the full CLI
  toolset. Neither investigation found evidence against it.

**Scope: S.** One PR, documentation only: this file. No code, no compose change, no port, no
service. If the decision is overturned to "adopt", the follow-on design document is already
provisioned as a separate card and is not part of this PR.

**Changed since last revision.** N/A — v1.

---

## Decision

**Keep the built-in dashboard. Do not adopt hermes-webui as the interface.**

"Keep the built-in dashboard" is one of the two outcomes issue #2 explicitly allows, and it is
recorded here as a chosen answer with reasons, not as a default that happened because nothing
better turned up.

### Reasoning, in the order that mattered

**1. On the one job the issue named, the two tools are a wash — with the edge to the incumbent.**
The issue asks for exactly one comparison: see a blocked card with its reason, and clear it. Both
tools do it in one click, with no confirmation step, with immediate effect, and with an identical
audit trail. [SANDBOX] Where they differ, the built-in dashboard is *ahead*: it carries a standing
"⚠ N task(s) needs attention" banner that jumps straight to the thing needing action, which
hermes-webui has no equivalent of — a difference that grows with board size, not shrinks.
[SANDBOX] A replacement that is not better at the job it is being adopted for has to be carried
entirely by its other properties, and its other properties are costs.

**2. The thing that would have justified it is out of scope by the issue's own terms.** The
issue's motivation is real and I am not dismissing it: gates get answered away from a desk, the
notifier already reaches a phone, and the reply currently requires a laptop. But hermes-webui does
not fix that while bound to localhost — nothing does. It fixes it only when exposed, and issue #2
puts "exposing anything beyond localhost without a separate decision" out of scope. So the in-scope
version of adopting hermes-webui delivers a Kanban view that is no better at the one job, and
defers the entire benefit to a decision this record is not allowed to make. The honest sequencing
is the reverse of adoption: decide about exposure first, on its own merits, and only then ask which
interface deserves to be exposed.

**3. Its blast radius is the profile's, not a dashboard's.** See the next section. This is the
issue's central question — "what it can reach and what happens when it is wrong" — and the answer
is that a bug or compromise in hermes-webui is equivalent to a compromised terminal session for the
selected profile, including repository credentials, not to a broken read-only view.

**4. Every other axis the issue named also points the same way** — version coupling against a
deliberately pinned image, a second always-on service, and third-party trust. Section "Risk axes"
covers these. No axis came back in hermes-webui's favour.

### What this decision costs

Recording it honestly: the board stays reachable only from the machine running the stack, the
off-desk reply problem stays unsolved, and the Kanban view stays the part of the built-in dashboard
the issue says has been least useful. This decision does not fix those. It declines to fix them
*this way*.

---

## 1. What hermes-webui can reach (sandbox-verified)

All of this was observed by running it, not read from its documentation. [SANDBOX]

**Hermes APIs.** hermes-webui is not an HTTP client of a stable Hermes API. It imports Hermes
Python modules in-process — `run_agent.AIAgent` and the same `hermes_cli.kanban_db` module the CLI
and dispatcher use. [SANDBOX][RESEARCH] Roughly 100 distinct `/api/*` calls were captured in one
ordinary session, including `/api/settings`, `/api/profiles`, `/api/profile/active`, `/api/sessions*`,
`/api/models`, `/api/workspaces`, `/api/projects`, `/api/updates/check`, and a complete kanban
surface: `/api/kanban/boards`, `/api/kanban/board`, `/api/kanban/config`, `/api/kanban/assignees`,
`/api/kanban/stats`, `/api/kanban/events/stream` (long-lived SSE), `/api/kanban/tasks/<id>`,
`/api/kanban/tasks/<id>/log`, and `POST /api/kanban/tasks/<id>/unblock`. [SANDBOX]

**Can it act as a profile?** Yes — confirmed by action, not inference. The unblock call actually
flipped a task from `blocked` to `ready` in the underlying SQLite board and recorded a canonical
`unblocked` event; the effect was cross-checked from a separate shell with `hermes kanban show`.
[SANDBOX] Beyond the board, its chat runs with `toolsets: ["hermes-cli"]` — read from the running
config, not from the README — which is the same full bundle a terminal `hermes chat` session gets:
terminal, code execution, file read/write, cron, delegation. [SANDBOX] The research note reached
the same conclusion from the source, and adds that the only in-chat gate is an approval card for
dangerous *shell commands*, which does not gate file writes, memory edits, cron creation, or
delegation to sub-agents. [RESEARCH] A profile switcher in the UI means whoever is authenticated
acts as whichever profile is selected, with that profile's full reach. [RESEARCH]

**Files.** Two different answers, and the distinction matters. The *workspace browser panel* is
scoped to an explicitly configured directory and never listed anything outside it. [SANDBOX] The
*chat agent's* file and code-execution tools are not scoped to that directory at all — they are the
same unrestricted tools a terminal session has. [SANDBOX] So "the file browser is sandboxed" is
true and nearly irrelevant.

**Caveat on completeness.** The sandbox did not drive a live LLM chat turn end-to-end, because the
scratch profile was deliberately given no model credentials. The toolset *grant* was directly
observed; a tool call being executed through it was not. [SANDBOX] I treat the grant as sufficient
evidence of reach, and flag it as an assumption below rather than overstating it.

**Two observations neither the issue nor the research note anticipated.** First, opening
hermes-webui's page causes the **browser** to fetch seven assets from `cdn.jsdelivr.net`
(xterm, prism, katex) — all 200 OK. The server itself only ever talks to 127.0.0.1, so this is
third-party egress from the tab, not the host reaching out; the built-in dashboard made zero
non-localhost requests because its assets are bundled. [SANDBOX] Second, and more serious:
hermes-webui resolves which board to show through `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD`
environment pins, *not* from `HERMES_HOME`. Launched from a shell that already had those pins set,
it came up labelled "Board: Default" while actually displaying the live board. No writes occurred,
and it was corrected immediately. [SANDBOX] Following hermes-webui's own documented isolation
recipe is therefore not sufficient to guarantee it is pointed where you think it is — its docs do
not mention those variables. This is a sharp edge that was hit in practice, not a hypothetical.

## 2. The comparison the issue asked for: find a blocked card, read its reason, clear it

Run independently on each UI, against a throwaway board with realistically-worded blocked cards
(e.g. "Waiting on the security team to confirm the new staging key scope before I rotate it").
[SANDBOX]

| | hermes-webui | Built-in dashboard |
|---|---|---|
| Finding it | Open Kanban tab, scan the Blocked column for a ⊘/BLOCKED marker | "⚠ 1 task needs attention" banner, clickable, jumps straight to it |
| Reading the reason | One click; reason appears as the summary line and verbatim in Comments | One click; reason in a dedicated "Final Result (run summary)" field, separate from the event history |
| Clearing it | One "Unblock" button, no confirmation, immediate (`POST .../unblock`) | One "Unblock" button, no confirmation, immediate (`PATCH ... status=ready`) |
| Audit trail | Canonical `unblocked` event | Same canonical `unblocked` event |

**Finding** goes to the dashboard: the attention banner is a direct shortcut to precisely this job,
and hermes-webui has no aggregate call-out, which is fine with one card and worse with twenty.
**Reading** is roughly equal; the dashboard's dedicated field is more scannable than a comment
stream, but that is cosmetic. **Clearing** is indistinguishable from the operator's seat. The
different endpoints underneath are a maintenance-surface difference, not a user-facing one.
[SANDBOX] Screenshots of both card-detail and post-unblock states were captured as evidence.

## 3. Risk axes

**Third-party trust and blast radius.** ~19,018 stars (verified via the GitHub API, confirming the
issue's "~19k"), MIT licensed, 2,747 forks, 951 open issues/PRs, 326+ contributors. [RESEARCH] The
popularity is real; it is also not the question. The question is reach, and the reach is the
selected profile's entire CLI toolset plus direct board mutation (§1). A supply-chain compromise,
an XSS, or a bug anywhere in that ~100-endpoint surface is a compromise of an agent that holds
repository credentials. The built-in dashboard's kanban plugin has comparable *board* mutation
reach, but it does not bundle a full agent chat session into the same page by default. [SANDBOX]
One maintenance caveat to keep in proportion: commit recency could only be verified to
2026-08-26, the repo's reported `pushed_at` of 2026-10-18 did not reconcile with the commits API,
and issue-response latency could not be measured at all because the API rate-limited. [RESEARCH]
That is an unresolved gap, not a finding of neglect.

**Password is not safe exposure.** hermes-webui's own startup banner says it plainly: *"No password
set. Any process on this machine can read sessions and memory via the local API."* [SANDBOX] And
its `.env.docker.example`: *"REQUIRED if you expose the container on anything other than 127.0.0.1.
Without a password, anyone who can reach the port can run commands as the agent."* [RESEARCH] Its
`docs/remote-access.md` then instructs `HERMES_WEBUI_HOST=0.0.0.0` with a password for
Tailscale/phone access. [RESEARCH] That is the opposite posture from the built-in dashboard, whose
unauthenticated-but-loopback-only stance is coherent precisely because it never claims to be safe
off the host. The password is positioned by its own authors as the *enabler* of wider exposure, not
as a substitute for not being exposed — and what it would be gating is not a view, but shell, file,
cron and delegation access. **This record neither recommends nor authorizes widening exposure of
either tool.**

**Hermes-version coupling.** This repository pins its Hermes image deliberately. hermes-webui does
not pin a Hermes Agent version on either install path: `bootstrap.py` installs hermes-agent's
`main` branch, and its two-container compose uses the floating `nousresearch/hermes-agent:latest`.
[RESEARCH] Because chat runs against Hermes internals in-process — module layout, the exact
`AIAgent(...)` constructor signature, `run_conversation(..., task_id=...)`, the `state.db` schema —
an upstream refactor can break it even when Hermes's documented API is untouched. It has already
happened once: `api/agent_compat.py` exists as a "compatibility-only" shim for Hermes names that
moved. [RESEARCH] The project's own RFC calls the arrangement *"a compatibility bridge, not the
long-term API contract."* [RESEARCH] The sandbox saw this mechanically: bootstrap had to probe for
an interpreter that could import `run_agent.AIAgent` before it would start, and it worked only
because the host's install happened to be shaped the way hermes-webui expected. [SANDBOX] Adopting
it means an upgrade has two things to satisfy instead of one, with the second one unpinned by
design.

**One more service.** A second always-on HTTP server beside the gateway and the dashboard, with its
own state directory, session and attachment storage, PID and log files, `ctl.sh` lifecycle, and its
own update check firing on every page load. [SANDBOX] In fairness its footprint is modest — pure
Python and vanilla JS, no bundler or runtime build step [SANDBOX] — and no published image size was
found to measure it against [RESEARCH]. But the standard this repo set when the notifier went from
4.5GB to 451MB was that a service earns its size; here the size is not the problem, the *second
lifecycle* is: another thing to run, upgrade, back up, and reason about during an incident, for a
job the existing service already does slightly better.

## 4. Non-goals

- **Replacing the notifier.** Chat stays the alert path. Out of scope per the issue.
- **Any exposure beyond localhost.** See below.
- **A general feature comparison.** The issue named one job; this record evaluates that job and
  declines to accumulate a feature matrix around it.
- **Judging hermes-webui as a project.** It appears competent and well-built. This record is about
  fit here, not quality there.
- **Implementing anything.** No code, compose, or port changes accompany this record.

## 5. Exposure: explicitly out of scope

**This record does not authorize exposing anything beyond localhost.** Nothing in the evaluation
was exposed beyond 127.0.0.1 at any point: both tools were bound to loopback for the entire
sandbox run, verified via `/proc/net/tcp` as well as their startup banners, and no password was
configured on either. [SANDBOX] The built-in dashboard remains localhost-only and unauthenticated,
which is the posture it was designed for.

Any future change to that — binding to `0.0.0.0`, a Tailscale or reverse-proxy bind, or publishing
a port off the host — is a **separate decision requiring its own record and its own human
approval**, for either tool. Keeping the built-in dashboard must not be read as implying its
localhost binding is now safe to relax, and nothing here should be cited as precedent for doing so.

## When to re-open this

Re-open if any of these change — otherwise this is settled, and "someone found the project again"
is not a reason to redo the work:

1. hermes-webui ships a genuinely scoped mode: a read-only or restricted-toolset session that does
   not carry shell, file, cron and delegation access by default.
2. It talks to Hermes over a versioned, documented API instead of in-process imports — i.e. its own
   RFC's "long-term API contract" lands — and can be pinned alongside this repo's Hermes image.
3. A human separately decides, on its own merits and in its own record, that the board should be
   reachable off-host. That decision would legitimately reopen the interface question, and should
   reopen it *after* the exposure question is settled, not before.
4. The built-in dashboard's Kanban view regresses or is removed upstream.

## Assumptions and unverified claims

Every inference the issue and the source reports did not state outright:

1. **Docs repository, path, and filing.** No longer inferred: the repository was named
   explicitly as `Zhachory1/docs` and its tree was read directly at commit `a3e1ae8`. It is a
   PARA-structured repository (`projects/`, `areas/`, `resources/`, `archives/`) whose `projects/`
   directory was empty apart from `.gitkeep`, so this is the first project folder in it and there
   is no pre-existing decision-record example to match. Format follows the conventions stated in
   its `README.md` and `AGENTS.md`: the project-folder shape names `decisions.md`, frontmatter
   carries id/status/owner/card/prs, and the title states the answer rather than the topic. What
   remains a judgment call rather than a verified fact is `projects/` versus `areas/` — see
   decision 4 in the review header.
2. **This pull request comes from a fork, not a branch on the docs repository.** The credential
   available to me authenticates as `zhach1`, which has read-only access to `Zhachory1/docs`
   (`push: false`), so a branch could not be pushed to the repository directly. The record is
   therefore proposed as a standard cross-fork pull request from a fork of the repository. If the
   intent was for these documents to land on branches of `Zhachory1/docs` itself, the account
   needs write access and this PR can be re-opened from a direct branch; nothing about the
   document's content depends on which way it arrives.
3. **Toolset reach was observed as a grant, not exercised.** The sandbox read `toolsets:
   ["hermes-cli"]` from the running config but did not execute a tool call through a live chat
   turn, because no model credentials were given to the scratch profile. I treat the grant as
   establishing reach. If someone considers that insufficient, the remaining step is to run one
   authenticated chat turn and watch a shell command execute.
4. **Maintenance health is partially unverified.** Commit recency confirmed only to 2026-08-26
   (a `pushed_at` of 2026-10-18 did not reconcile); issue-response latency not measured at all
   (API rate-limited); no CODEOWNERS found, so "single owner plus wide community PR pipeline" is
   read from the commit pattern, not stated fact. None of these are load-bearing for the decision,
   which rests on reach, coupling, and the wash on the comparison — but if the decision were
   overturned to "adopt", they would become load-bearing and should be re-checked.
5. **Footprint is characterized, not measured.** No published image size exists; "modest footprint"
   rests on the observed absence of a runtime build step and of bundled ML dependencies, not on a
   number. The 4.5GB→451MB notifier figure is quoted from issue #2 and not independently verified.
6. **The sandbox ran its own instances on its own ports** (127.0.0.1:8788 and 127.0.0.1:9129)
   against a throwaway board, not this stack's dashboard on 127.0.0.1:9219. The UX comparison is
   therefore of the same software, not of this deployment's live instance.
7. **Non-Docker install path only.** hermes-webui was installed via `python3 bootstrap.py` against
   an existing agent directory; the Docker paths were not exercised. The version-coupling findings
   for the Docker path come from reading its compose files, not from running them. [RESEARCH]
8. **The CDN egress was not traced further.** Seven jsdelivr assets were observed loading; whether
   they can be vendored or disabled by configuration was not investigated.

## Out of scope but worth someone's attention

Noticed during this evaluation, not acted on, not part of this record's decision:

- **The board-resolution sharp edge is not hermes-webui's alone.** Any tool resolving the board
  through ambient `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD` pins can silently display the live board
  while labelled otherwise. The sandbox hit this. It may deserve its own hardening card
  independently of this decision.
- **The off-desk reply need remains unsolved** and is the one genuinely unaddressed item from
  issue #2's motivation. See decision #2 in the review header.
- **The two tools are not mutually exclusive.** hermes-webui ships a compose layout running both
  against the same Hermes home, with a documented history of lock-file races between them.
  [RESEARCH] Not relevant under this decision; relevant immediately if it is overturned.

## Sources

- **[SANDBOX]** — Empirical sandbox run, 2026-10-20: hermes-webui installed from `master` HEAD via
  `python3 bootstrap.py --host 127.0.0.1`, run against a purpose-built throwaway kanban board
  alongside the built-in `hermes dashboard`, both loopback-only; both UIs driven with
  Playwright + Chromium with full network capture; board effects cross-checked independently via
  `hermes kanban show`; port bindings confirmed via `/proc/net/tcp`; sandbox torn down afterwards.
  Recorded on the ralph board as card `t_18025034`, with four screenshots attached as evidence.
- **[RESEARCH]** — Read-only source and documentation review, 2026-10-20, of
  `github.com/nesquena/hermes-webui` at `master`: README, ARCHITECTURE.md, docs/docker.md,
  docs/remote-access.md, Dockerfile, requirements.txt, LICENSE, `.env.docker.example`, both
  compose files, plus the GitHub repository and commits APIs. Recorded on the ralph board as card
  `t_2e0a9333`.
- **Issue** — Zhachory1/ralph#2, "Evaluate hermes-webui as the interface, in place of the built-in
  dashboard", including its acceptance criteria and its "Not in scope" section.