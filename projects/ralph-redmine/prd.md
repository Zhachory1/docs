---
id: redmine-issue-tracking-prd
status: in review
owner: Zhachory1
kanban_card: t_021bba46
pull_requests:
  - https://github.com/Zhachory1/docs/pull/5
---

# PRD: Redmine as the canonical issue tracker

Status: in review, v4. No build authorised.
Card: t_021bba46. Research: [public research notes](research-notes.md).

Since this draft was written, the owner created the private `Work Requests` project and one manual document-migration issue. The REST API remains disabled; no agent issue integration is running.

## Review header

**Decisions needed from you (5).** Full text in "Open decisions" at the end. The list is
shorter than v3's, not longer: you answered D6 and D7 on PR #5 and v4 applies both.
1. One starter project or two?
2. May the maintainer change issue status, or only humans?
3. May the maintainer create issues, or only comment on issues you create?
4. Enabling the REST API is instance-wide and is yours to do. Confirm you want it.
5. On an ambiguous match, stop and ask, or create a new issue and link it?

**Council dissent.** Independent review card t_1f1c1c9a reported no blockers. Your
round-1 review requested changes and your round-2 answers settled all three comments.
One of your answers rejected an argument this design made — see design.md, "Why the
orchestrator may, and the executing cards may not". Where an answer of yours left a
smaller question behind, v4 applies the answer and records the reading it chose under
Assumptions, rather than opening a new decision and handing it back to you.

**Scope rating.** Small-to-medium. Three build slices, each one PR, each shippable
alone. See tasks.md. The largest risk is not size, it is that the REST API is off
today, so slice 0 is a human action and nothing downstream can be proven until it lands.

**Changes since last revision (v3 to v4).** Your three answers are applied.
"Delete the allowlist": the client-side `REDMINE_PROJECT_ALLOWLIST` check is gone, the
maintainer now chooses the project, and Redmine role membership is the only boundary
left (D7). "Me writing comments is the acknowledgement": the requester is acknowledged by
a person writing on the issue, nothing in the design sends anything, and the "No Chat
writing" non-goal stands (D6). Your tool-access answer: the orchestrator holds a narrow
Redmine tool and sets In Progress, which also closes design.md decision 4. Personas,
Lifecycle, Scope and Assumptions moved with those three.

**Earlier revisions.** v2 opened D6 and D7 from your round-1 comments. v3 moved the
assumptions into this document from a Kanban card summary that a reader of this
repository cannot open.

## Problem

A request arrives in a Chat thread. It becomes a Hermes card. A follow-up arrives a
week later in a different thread and becomes a second card. Nothing connects them.
There is no single place that answers "what is the state of this request, who owns it,
what happens next". The cards are an execution queue and they are good at that; they
are not a record, because a card dies when its run ends.

## Outcome

One request has one Redmine issue. The issue holds status, owner, next action, and
links out to the threads, the Hermes cards, and any GitHub issue or PR. A person opens
the issue and knows where things stand without reading a transcript.

The Hermes board stays what it is: the execution queue. Redmine does not schedule work
and does not dispatch agents.

Measured by: for every request that produced a card in the period, exactly one Redmine
issue exists, and its last note is not older than its last completed card.

## Starter projects

**Recommended: one project, "Work Requests".**

Every incoming request, from any thread, lands here. Trackers separate the kinds of
work. Categories separate the subject areas. One project means one membership, one
role, one allowlist entry to review.

**"Ralph Operations" is deferred, not rejected.** A second project earns its keep only
when the two populations need different *readers*. Open it when either is true:
- Ralph's own maintenance work (upgrades, failed jobs, self-repair) is noisy enough
  that it buries human requests in the same issue list.
- Someone needs to read Work Requests who must not read operations issues, or the
  reverse. That is an access separation, and it is the only argument a second project
  wins on its own.

Neither warrants another project yet: `Work Requests` has only its first manual issue,
so there is no volume to separate and no second reader to protect.

You create the projects by hand in the web UI. The maintainer chooses which existing
project an issue belongs in, since you deleted the allowlist (D7), but it still cannot
create one: the design never creates a project and the service account is not permitted
to (see design.md, "Accounts and permissions").

## Personas

**Owner (you).** Creates projects, holds an admin account, reads issues, edits anything,
answers the maintainer's questions, closes issues. The only actor who can approve a build.

**Issue maintainer (`ralph-issue-maintainer`).** One scoped Hermes profile. Matches an
incoming request to an existing issue or creates a new one, appends notes when work
completes, asks the owner when the match is ambiguous. It is the only agent that creates
issues.

**Orchestrator (`ralph-orchestrator`).** Holds the same tool narrowed to three calls:
read an issue, set its status, append a note. New in v4, from your review. A request
reaches it only after the chat maintainer and the issue maintainer have both handled it,
and without it nothing moves an issue off New while the work runs. It cannot create or
relate issues, and its Redmine account cannot either.

**Executing profiles (builder, reviewer, researcher, and the rest).** Do the work on
Hermes cards. They never touch Redmine, and this is the one place v4 kept the old line:
they read repositories, dependencies and web pages, which no profile upstream has
filtered. They report results in their card completion metadata, which is the handoff the
kernel already carries, and the orchestrator or the report card puts it on the issue.

**Readers.** Anyone with a Redmine account who needs the state of a request. Read-only
through the web UI, no agent involved.

## Lifecycle and issue ID

The issue ID is Redmine's own issue number. Redmine assigns it on create and it is
globally unique across the instance. We do not invent a second identifier. Cards,
threads, and commit messages reference `#NNN`.

Statuses, using Redmine defaults where they fit (statuses are admin-defined and free
form; research-notes.md §2):

- **New** — recorded, nothing started.
- **In Progress** — a Hermes card for this issue is running. The orchestrator sets it in
  the same run that creates the execution cards.
- **Needs Owner** — blocked on a human judgment. The maintainer sets this when it must
  ask, and it is the status that should page you.
- **Resolved** — work finished, awaiting your confirmation.
- **Closed** / **Rejected** — terminal. Owner only.

The maintainer never closes an issue. Closing is a judgment about whether the request
was satisfied, and the requester owns that judgment.

## Scope

In scope:
- Reading Redmine and writing issues, notes, and relations through one narrow tool: the
  full surface for the maintainer, read plus status plus note for the orchestrator.
- Matching a new request to an existing issue, including across threads.
- Asking the owner when the match is unclear.
- Linking issues to Hermes cards and to GitHub issues/PRs by reference.

Non-goals:
- **No Chat writing.** Nothing here sends a Chat message. Redmine writes and Chat sends
  are separate problems and stay separate.
- **No agent-created projects, accounts, roles, or credentials.** Those are yours.
- **No Redmine as a scheduler.** It does not dispatch, queue, or assign work to agents.
- **No migration of history.** The instance starts empty and stays that way until the
  first live request.
- **No second service container.** See design.md for why the existing Hermes container
  is enough.
- **No automatic closing of issues.**
- **No poller watching for other profiles' results.** The task graph already does this.
- **No time tracking, no wiki, no forums, no custom workflow engine work beyond the
  statuses above.**

## Private data

An issue carries a **summary and references**, never source material.

What goes in an issue:
- A short restatement of the request in the maintainer's own words, under a few hundred
  characters.
- Stable identifiers: source thread id, Hermes card id, GitHub URL.
- Status, owner, next action.

What never goes in an issue:
- Raw Chat transcript or quoted message bodies.
- Customer names, credentials, tokens, file contents, or anything pasted from a private
  document.

This is enforced in two places, not one: the maintainer is told to summarise, and the
tool layer refuses a create or note over a byte cap and strips a declared marker set.
A rule the model is asked to follow is a preference; a rule the tool enforces is a
boundary. Acceptance test A7 tests the boundary, not the preference.

Anyone reading the issue who needs the source follows the thread id back to Chat, where
the existing access controls already apply. The issue adds no new disclosure surface.

## Open decisions

**D1. One project or two?** Recommendation: one, "Work Requests". Cost of being wrong:
low. Splitting later is a move operation in the web UI, and issue IDs survive a move.

**D2. May the maintainer set status?** Recommendation: yes, but only New to In Progress
to Resolved, and New or In Progress to Needs Owner. Closed and Rejected stay yours. If
you prefer the maintainer never touch status, it still works: the issue then reports
state through notes alone, and you lose the ability to filter by status, which is most
of the point of using a tracker.

**D3. May the maintainer create issues?** Recommendation: yes. If it may only comment,
then every new request waits on you to open an issue first, and the system stops being
an answer to the disconnected-card problem. Creating is bounded by the project allowlist
and the account's role.

**D4. Enabling the REST API.** It is off today. It is an instance-wide setting, not a
per-project one (research-notes.md §1), so turning it on exposes the API surface for
every project and every user key on that instance, not only ours. You enable it; no
agent does. If you do not want it instance-wide, this whole design stops here and we
need a different approach.

**D5. Ambiguous match.** Recommendation: stop and ask. The maintainer sets Needs Owner
and blocks its card. The alternative, creating a new issue and relating it to the
candidate, is cheaper per event and produces exactly the duplicate sprawl this design
exists to prevent. The cost of asking is one interruption; the cost of guessing wrong
is a split history that nobody notices for a month.

**D6. Does the requester get an acknowledgement? Answered: yes, by a person.** You wrote
"Me writing comments is the acknowledgement." So the acknowledgement is a comment a human
writes, and nothing in this design sends anything: the "No Chat writing" non-goal stands,
`watcher_user_ids` is not used, and the two other options v3 costed are dropped.

That leaves the design one job, and it is all v4 adds. The issue has to be findable the
moment it exists, or the comment you would write costs a search first. The intake card's
completion metadata already carries the issue id; it now carries the issue URL as well,
and acceptance A1 asserts it.

One thing your answer does not settle, and it belongs to the instance rather than to this
design: a comment reaches the requester only if the requester can see it. That needs a
Redmine account per requester and working instance mail — neither exists today and this
instance's mail configuration has never been read — or it needs whoever answers the
thread to paste the comment back. Nothing here depends on which; it is worth knowing
before the first live request.

**D7. Should the client-side project allowlist go? Answered: yes, delete it.** You wrote
"Delete the allowlist", twice. v4 deletes it. `REDMINE_PROJECT_ALLOWLIST` is gone from the
tool layer, from the install options, and from the overlay job; `issue_create` takes the
project as an argument, so the maintainer decides which project an issue belongs in; and
Redmine's per-project role membership is the only boundary left. Acceptance A6 is
rewritten to assert that Redmine refuses, not that the client refuses first.

Two consequences worth your eye, because deleting a layer moves weight onto what is left.
Membership is now the whole control, so widening the service account's membership widens
what the agent can write and no code of ours will catch it. And the orchestrator, which
v4 gives a narrow tool, gets its own Redmine account whose role cannot create issues —
with the client-side check gone, a profile that may not create issues should be unable
to, not merely disallowed (design.md, "Accounts and permissions").

What v4 did not do, because you did not ask for it: the maintainer still cannot create a
project. Your round-1 comment ended mid-word at "should be added i", and the reading that
survives "delete the allowlist" is that the maintainer chooses among projects it is a
member of and asks you when none fits. Creating projects stays a non-goal. Say so if you
meant the stronger thing — it is a change to the account's role, not a check to delete.

## Assumptions

Nothing in the brief or the research stated these. Each one is a place the design breaks
if it is wrong. The first twelve date from v1, four more were made answering your round-1
review, and the last four were made applying your round-2 answers.

1. The Redmine instance is local, single-tenant, and reachable over the network from the
   existing Hermes container. The brief says the instance exists; it does not say the
   container can route to it.
2. The REST API can be enabled. It is off today. If instance policy forbids turning it
   on, this design has no path and must be rethought. That is D4.
3. Enabling REST is instance-wide, not per-project. Taken from the Administration >
   Settings > API location in the research note; not proven for the target version.
4. The target Redmine supports classic per-user API keys (documented since 1.1.0).
   Personal access tokens are assumed absent: the research could not date them.
5. `cf_<id>=<value>` filters as an exact match. Only `~` substring matching is
   documented. The tool compares the returned value, but the query shape is unverified.
6. A `multiple=true` custom field can be filtered on at all. Not documented in the pages
   read. A fallback is named in design.md.
7. Redmine enforces no uniqueness on custom field values, so dedupe rests on a file lock
   plus a pre-check, not on the database.
8. The existing Chat-to-card producer follows the inbox-producer shape: host-side
   credential, staged evidence, operation id of thread id plus latest message id. Read
   from the public inbox-producer; the Chat producer itself was never read and may
   differ.
9. `ralph-orchestrator` is willing to take on the card-linking role in design.md. That
   is the only change this design asks of an existing profile.
10. The private overlay job mechanism is the right carrier for `REDMINE_URL` and each
    profile's `REDMINE_API_KEY`. Inferred from how `GH_TOKEN` is handled; no instance
    configuration was inspected. (`REDMINE_PROJECT_ALLOWLIST` travelled this way too
    until v4 deleted it.)
11. A per-user API key attributes writes to that user in the issue journal. Keys are
    documented as per-user and role-scoped; journal attribution is an inference.
12. One project is enough today. The brief states the instance has no projects; the "no
    second reader to protect" half is an inference about your situation.
13. "There should be some acknowledgement here" meant the requester is the audience, not
    the board or the owner. Made in v2 and not contradicted by your answer, which names
    who writes the acknowledgement but not who reads it. See assumption 17.
14. The truncated allowlist comment must not be read broadly without your confirmation.
    Made in v2; "delete the allowlist" settled it, and v4 applies the narrow reading
    only. See assumption 20.
15. `watcher_user_ids` was the cheapest acknowledgement path, and this instance's mail
    configuration was never read. v4 does not use it — the acknowledgement is a human
    comment — but the mail question survives in D6 for whoever has to read that comment.
16. Nothing in the v3 design set **In Progress**, so an issue read New for as long as the
    work ran. Found while answering your tool-access comment; closed in v4, where the
    orchestrator sets it.
17. "Me writing comments is the acknowledgement" means a human comment is the whole
    acknowledgement path and no agent-sent message is wanted anywhere — not that an agent
    should write the comment on your behalf.
18. Your flow description makes the orchestrator a trusted reader of already-filtered
    input, so it may hold a Redmine tool. Executing profiles stay out, because they read
    material that is past the end of that filtering chain. If you meant executing cards
    too, say so: that is a different design and this one argues against it.
19. The orchestrator's tool is narrowed to `issue_get`, `issue_status_set` (In Progress
    only) and `issue_note_add`. You said it needs to update the issue; which calls that
    means is my reading, and widening it is one line in an allowlist plus one permission
    on its role.
20. "Delete the allowlist" means the client-side `REDMINE_PROJECT_ALLOWLIST` check, not
    the Redmine role membership behind it. Deleting the membership as well would leave
    the account able to write to every project on the instance.
