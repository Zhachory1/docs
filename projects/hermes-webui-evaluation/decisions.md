---
id: dec-hermes-webui-evaluation
title: Keep the built-in dashboard as the monitor; adopt hermes-webui as the chat space
status: in review
owner: Zhachory1
card: t_f34790a4
source: Zhachory1/ralph#2
revision: v4
date: 2026-10-20
prs:
  - Zhachory1/docs#1
---

# Keep the built-in dashboard as the monitor; adopt hermes-webui as the chat space

Status: **in review** — proposed, awaiting human ratification at the decision gate.
Decision: **Keep the built-in dashboard as the monitor. Adopt hermes-webui as the chat space,
running alongside it on the same machine.** Three conditions ride with the adoption: pin the two in
matched pairs, bind to localhost only, and clear the board environment pins. See "Conditions of
adoption".
Source: Zhachory1/ralph issue #2, "Evaluate hermes-webui as the interface, in place of the built-in
dashboard".
Revision: v4 (2026-10-20). v3 named the option "adopt hermes-webui for chat only, alongside the
dashboard" and declined it on two grounds. You have answered both from use, and the answer flips.
This revision reverses the recommendation — see "Changed since v3".

---

## Review header

**Decisions needed from you (4).**

1. **Ratify the reversal, and the three conditions with it.** Merging adopts hermes-webui as the
   chat space *and* commits to matched-pair pinning, localhost-only binding, and cleared board
   pins. The conditions are not decoration: the first one is a standing obligation that does not
   expire, and I would rather you ratify it knowingly than discover it at the first Hermes bump.
2. **When the matched pair breaks, what gives?** A Hermes upgrade can land before hermes-webui is
   known-good against it, because hermes-webui drives Hermes through in-process imports of internal
   module layout rather than a versioned API ([RESEARCH], [SANDBOX]). Two answers are defensible:
   hold the Hermes upgrade until the pair is re-qualified, or take the Hermes upgrade and accept
   that the chat space is down or stale until it catches up. This record cannot infer which you
   want, it is the single recurring cost of adoption, and the follow-up deployment card needs the
   answer before it designs anything.
3. **"Chatting with the orchestrator" — between runs, or steering a card mid-run?** Carried from v3
   decision 3, still unanswered, and adoption does not settle it. Neither tool attaches to a
   dispatched worker while it runs; both spawn or resume their own session ([CODE], [RESEARCH]).
   Adopting hermes-webui gets you a better *between-runs* chat. If what you actually want is to
   interrupt a running card and steer it, nothing here delivers it and it is its own card.
4. **Where this file lives.** Still `projects/hermes-webui-evaluation/decisions.md`, because the
   card said so. This revision strengthens the case to move it: an evaluation ends, but matched-pair
   pinning is a responsibility that runs indefinitely, which is what `areas/` is for in this
   repository's own rule. Move it on merge if you agree.

**Council dissent.**

- **You versus v3, on the chat-only option — conceded in full, and v3 was wrong on both grounds.**
  v3 declined it because the surface overlapped one you already owned and because nobody had run a
  chat turn on either tool. You have used the dashboard's chat tab and found it not good enough.
  That retires both: an overlap with a surface you will not use is not an overlap, and your use *is*
  the missing evidence. **Resolution: reversed.** v3's own overturn criterion named this exact
  outcome, so this is the record working as written rather than changing its mind.
- **The code read versus your experience, on "I cannot run multiple sessions" — the one unresolved
  conflict.** The dashboard's bundle ships a `SessionsPage` and its chat takes a `resume=` parameter
  ([CODE], §1), so something session-shaped exists in it. You report you cannot run multiple
  sessions. **Resolved in your favour**, because a capability a daily user cannot find or use is not
  a capability, and because nothing in this decision turns on which of the two readings is right.
  The detail is flagged as Assumption 3 — it matters only if someone later proposes fixing the
  dashboard's chat instead, which is not what you asked for.
- **Agent reviewer (t_b493c10f), promoted rather than carried.** Its finding was that version
  coupling was sandboxed only on the non-Docker path while this stack deploys via Docker Compose.
  Under v1–v3 that was a caveat on a cost being used to decline. Under v4 it is a caveat on a cost
  we are choosing to pay, on the install path we will actually use, which makes it more load-bearing
  than it has ever been. See Assumption 1 and decision 2.

**Scope: S.** One pull request, documentation only: this file, on the existing branch
`decision/hermes-webui-evaluation`. No code, no compose change, no port, no service — the
deployment is a separate card after this merges, and designing it here is a non-goal.

**Changed since v3.**

- **The decision is reversed.** v3: keep the dashboard as monitor and chat surface, do not adopt.
  v4: keep the dashboard as the monitor, adopt hermes-webui as the chat space. The title and the
  decision line changed with it.
- **The two grounds v3 declined on are withdrawn by name** (§5), so the old text cannot be cited
  later for an argument it no longer makes.
- **v3's "the gain is off-desk, and off-desk is ruled out" reasoning is withdrawn too**, and that is
  the part v3 got most wrong. Your four complaints — no multiple sessions, hard to read, hard to
  change the design, not built for day-to-day work — are all complaints from a desk. The ergonomic
  gain was mislocated, not merely outweighed.
- **The surviving costs are recorded as conditions of adoption, not as reasons against** (new
  section). Same three facts, same citations, opposite role in the argument.
- **Your statements are facts in the record now, not assumptions.** Everything you have said in
  review moved out of Assumptions into "What you have settled". Assumptions is down from ten entries
  to seven, and the human-sourced ones are gone from it entirely.
- **§1 keeps the dashboard chat-tab finding but it no longer carries the decision.** The tab exists;
  you have used it; it lost on the merits. Recording that it exists still matters, because it is why
  the overlap is permanent and why adoption adds a surface rather than replacing one.
- **Decision 2 of v3 — "does your dashboard show a Chat tab?" — is answered yes and retired.**

---

## Decision

**Keep the built-in dashboard as the monitor. Adopt hermes-webui as the chat space, alongside it.**

Plainly: the dashboard stays where it is and keeps doing what it does — showing the board, the
cards, and the "needs attention" banner. hermes-webui gets installed on the same machine as the
place you go to talk to the agents. Neither replaces the other. Nothing is removed.

### Why this is the answer

**Because the chat surface we already own was tried and rejected by its user.** The built-in
dashboard ships an always-enabled Chat tab that runs a full Hermes session in the browser, scoped to
any profile and able to resume a past one ([CODE], §1). v3 leaned on that fact to decline
hermes-webui: the slot was not empty. You have since used that tab and reported what using it is
like — you cannot run multiple sessions, it is hard to read, it is hard to change the design, and
the suite as a whole is not built for working with the agents day to day. A surface that exists and
goes unused does not fill the slot; it only occupies it.

**Because that is the better-shaped chat, and shape is the whole point here.** hermes-webui presents
conversation as a chat application — composer, session list, model and reasoning surfaces,
attachment storage, profile switcher ([SANDBOX], [RESEARCH]) — against the dashboard's
`hermes --tui` bridged into xterm.js ([CODE]). On the one axis issue #2's framing cares about, that
is not a close comparison, and v3 said so while still declining.

**Because the overlap costs nothing.** Two chat surfaces where one is unused is one chat surface and
a dead tab. The dashboard's tab cannot be switched off in any case — it is a module constant, not a
feature flag ([CODE], §1) — so the overlap is permanent whatever is decided, and declining
hermes-webui does not remove it.

**And because this is the outcome the previous revision wrote down for itself.** v3's "How to
overturn this cheaply" said: open the tab, and if you dislike it enough to avoid talking to the
system, that is the finding and the answer flips to the chat-only option. The step was taken and
that is what it returned.

### What this does not buy

Stated plainly, so ratification is not a surprise. Adoption buys a better composer for conversations
you can already have. It does not buy reach into a running worker (§3), it does not make anything
reachable off this machine, and it does not make the board view better — the board stays the
dashboard's job. If after a month the honest report is "I talk to the agents about as much as
before", the back-out triggers below are how to undo it.

## Conditions of adoption

These are the costs the evidence actually supports. They do not argue against adoption; they are
what adoption obliges. The deployment card inherits all three as acceptance criteria.

1. **Pin hermes-webui and Hermes in matched pairs.** hermes-webui runs the agent in-process —
   `run_agent.AIAgent` imported directly, `run_conversation(..., task_id=...)` — against internal
   module layout rather than a versioned API ([RESEARCH]). A pinned hermes-webui is therefore pinned
   to a *particular* Hermes: every Hermes image bump needs a matching, re-qualified hermes-webui,
   and pinning the image does not remove this, it only makes the pairing explicit. This is a
   standing obligation with no end date, it is the single recurring cost of this decision, and
   decision 2 asks what happens the first time the pair cannot be kept.
2. **Bind to localhost only, and keep it there.** Both tools stayed loopback-only for the entire
   sandbox run, verified via `/proc/net/tcp` and their startup banners, with no password on either
   ([SANDBOX]). hermes-webui's chat carries the full `hermes-cli` toolset and a profile switcher
   ([SANDBOX], [RESEARCH]), so whoever reaches it acts as the profile they select. Issue #2 puts
   exposure out of scope and you have said off-host is a non-starter; this record adopts the tool on
   that basis and authorizes nothing beyond 127.0.0.1. Exposing it later is a separate decision, and
   it is the decision that would make the missing password matter.
3. **Clear `HERMES_KANBAN_DB` and `HERMES_KANBAN_BOARD` in its environment.** hermes-webui resolves
   which board to display from those environment pins, not from `HERMES_HOME`, and its documentation
   mentions neither. Launched from a shell that had them set, it came up **labelled "Board: Default"
   while actually displaying the live board**; no writes occurred and it was corrected immediately
   ([SANDBOX]). The consequence for us is specific: the documented isolation recipe — point
   `HERMES_HOME` at a different home — does not cover these two variables, so an inherited pin
   silently mislabels the live board. The service environment must clear both; if hermes-webui is
   ever meant to show a particular board, set them explicitly there rather than inherit them.

## 1. The conversational surfaces this system already has (verified)

Kept because the overlap is permanent and the record should say so once, with citations. Read from
the source of the Hermes install this worker runs on — version `0.22.2`, commit
`d923a5eb487a17c0a68801bba9ef0f56c81040a8`. [CODE]

**The dashboard has a Chat tab, and it is not optional.**
`/opt/hermes/hermes_cli/web_server.py` lines 461-464: *"In-browser Chat tab (`/chat`, `/api/pty`,
`/api/ws`): always enabled"*, implemented as a module constant rather than a feature flag. The
single-page app ships the matching pages: `web_dist/assets/ChatPage-RJV46vkP.js` and
`SessionsPage-Dp4jBTPn.js`. [CODE] You have confirmed the tab is present on your build, which
retires v3's decision 2 — and because it cannot be disabled, adopting hermes-webui adds a surface
rather than replacing one.

**What that tab runs.** `/opt/hermes/hermes_cli/web_server_chat.py`, `_resolve_chat_argv`
(lines 382-470): it spawns `hermes --tui` behind a pseudo-terminal and bridges it to xterm.js. A
`profile` parameter scopes the chat by pointing `HERMES_HOME` at that profile's directory — the
source calls it *"the same propagation `hermes -p <name>` performs"* — and a `resume` parameter
reattaches to a past session, resolved to its newest descendant. [CODE] It is a terminal emulator,
not a message composer, with everything that implies about reading it on a small screen.

**The board is the other conversational channel, and the only durable one.** A comment on a card is
delivered into the next worker's context: the `worker_context` this card handed me opens with the
card body and carries the full comment thread and each parent's handoff. [BOARD] That is how your
review comments reached each revision of this record, including this one. A chat window has
immediacy; the comment thread has memory and a guaranteed reader. Adopting a chat space does not
change that, and should not be expected to.

**Nothing else is connected.** The gateway serves ten profiles, `ralph-orchestrator` among them, but
its only connected platform is the local API server on `127.0.0.1:8642`, and the channel directory
lists no messaging platform at all (`/home/hermes/.hermes/channel_directory.json`:
`"platforms": {}`). [CODE]

## 2. What hermes-webui's chat is, and what it can reach

From the sandbox run and the source review, not from its README.

**It is a chat application, and it runs the agent in-process.** Chat is backed by
`run_agent.AIAgent` imported directly, with `run_conversation(..., task_id=...)` — so a conversation
can be bound to a kanban task, the one chat affordance in either tool that points at the board.
[RESEARCH] Around it the sandbox observed a session surface (`/api/sessions*`,
`/api/sessions/gateway/stream`), `/api/models`, `/api/reasoning`, `/api/transcribe/capability`, its
own attachment and session storage, and a profile switcher — roughly 100 distinct `/api/*` calls in
one ordinary session. [SANDBOX][RESEARCH] Issue #2 credits it with a mobile layout; that was not
exercised, and under the local-only constraint it is not what this decision buys.

**Its reach is a full agent session.** Chat runs with `toolsets: ["hermes-cli"]`, read from the
running config rather than the README — the same bundle a terminal `hermes chat` gets: terminal,
code execution, file read/write, cron, delegation. [SANDBOX] The workspace browser panel is scoped
to a configured directory; the chat agent's file and code-execution tools are not scoped to it at
all. [SANDBOX] This is not a mark against it — the dashboard's Chat tab spawns the same kind of
session with the same toolset under a profile you choose [CODE], and the mitigation for both lives
in base Hermes, in sandboxes and approved tools, as you said in review. It is why condition 2 exists.

**It can act on the board, confirmed by action.** `POST /api/kanban/tasks/<id>/unblock` actually
flipped a task from `blocked` to `ready` in the underlying SQLite board and recorded a canonical
`unblocked` event, cross-checked from a separate shell. [SANDBOX] Its kanban integration is more
mature than its documentation suggests. This record still leaves the board to the dashboard; it
notes the capability because condition 3 is about making sure that capability points at the board
you think it does.

**What nobody has done: run a conversation turn on it.** The sandbox profile had no model
credentials, by design, so hermes-webui's chat was characterised from its running config and its
observed endpoints, never used. [SANDBOX] See Assumption 2 — this is now the weakest point of an
*adopt* decision, where in v3 it was an argument for declining.

## 3. The comparison under this frame: having a conversation with the orchestrator

| | hermes-webui chat | Dashboard Chat tab | Card comment |
|---|---|---|---|
| Form | Chat application: composer, session list, reasoning and model surfaces, attachments [SANDBOX] | `hermes --tui` in xterm.js, bridged over `/api/pty` [CODE] | Markdown in a thread |
| Talk as the orchestrator | Profile switcher [RESEARCH] | `profile=` scopes `HERMES_HOME`, as `hermes -p` does [CODE] | n/a — you address the card, the next run reads it |
| Continue a past session | Session list [SANDBOX] | `resume=`, resolved to newest descendant [CODE]; in use you report you cannot run multiple sessions | The thread is the memory |
| Reach a worker **mid-run** | No — spawns its own session [RESEARCH][SANDBOX] | No — a scoped chat spawns its own [CODE] | No — delivered at the next run |
| Reaches a future run | Only if written to the board | Only if written to the board | Yes, in `worker_context` [BOARD] |
| Away from this machine | Mobile layout, but off-host is a non-starter [ISSUE] | Same constraint | Same constraint |
| Agent capability | Full `hermes-cli` toolset [SANDBOX] | Full session, same toolset [CODE] | Whatever the next worker has |
| Standing cost | Matched-pair Hermes pinning, one more lifecycle [RESEARCH] — now condition 1 | None — ships in the image you pin | None |

**Verdict: hermes-webui wins the form row, and the form row is the one being bought.** The rest of
the table is a wash or a tie, and v3 read that as "nothing to gain". The correction is that form is
not a tiebreaker here — it is the deciding property, because the surface with the worse form is the
one its user stopped using. A chat you avoid scores zero on every other row too.

Two limits stand. Neither tool reaches a worker mid-run, so decision 3 is not settled by adopting
either. And the board column remains the only durable channel: anything that must reach a future run
still goes on the card, chat space or no chat space.

## 4. The monitor comparison, unchanged and still the dashboard's

On the job issue #2 named — find a blocked card, read its reason, clear it — the two are a **wash**:
reading and clearing are indistinguishable from the operator's seat, both land the same canonical
`unblocked` event, and the dashboard's "⚠ 1 task needs attention" banner is a small genuine
convenience on *finding*, worth one sentence and no more. [SANDBOX] Screenshots of both card-detail
and post-unblock states are attached to card `t_18025034`. The dashboard keeps the monitor because
nothing displaced it, and because you have said to keep it there. Your standing complaint from issue
#2 — that its Kanban view "has been least useful during this work" — remains untested across a
working week and is unaffected by this decision; if it bites, it is a board decision, not a chat one.

## 5. What this decision no longer rests on

Withdrawn by name so the earlier text cannot be cited for arguments it no longer makes.

**Retired in v2, still retired:** blast radius (not a property of the UI; mitigated in base Hermes by
sandboxes and approved tools), password versus exposure (moot while nothing binds past localhost),
third-party and always-on (the current UI is already both), the unpinned image (withdrawn as a
reason, kept as a requirement — now condition 1), and the banner (demoted to a sentence).

**Retired in v3:** "the dashboard is the better monitor" was never a reason to decline a chat tool.

**Retired here, in v4 — these are v3's own reasons for declining the chat-only option:**

1. **"It overlaps a surface you already own."** Withdrawn. The surface is unused, and an overlap
   with a surface nobody uses costs nothing. The overlap is also unavoidable: the dashboard's Chat
   tab cannot be turned off (§1).
2. **"Nobody has run a chat turn on either tool."** Withdrawn as stated. You have run plenty on the
   dashboard's chat, and that experience is the evidence v3 said was missing. It remains true that
   nobody has run a turn on *hermes-webui* — that is Assumption 2, not a reason to decline.
3. **"The ergonomic gain is concentrated off the desk, and off the desk is ruled out."** Withdrawn,
   and this was v3's worst reasoning rather than merely its weakest. Your complaints are desk
   complaints. The gain was in the wrong place, not just too small.

## 6. Non-goals

- **Planning or building the deployment.** This record decides *what* and states the conditions it
  must satisfy. How hermes-webui gets installed, pinned, supervised, and launched is a separate card
  once this merges. No compose file, no unit, no port assignment, no pinning mechanism here.
- **Replacing or removing anything.** The dashboard stays the monitor; its Chat tab stays because it
  cannot be removed; the notifier stays the alert path.
- **Any exposure beyond localhost.** Condition 2 and §7.
- **Building a mid-run steering channel.** Decision 3 may be what you want; neither tool offers it
  and this record does not design it.
- **Fixing the dashboard's chat instead.** You asked for a chat space, not a repair; and the
  dashboard chat is a terminal in a tab by construction, so "make it a chat application" is a
  different project.
- **Re-litigating the base system's agent-capability model.** Sandboxes and approved tools are the
  mitigation layer and are not this record's business.

## 7. Exposure

Nothing in this evaluation was exposed beyond 127.0.0.1: both tools stayed loopback-only for the
entire sandbox run, verified via `/proc/net/tcp` and their startup banners, with no password on
either. [SANDBOX] This record adopts hermes-webui on localhost only (condition 2) and authorizes
nothing further; exposing it is its own decision, and the one that would make the absent password
load-bearing.

## When to back this out

Adoption is reversible: nothing migrates into hermes-webui, and removing it leaves the dashboard
exactly as it is today. Back it out if one of these holds.

1. **The matched pair becomes the thing you fight.** If re-qualifying hermes-webui against a Hermes
   bump costs more than the chat space is worth, or blocks a Hermes upgrade you want — condition 1
   and decision 2.
2. **The chat space goes unused too.** The same test that retired the dashboard's tab applies here:
   after a working month, if you are not talking to the agents appreciably more, the form argument
   was wrong and the cost has no return.
3. **hermes-webui moves to a versioned, documented API** — its own RFC's "long-term API contract" —
   in which case condition 1 relaxes. That is a reason to revisit the conditions, not the decision.

## What you have settled

Facts in this record, stated by you in review, no longer inferences.

- **hermes-webui is built for humans chatting with the orchestrator, not as a monitor.** Your
  framing; the frame of this record since v3.
- **The dashboard's Chat tab exists on your build, and it is not good enough.** You cannot run
  multiple sessions, it is hard to read, it is hard to change the design, and the suite as a whole
  is not built for working with the agents day to day.
- **The dashboard stays the monitor.** Explicitly, alongside the adoption, rather than by default.
- **Off-desk replies are already handled**, by the notifier in the stack plus a poller watching
  messages.
- **Off-host access is a non-starter.** Only the local machine runs this.
- **The risk appetite is yours, on your machine.**
- **Blast radius is mitigated in base Hermes**, by sandboxes and approved tools, and does not depend
  on the UI.
- **The current UI is already a third-party always-on service.**
- **An unpinned image is a thing to pin**, not a reason to decline.

## Assumptions

Every inference the brief and the sources did not state, ordered by weight.

1. **Load-bearing: the Docker install path was never exercised, and that is the path we will use.**
   hermes-webui was installed via `python3 bootstrap.py` against an existing agent directory; its
   Docker paths were read from compose files, not run [RESEARCH], while this stack deploys via
   Docker Compose. Condition 1's cost is therefore inferred rather than observed for the real path.
   Raised by the agent reviewer against v1; it mattered less when the cost was a reason to decline
   and matters most now that it is a cost we are accepting. The deployment card should observe it
   rather than re-argue it.
2. **Load-bearing: hermes-webui's chat is as good as its shape suggests.** No conversation turn was
   ever run on it — the sandbox profile had no model credentials, by design [SANDBOX] — so the case
   for it rests on its form as read from endpoints, bundles, and config, plus your judgement of the
   alternative. Adoption is the cheap way to test this; back-out trigger 2 is what to do if it
   fails.
3. **Moderate: I read "I cannot run multiple sessions" as about the dashboard chat in use, not as a
   claim that no session machinery exists.** The bundle ships `SessionsPage` and chat takes
   `resume=` [CODE]. Nothing here turns on the difference, but if anyone later proposes fixing the
   dashboard's chat rather than adopting a chat space, this is the line to check first.
4. **Moderate: neither chat surface can reach a running worker.** Inferred from structure, not from
   a test: the dashboard's profile-scoped chat spawns its own session [CODE] and hermes-webui
   constructs its own `AIAgent` in-process [RESEARCH]. I did not try to interrupt a live dispatched
   worker from either. Decision 3 depends on this being right.
5. **Moderate: "the chat space" means a surface for you, not for the agents.** I read your request
   as a place where you talk to the system; nothing in this record gives agents a new channel to
   each other, and the board stays the inter-run channel.
6. **Low weight: maintenance health is partly unverified.** Commit recency confirmed only to
   2026-08-26; issue-response latency not measured (API rate-limited); no CODEOWNERS found.
   [RESEARCH] This was not load-bearing while the record declined; adopting makes it worth a glance
   before the deployment card pins a revision.
7. **Low weight: the CDN egress was not traced further.** Opening hermes-webui's page makes the
   browser fetch seven assets from `cdn.jsdelivr.net` (xterm, prism, katex); the built-in dashboard
   bundles its assets and made zero non-localhost requests. [SANDBOX] Whether those can be vendored
   or disabled was not investigated, and under condition 2 it is a privacy question rather than a
   reachability one.
8. **This pull request comes from a fork.** The credential available to me authenticates as
   `zhach1`, which has `push: false` on `Zhachory1/docs`, so the branch lives on a fork and PR #1 is
   cross-fork. Nothing about the content depends on it.

## Observations — out of scope, not acted on

- **There is no messaging platform connected to this stack.** The gateway serves ten profiles but
  its only connected platform is the local API server; the channel directory is empty. [CODE] If
  "talk to the orchestrator from wherever you are" ever becomes a goal, wiring a platform to the
  gateway is a smaller change than a web UI, and it is a different card.
- **The board-resolution hazard is not hermes-webui's alone.** Any tool resolving the board from
  ambient `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD` pins can display the live board while labelled
  otherwise. Condition 3 handles it for this deployment; hardening it generally deserves its own
  card.
- **The sandbox report contained one claim this record had to correct** — that the dashboard bundles
  no chat. The report is otherwise solid; a report's absences are weaker evidence than its
  observations, which is a lesson about consuming research on this board rather than a complaint.
- **`GH_TOKEN` is not forwarded into dispatcher-spawned workers** (only `GIT_AUTHOR_NAME`/`EMAIL`
  are), so an authoring worker has to recover it from the gateway process environment to push. Worth
  fixing if authoring workers are expected to open pull requests routinely.

## Sources

- **[CODE]** — Hermes Agent source and runtime state on the host running this worker, read
  2026-10-20 at version `0.22.2`, commit `d923a5eb487a17c0a68801bba9ef0f56c81040a8`:
  `/opt/hermes/hermes_cli/web_server.py` (lines 461-464),
  `/opt/hermes/hermes_cli/web_server_chat.py` (`_resolve_chat_argv`, lines 382-470),
  `/opt/hermes/hermes_cli/web_dist/assets/ChatPage-RJV46vkP.js` and `SessionsPage-Dp4jBTPn.js`,
  `/home/hermes/.hermes/gateway_state.json`, `/home/hermes/.hermes/channel_directory.json`.
- **[SANDBOX]** — Empirical sandbox run, 2026-10-20: hermes-webui installed from `master` HEAD via
  `python3 bootstrap.py --host 127.0.0.1`, run against a purpose-built throwaway kanban board
  alongside the built-in `hermes dashboard`, both loopback-only; both UIs driven with
  Playwright + Chromium with full network capture; board effects cross-checked independently via
  `hermes kanban show`; port bindings confirmed via `/proc/net/tcp`; torn down afterwards. Recorded
  on the ralph board as card `t_18025034`, with four screenshots attached.
- **[RESEARCH]** — Read-only source and documentation review, 2026-10-20, of
  `github.com/nesquena/hermes-webui` at `master`: README, ARCHITECTURE.md, docs/docker.md,
  docs/remote-access.md, Dockerfile, requirements.txt, LICENSE, `.env.docker.example`, both compose
  files, plus the GitHub repository and commits APIs. Recorded on the ralph board as card
  `t_2e0a9333`.
- **[BOARD]** — The ralph kanban board: the `worker_context` delivered to this revision's card
  `t_f34790a4` (v3 came from `t_582469df`; v1 and v2 from `t_2f2c6131`), carrying the card body, the
  comment thread, and parent handoffs.
- **[ISSUE]** — Zhachory1/ralph#2, read at its current body, including acceptance criteria and
  "Not in scope".
- **Review of v1–v3** — six line-anchored review comments and four summary comments by `zhach1` on
  `Zhachory1/docs#1`, 2026-10-20. The two comments of 19:47 UTC are the ones this revision answers:
  the dashboard's chat tab exists and is not good enough, and the request to keep the dashboard as
  the monitor while adopting hermes-webui as the chat space. Plus the agent review recorded on card
  `t_b493c10f`.
