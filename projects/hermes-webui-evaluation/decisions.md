---
id: dec-hermes-webui-evaluation
title: Keep the built-in dashboard as monitor and as chat surface; do not adopt hermes-webui
status: in review
owner: Zhachory1
card: t_582469df
source: Zhachory1/ralph#2
revision: v3
date: 2026-10-20
prs:
  - Zhachory1/docs#1
---

# Keep the built-in dashboard as monitor and as chat surface; do not adopt hermes-webui

Status: **in review** — proposed, awaiting human ratification at the decision gate.
Decision: **Keep the built-in dashboard. Do not adopt hermes-webui — not as the monitor, and not
as a second surface for chat.**
Source: Zhachory1/ralph issue #2, "Evaluate hermes-webui as the interface, in place of the built-in
dashboard".
Revision: v3 (2026-10-20). v2 answered "which of these two is the better monitor". You then said
hermes-webui is built for humans *chatting with the orchestrator*, not for monitoring. v3 answers
the question that frame implies. The answer is the same; the reason is different again, and one
load-bearing fact changed — see "Changed since v2".

---

## Review header

**Decisions needed from you (4).**

1. **Ratify or overturn, now on the conversational axis.** The recommendation is unchanged, and it
   is still **weakly held**. What changed is why: the premise that this system has no
   conversational surface does not hold. The built-in dashboard ships an always-enabled in-browser
   Chat tab that runs a full Hermes session, scoped to any profile — including `ralph-orchestrator`
   — and can resume a past session ([CODE], §1). So hermes-webui is not filling an empty slot; it is
   offering a nicer-shaped chat than one you already have and, as far as I can tell, have not
   opened. Merging ratifies; a comment overturns.
2. **Check one thing I could not: does your dashboard on `127.0.0.1:9219` show a Chat tab?** I read
   the Chat tab out of the Hermes install this worker runs on (`0.22.2`, `d923a5e`), and could not
   reach port 9219 from inside the worker container to confirm the pinned image you actually
   publish has it. If it does not, decision 1 is much closer, because then hermes-webui really
   would be adding a surface rather than duplicating one. This is a ten-second check at your desk
   and it is the single fact this revision leans on hardest.
3. **What does "chatting with the orchestrator" mean — between runs, or during one?** Both chat
   surfaces (hermes-webui's and the dashboard's) *start or resume* a session; neither attaches to a
   dispatched worker while it is running ([CODE], §3). If you meant "ask the system things and have
   it act, between cards", you have that today. If you meant "interrupt the orchestrator mid-card
   and steer it", neither tool delivers it, this evaluation cannot settle it, and it should be its
   own card. I could not infer which you meant and the answer changes what gets built next.
4. **Where this file lives.** Still filed as `projects/hermes-webui-evaluation/decisions.md` because
   the card said so; this repository's own rule would move a standing responsibility to `areas/`.
   Move it on merge if you agree. (Unchanged from v1 and v2; you have not commented on it.)

**Council dissent.**

- **You, on PR #1 — the frame is accepted in full, and it was the right correction.** Both earlier
  revisions compared two monitors and scored them on finding a blocked card. If hermes-webui's job
  is conversation, that comparison was answering a question nobody asked. §3 redoes it on the
  conversational axis. **Resolution: conceded on the frame, unchanged on the answer** — and the
  record now says which of its claims the frame retires.
- **The sandbox report versus the code, on whether the dashboard has a chat — the one real evidence
  conflict in this revision.** The sandbox report states the built-in dashboard "does not bundle a
  full agent chat session into the same page by default" [SANDBOX]. The dashboard's own source says
  otherwise: the Chat tab is a module constant set to enabled, not a flag
  (`web_server.py:461-464`), and its single-page app ships `ChatPage` and `SessionsPage` bundles
  [CODE]. **Resolved in favour of the code**, because the sandbox never opened the tab — it was
  driving the kanban comparison the card asked for, and recorded an absence it had not looked for.
  This is exactly the kind of claim a decision should not inherit unchecked, and v2 did inherit it.
  Decision 2 exists because resolving it in favour of the code is still one install removed from
  *your* install.
- **Agent reviewer (t_b493c10f), carried forward.** Its promoted finding stands untouched by this
  reframing: version coupling was sandboxed only on the non-Docker path, while this stack deploys
  via Docker Compose, and coupling is still the surviving cost. See Assumption 3.

**Scope: S.** One PR, documentation only: this file. No code, no compose change, no port, no
service.

**Changed since v2.**

- **New, and load-bearing: the incumbent already has the thing.** §1 records the dashboard's Chat
  tab, profile scoping, and session resume, read from source with paths and line numbers. v2 did
  not know this, and it is why the answer survives the reframing instead of being overturned by it.
- **The comparison is redone on the conversational axis** (§3), replacing "who shows a blocked card
  better" as the deciding comparison. The monitor comparison is demoted to §4 and compressed to a
  single paragraph: it is still true, it is just no longer the question.
- **A third option is named and declined: adopt hermes-webui for chat only, alongside the
  dashboard** (§2, "The option this frame opens"). Neither earlier revision considered it, and
  under this frame it is the strongest version of the case for adoption.
- **The never-tested gap moved.** v2's weak point was that nobody tested the kanban view across a
  working week. v3's is sharper: **nobody ran a single chat turn on either tool**, so the axis this
  decision now turns on is the one the evidence covers least. Promoted from Assumption 2 (low
  weight) to Assumption 2 (load-bearing), inverted by the reframing.
- **"How to overturn this cheaply" is rewritten** around chat rather than around the board, and now
  starts with opening a URL rather than installing anything.
- Your reframing comment is recorded as settled fact, not as my inference ("No longer assumptions").
- The decision itself is **unchanged** and remains explicitly weakly held.

---

## Decision

**Keep the built-in dashboard. Do not adopt hermes-webui, as monitor or as chat surface.**

### The question this frame implies, and the answer

You asked, in effect, two things: should this system have a conversational surface to the
orchestrator, and is hermes-webui a reasonable one.

**Should it: yes, and it has three.** A terminal session under any profile; the dashboard's Chat
tab, which is that same session in a browser and can be scoped to `ralph-orchestrator` or resumed
from a past session; and the card comment thread, which is the only one of the three that is
durable and reaches a future run. §1 records all three with citations. The channel the system is
genuinely missing is not a chat window — it is a way to reach a worker *while it is running*, and
nothing in either tool provides that.

**Is hermes-webui a reasonable one: yes, and it is probably the better-shaped chat.** It presents
conversation as a chat application rather than a terminal emulator, with a session list, model and
reasoning surfaces, attachment storage, a transcription capability probe, and a profile switcher
[SANDBOX][RESEARCH], plus the mobile layout issue #2 credits it with. Against the dashboard's
xterm-in-a-tab, that is a real ergonomic difference and a larger one than anything the monitor
comparison found.

**Which is why the recommendation still does not change.** The ergonomic gain is concentrated
exactly where you have ruled out going: a chat application beats a terminal emulator mainly when
you are not at a terminal — on a phone, on a tablet, away from the desk. You have said off-host is
a non-starter and only the local machine runs this. At the desk, the best chat surface on this
machine is the terminal itself, and the dashboard's Chat tab already covers "in a browser, as a
chosen profile" for the cases where a browser is what you have open. What hermes-webui adds over
that, under your own constraint, is a nicer composer for a conversation you can already have —
against a standing cost that has not changed: because its chat runs Hermes in-process against
internal module layout rather than a versioned API, a pinned hermes-webui is pinned to a
*particular* Hermes, so every Hermes bump becomes a two-sided upgrade [RESEARCH][SANDBOX]. That
trade was not worth making for a monitor and it is not worth making for a composer.

### The option this frame opens, and why it is declined

The honest strongest case for adoption, which neither v1 nor v2 considered: **keep the dashboard as
the monitor and add hermes-webui purely as the chat surface.** The two are not mutually exclusive —
hermes-webui ships a compose layout that runs both against the same Hermes home [RESEARCH] — so
this is a real option, not a rhetorical one, and it drops the disruption of replacing a working
monitor.

It is declined for now on two grounds, both of which you can overturn cheaply. First, the surface
it would add overlaps one you already have and have not tried (decision 2). Second, **no one has
run a single conversation turn on either tool**: the sandbox deliberately gave its scratch profile
no model credentials, so hermes-webui's chat was characterised from its running config and its
endpoints, never used [SANDBOX]. Adopting a second always-on service for a chat experience nobody
has had yet is a decision made on a brochure. Try the tab you own, and if conversation is the thing
that matters, run a chat trial (see "How to overturn this cheaply") — that is the evidence this
record would need, and it does not exist today.

### What this decision is weak against

Stated plainly, because it is the strongest argument on the other side. A terminal UI rendered into
a browser tab is not a chat interface designed for humans, and your comment says exactly that about
how hermes-webui was built. If, having opened `/chat`, you find the TUI-in-xterm unpleasant enough
that you avoid talking to the system at all, then "you already have a chat surface" is technically
true and practically worthless, and the right answer flips to the option above. That judgement
needs five minutes with the tab, which I cannot do for you.

---

## 1. The conversational surfaces this system already has (verified)

Recorded because this revision turns on it. Read from the source of the Hermes install this worker
runs on — version `0.22.2`, commit `d923a5eb487a17c0a68801bba9ef0f56c81040a8` — not from
documentation. [CODE]

**The dashboard has a Chat tab, and it is not optional.** `/opt/hermes/hermes_cli/web_server.py`
lines 461-464: *"In-browser Chat tab (`/chat`, `/api/pty`, `/api/ws`): always enabled"*, implemented
as a module constant rather than a feature flag. The single-page app ships the matching pages:
`web_dist/assets/ChatPage-RJV46vkP.js` and `SessionsPage-Dp4jBTPn.js`. [CODE]

**What that tab runs.** `/opt/hermes/hermes_cli/web_server_chat.py`, `_resolve_chat_argv`
(lines 382-470): the tab spawns `hermes --tui` behind a pseudo-terminal and bridges it to xterm.js
in the page. It takes a `profile` parameter that scopes the whole chat by pointing `HERMES_HOME` at
that profile's directory — the source says it is *"the same propagation `hermes -p <name>`
performs"* — and a `resume` parameter that reattaches to a past session, resolved to its newest
descendant. So "chat with the orchestrator, from a browser, as the orchestrator" is a URL on a port
you already publish. [CODE]

**What it is not.** It is a terminal emulator, not a message composer: `ChatPage` is built on xterm
against `/api/pty` and `/api/ws`. [CODE] Everything true of `hermes chat` in a terminal is true of
it, including the parts that are unpleasant on a small screen.

**The board is the other conversational channel, and the only durable one.** A comment on a card is
delivered into the next worker's context: the `worker_context` this very card handed me opens with
the card body and includes the full comment thread and each parent's handoff. [BOARD] That is how
your six review comments on PR #1 reached v2 and how your reframing comment reached v3. A chat
window has immediacy; the comment thread has memory, an audit trail, and a guaranteed reader.

**Nothing else is connected.** The gateway is running and serves ten profiles, `ralph-orchestrator`
among them, but its only connected platform is the local API server on `127.0.0.1:8642`,
and the channel directory lists no messaging platform at all
(`/home/hermes/.hermes/channel_directory.json`: `"platforms": {}`). [CODE] So the card's premise is
right about messaging — there is no chat app wired to this stack today — and wrong about the
browser: the dashboard's own Chat tab is there.

## 2. What hermes-webui's chat is, and what it can reach

From the sandbox run and the source review, not from its README.

**It is a chat application, and it runs the agent in-process.** Chat is backed by
`run_agent.AIAgent` imported directly, with `run_conversation(..., task_id=...)` — so a conversation
can be bound to a kanban task, which is the one chat affordance in either tool that points at the
board. [RESEARCH] Around it the sandbox observed a session surface (`/api/sessions*`,
`/api/sessions/gateway/stream`), `/api/models`, `/api/reasoning`, `/api/transcribe/capability`, its
own attachment and session storage, and a profile switcher — roughly 100 distinct `/api/*` calls in
one ordinary session. [SANDBOX][RESEARCH] Issue #2 credits it with a mobile layout; that was not
exercised. [ISSUE]

**Its reach is a full agent session.** Chat runs with `toolsets: ["hermes-cli"]`, read from the
running config rather than the README — the same bundle a terminal `hermes chat` gets: terminal,
code execution, file read/write, cron, delegation. [SANDBOX] The workspace browser panel is scoped
to a configured directory; the chat agent's file and code-execution tools are not scoped to it at
all. [SANDBOX] A profile switcher means whoever is authenticated acts as the selected profile.
[RESEARCH] None of this counts against it: the dashboard's Chat tab spawns the same kind of session
with the same toolset under a profile you choose [CODE], and the mitigation for both lives in base
Hermes — sandboxes and approved tools — as you said in review.

**It can act on the board, confirmed by action.** `POST /api/kanban/tasks/<id>/unblock` actually
flipped a task from `blocked` to `ready` in the underlying SQLite board and recorded a canonical
`unblocked` event, cross-checked from a separate shell. [SANDBOX] Its kanban integration is more
mature than its documentation suggests, and that is worth saying even in a record that declines it.

**One tool-specific hazard, carried from v2 because it survives the reframing.** hermes-webui
resolves which board to show from `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD` environment pins, not
from `HERMES_HOME`, and its documentation mentions neither variable. Launched from a shell that had
those pins set, it came up **labelled "Board: Default" while actually displaying the live board**;
no writes occurred and it was corrected immediately. [SANDBOX] Mitigable by setting the variables
explicitly on launch, and not purely its problem — see Observations.

**What nobody did:** run a conversation turn, on either tool. The sandbox profile had no model
credentials, deliberately. [SANDBOX] See Assumption 2.

## 3. The comparison that matters under this frame: having a conversation with the orchestrator

| | hermes-webui chat | Dashboard Chat tab | Card comment |
|---|---|---|---|
| Form | Chat application: composer, session list, reasoning and model surfaces, attachments [SANDBOX] | `hermes --tui` in xterm.js, bridged over `/api/pty` [CODE] | Markdown in a thread |
| Talk as the orchestrator | Profile switcher [RESEARCH] | `profile=` scopes `HERMES_HOME`, as `hermes -p` does [CODE] | n/a — you address the card, the next run reads it |
| Continue a past session | Session list [SANDBOX] | `resume=`, resolved to newest descendant [CODE] | The thread is the memory |
| Reach a worker **mid-run** | No — spawns its own session [RESEARCH][SANDBOX] | No — a scoped chat spawns its own [CODE] | No — delivered at the next run |
| Reaches a future run | Only if written to the board | Only if written to the board | Yes, in `worker_context` [BOARD] |
| Away from this machine | Mobile layout, but off-host is a non-starter [ISSUE] | Same constraint | Same constraint |
| Agent capability | Full `hermes-cli` toolset [SANDBOX] | Full session, same toolset [CODE] | Whatever the next worker has |
| Standing cost | Matched-pair Hermes pinning, one more lifecycle [RESEARCH] | None — ships in the image you pin | None |

**Verdict: hermes-webui wins on form and loses on everything the form would buy.** Its chat is the
better-shaped chat, and the shape pays off mainly off the desk, which is ruled out. On the two
properties this system's loop actually runs on — reaching a worker mid-run, and being read by the
next run — all three columns say the same thing, and the third column is the only one that says
yes to the second. The board, not the chat window, is this system's conversational channel, and
that is a design fact rather than a missing feature.

**What this comparison does not cover**, and it is a sharper limit than v2's: neither chat was ever
used. The form column is read from endpoints, bundles, and configuration, not from a conversation.

## 4. The monitor comparison, demoted

v2's comparison stands, it is simply no longer the deciding one. On the job issue #2 named — find a
blocked card, read its reason, clear it — the two are a **wash**: reading and clearing are
indistinguishable from the operator's seat, both land the same canonical `unblocked` event, and the
dashboard's "⚠ 1 task needs attention" banner is a small genuine convenience on *finding*, worth
one sentence and no more. [SANDBOX] Screenshots of both card-detail and post-unblock states are
attached to card `t_18025034`. The dashboard stays the monitor by default, not by merit: nothing
displaced it. Your standing complaint from issue #2 — that its Kanban view "has been least useful
during this work" — remains untested across a working week, and is now the second-best reason to
run the trial below rather than the first.

## 5. What this decision no longer rests on

v1 gave five reasons; your review retired four, and v2 withdrew them by name. They stay withdrawn
and are not re-argued here: **blast radius** (not a property of the UI; mitigated in base Hermes by
sandboxes and approved tools), **password versus exposure** (moot while nothing binds past
localhost), **third-party and always-on** (the current UI is already both; the residual is one more
lifecycle, weighted at one line), and **the unpinned image** (withdrawn as a reason, kept as a
requirement on adoption — in your words, if it is not a pinned image, it should be). The **banner**
was demoted to a sentence and stays demoted.

v3 retires one more, this time its own: **"the dashboard is the better monitor" is no longer a
reason to decline hermes-webui**, because under your framing it was never offering to be the
monitor. What survives from v2 is exactly one cost — in-process coupling means pinning in matched
pairs, and that constraint does not go away after you fix the pin [RESEARCH][SANDBOX] — and it is
now weighed against a chat surface that overlaps one you already own rather than against a board
view.

## 6. Non-goals

- **Replacing the notifier.** Chat stays the alert path, out of scope per issue #2 — and per your
  review, the notifier plus a poller is already how you communicate with this stack.
- **Any exposure beyond localhost.** See §7.
- **Building a mid-run steering channel.** Decision 3 may turn out to be what you want; it is not
  something either tool offers and not something this record designs.
- **Re-litigating the base system's agent-capability model.** Sandboxes and approved tools are the
  mitigation layer and are not this record's business.
- **Judging hermes-webui as a project.** It looks competent, its chat is the nicer chat, and its
  kanban integration beats its documentation. This record is about fit here.
- **Implementing anything.** No code, compose, or port changes accompany this record.

## 7. Exposure

Nothing in this evaluation was exposed beyond 127.0.0.1: both tools stayed loopback-only for the
entire sandbox run, verified via `/proc/net/tcp` and their startup banners, with no password on
either. [SANDBOX] This record does not authorize exposing anything beyond localhost, which issue #2
places out of scope; any future change there is its own decision, for either tool.

## How to overturn this cheaply

In order, cheapest first. The first step costs nothing and may end the question.

1. **Open `http://127.0.0.1:9219/chat` and talk to the orchestrator for five minutes.** If the tab
   is absent, decision 2 is answered against this record and the chat-only option in §2 gets much
   stronger immediately. If it is present and tolerable, this is settled. If it is present and you
   hate using it, go to step 2 — that is the real finding and it is worth recording on this card.
2. **Run a chat trial, not a board trial.** Install hermes-webui on localhost alongside the
   dashboard, pin its Hermes revision so the matched-pair cost is observed rather than argued,
   launch it with `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD` set explicitly so the board-labelling
   hazard cannot bite, and use its chat — not its board — for a working week. The question it
   answers is whether a real chat UI changes how much you actually talk to this system. If it does,
   overturn to the chat-only option in §2; the monitor need not move either way.

Otherwise, re-open only if one of these changes — "someone found the project again" is not a reason:

1. It talks to Hermes over a versioned, documented API instead of in-process imports — its own
   RFC's "long-term API contract" — removing the matched-pair pinning constraint.
2. The dashboard's Chat tab or Kanban view regresses or is removed upstream.
3. The constraint that only the local machine runs this changes. That reopens the interface question
   legitimately, and the mobile layout becomes a real argument rather than an unreachable one.

## Assumptions

Every inference the brief and the sources did not state, ordered by how much this decision leans on
each.

1. **Load-bearing: your dashboard is the build I read.** The Chat tab, profile scoping, and session
   resume are read from `/opt/hermes` on the host this worker runs on (`0.22.2`,
   `d923a5eb487a17c0a68801bba9ef0f56c81040a8`). The dashboard you publish on `127.0.0.1:9219` comes
   from the image this stack pins, which I could not reach from inside the worker container
   (connection refused) and therefore could not confirm is the same build. Decision 2 is this
   assumption, handed to you as a ten-second check.
2. **Load-bearing: no one has had a conversation with either tool.** hermes-webui's chat was
   characterised from its running config and observed endpoints; the sandbox profile had no model
   credentials by design, so no turn was ever run. [SANDBOX] The dashboard's Chat tab was read from
   source, not opened. This decision turns on an ergonomic comparison neither source measured, and
   that is the honest weakness of v3 — in v2 this was a low-weight assumption; the reframing
   promoted it.
3. **Load-bearing: the Docker install path was never exercised.** hermes-webui was installed via
   `python3 bootstrap.py` against an existing agent directory; its Docker paths were read from
   compose files, not run [RESEARCH], and this stack deploys via Docker Compose. The matched-pair
   pinning cost — the one surviving cost — is therefore inferred for the path you would actually
   use. Flagged by the agent reviewer against v1 and still open.
4. **Moderate: neither chat surface can reach a running worker.** I infer this from structure
   rather than from a test: the dashboard's profile-scoped chat "spawns its own" gateway rather
   than attaching to the dashboard's [CODE] and `resume` reattaches to a session's newest
   descendant, while hermes-webui constructs its own `AIAgent` in-process [RESEARCH]. I did not try
   to interrupt a live dispatched worker from either. If mid-run steering is what you meant, verify
   this before building on it.
5. **Moderate: "chatting with the orchestrator" means the orchestrator profile.** I read your
   comment as being about talking to the system's main agent, and treat a session scoped to
   `ralph-orchestrator` as satisfying it. If you meant something narrower — a specific long-lived
   orchestrator process — nothing in this system has one, and that is decision 3.
6. **Low weight: the mobile layout is taken on the issue's word.** Issue #2 describes it; the
   sandbox drove desktop Chromium only. [SANDBOX] It does not matter under the local-only
   constraint, and would matter immediately if that constraint changed.
7. **Low weight: the comparison is of the software, not of this deployment.** The sandbox ran its
   own instances on `127.0.0.1:8788` and `127.0.0.1:9129` against a throwaway board, not this
   stack's dashboard on `127.0.0.1:9219`. [SANDBOX]
8. **Low weight: maintenance health is partly unverified.** Commit recency confirmed only to
   2026-08-26; issue-response latency not measured (API rate-limited); no CODEOWNERS found.
   [RESEARCH] Not load-bearing — but if you overturn to "adopt", it becomes so.
9. **Low weight: the CDN egress was not traced further.** Opening hermes-webui's page makes the
   browser fetch seven assets from `cdn.jsdelivr.net` (xterm, prism, katex); the built-in dashboard
   bundles its assets and made zero non-localhost requests. [SANDBOX] Whether those can be vendored
   or disabled was not investigated.
10. **This pull request comes from a fork.** The credential available to me authenticates as
    `zhach1`, which has `push: false` on `Zhachory1/docs`, so the branch lives on a fork and PR #1
    is cross-fork. Nothing about the content depends on it.

### No longer assumptions — stated by you in review

- **hermes-webui is built for humans chatting with the orchestrator, not as a monitor.** Your
  framing, now the frame of this record. v1 and v2 both scored it as a monitor; that was the wrong
  test, and §3 replaces it.
- **Off-desk replies are already handled.** The notifier is in the stack, and with a poller
  watching messages you can communicate with it.
- **Off-host access is a non-starter.** Only the local machine runs this. This is what caps the
  value of a mobile-shaped chat.
- **The risk appetite is yours, on your machine.** v1 applied a conservative posture to a service
  exposing no ports. Withdrawn.
- **Blast radius is mitigated in base Hermes**, by sandboxes and approved tools, and does not
  depend on the UI.
- **The current UI is already a third-party always-on service.**
- **An unpinned image is a thing to pin**, not a reason to decline.

## Observations — out of scope, not acted on

- **There is no messaging platform connected to this stack.** The gateway serves ten profiles
  but its only connected platform is the local API server; the channel directory is empty.
  [CODE] If "talk to the orchestrator from wherever you are" ever becomes a goal within the
  local-only constraint, wiring a platform to the gateway is a smaller change than adopting a web
  UI, and it is a different card.
- **The board-resolution hazard is not hermes-webui's alone.** Any tool resolving the board from
  ambient `HERMES_KANBAN_DB`/`HERMES_KANBAN_BOARD` pins can display the live board while labelled
  otherwise. It may deserve its own hardening card regardless of what is decided here.
- **The sandbox report contains one claim this record had to correct.** Its statement that the
  dashboard bundles no chat is contradicted by the dashboard's source. The report is otherwise
  solid; a report's absences are weaker evidence than its observations, and that is a lesson about
  how this board's research gets consumed, not a complaint about the run.
- **`GH_TOKEN` is not forwarded into dispatcher-spawned workers** (only `GIT_AUTHOR_NAME`/`EMAIL`
  are), so an authoring worker has to recover it from elsewhere to push. Worth fixing if authoring
  workers are expected to open pull requests routinely.

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
- **[BOARD]** — The ralph kanban board: the `worker_context` delivered to card `t_582469df` (this
  revision's card; v1 and v2 came from `t_2f2c6131`), which carries the card body, the comment
  thread, and parent handoffs.
- **[ISSUE]** — Zhachory1/ralph#2, read at its current body, including acceptance criteria and
  "Not in scope".
- **Review of v1 and v2** — six line-anchored review comments and two summary comments by `zhach1`
  on `Zhachory1/docs#1`, 2026-10-20, the second of which is the reframing this revision answers;
  plus the agent review recorded on card `t_b493c10f`.
