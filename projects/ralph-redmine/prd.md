---
id: redmine-issue-tracking-prd
status: in review
owner: Zhachory1
kanban_card: t_021bba46
pull_requests: []
---

# PRD: Redmine as the canonical issue tracker

Status: in review, v1. No build authorised.
Card: t_021bba46. Research: [public research notes](research-notes.md).

Since this draft was written, the owner created the private `Work Requests` project and one manual document-migration issue. The REST API remains disabled; no agent issue integration is running.

## Review header

**Decisions needed from you (5).** Full text in "Open decisions" at the end.
1. One starter project or two?
2. May the maintainer change issue status, or only humans?
3. May the maintainer create issues, or only comment on issues you create?
4. Enabling the REST API is instance-wide and is yours to do. Confirm you want it.
5. On an ambiguous match, stop and ask, or create a new issue and link it?

**Council dissent.** None. No critic lens ran on this draft. Card t_1f1c1c9a is the
first independent review, and this header will carry its dissent in v2.

**Scope rating.** Small-to-medium. Three build slices, each one PR, each shippable
alone. See tasks.md. The largest risk is not size, it is that the REST API is off
today, so slice 0 is a human action and nothing downstream can be proven until it lands.

**Changes since last revision.** First revision.

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

You create the projects by hand in the web UI. The design never creates a project, and
the service account is not permitted to (see design.md, "Account and allowlist").

## Personas

**Owner (you).** Creates projects, holds an admin account, reads issues, edits anything,
answers the maintainer's questions, closes issues. The only actor who can approve a build.

**Issue maintainer (`ralph-issue-maintainer`).** One scoped Hermes profile. Matches an
incoming request to an existing issue or creates a new one, appends notes when work
completes, asks the owner when the match is ambiguous. It is the only agent that writes
to Redmine.

**Executing profiles (builder, reviewer, researcher, and the rest).** Do the work on
Hermes cards. They never touch Redmine. They report results in their card completion
metadata, which is the handoff the kernel already carries.

**Readers.** Anyone with a Redmine account who needs the state of a request. Read-only
through the web UI, no agent involved.

## Lifecycle and issue ID

The issue ID is Redmine's own issue number. Redmine assigns it on create and it is
globally unique across the instance. We do not invent a second identifier. Cards,
threads, and commit messages reference `#NNN`.

Statuses, using Redmine defaults where they fit (statuses are admin-defined and free
form; research-notes.md §2):

- **New** — recorded, nothing started.
- **In Progress** — a Hermes card for this issue is running.
- **Needs Owner** — blocked on a human judgment. The maintainer sets this when it must
  ask, and it is the status that should page you.
- **Resolved** — work finished, awaiting your confirmation.
- **Closed** / **Rejected** — terminal. Owner only.

The maintainer never closes an issue. Closing is a judgment about whether the request
was satisfied, and the requester owns that judgment.

## Scope

In scope:
- Reading Redmine and writing issues, notes, and relations through one narrow tool.
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

## Assumptions

Listed in full in the completion summary for card t_021bba46. The load-bearing ones:
the Redmine instance is local and single-tenant, REST can be enabled, and the existing
Hermes container can reach it over the network.
