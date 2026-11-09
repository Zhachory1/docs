---
id: prd-ralph-librarian
title: A weekly librarian keeps each instance's docs repository organised, so knowledge search returns current documents and not stale or duplicate ones
status: in review
owner: Zhachory1
card: none yet (owner brief, 2026-11-09)
revision: v1
date: 2026-11-09
prs:
  - Zhachory1/docs#19   # this PRD
issue: Zhachory1/ralph#130
supersedes: none
---

# A weekly librarian keeps each instance's docs repository organised, so knowledge search returns current documents and not stale or duplicate ones

> **This repository is public.** No employer, person, space or message content
> appears here. The work instance's docs repository is "the work docs
> repository".

Status: **in review**. Nothing is built.

## Review header

**Decisions needed from you:** none block this PRD. One is still open and does
not change what the librarian does (see "Open owner decision").

**Decisions you made** (owner, 2026-11-09, in the brief and its review):

1. **Merge-duplicates is in v1** (R7 to R9). It is not split into a later phase.
2. **"Finished project" is defined mechanically** (R3), not left to judgement.
3. **A forced re-index follows every librarian pull request that merges** (R12).
   No locking against the hourly refresh: `refresh-knowledge` already holds one.
4. **The librarian adds no privacy check of its own** (R14). Whether the docs
   repository is public or private is the owner's setting; the librarian follows
   it the way every profile does.

**Scope:** M. Roughly three or four build PRs in the Ralph repository; the plan
sizes them.

## Who this is for

- **The owner**, who reviews what changes in approved documents and wants
  search to return the current answer.
- **Every searching profile**, which reads the docs repository through the
  knowledge index and cannot tell a replaced decision from its replacement.

## Current state, verified

Cited from Ralph `main` at `e5ce164` and docs `main` at `7ad9a6b` (2026-11-09).

- **The knowledge system is live** (Ralph#121, PRs #122 to #127). Agents file
  notes under `memory/<decisions|root-causes|conventions>/` and
  `memory/reports/YYYY/MM/` (`profiles/PRINCIPLES.md`, "Write down what lasts").
  Nothing tidies the repository afterwards.
- **Nobody owns this job.** README build-order step 4 names a "docs gardener";
  it was never built. The `auditor` reads the docs repository but writes only to
  `areas/agent-system/`, and its SOUL says "Propose; never apply"
  (`profiles/auditor/SOUL.md`).
- **The knowledge PRD excluded it on purpose.** Non-goal N5 of
  `prd-ralph-knowledge`: "No automatic deletion or summarisation of old
  memories in this version." This PRD lifts N5 (R15).
- **Documents link by id, not by path**, so archiving does not break them
  (`AGENTS.md` of the docs repository). Memory notes and pull-request text do
  cite paths.
- **Finished work is already visible in the repository.** `projects/` holds five
  folders. One, `ralph-knowledge`, has an approved PRD, design and plan, and its
  `issue:` (Ralph#121) is closed. Four others carry `status: in review` although
  their pull requests have merged, so their front matter is behind.
- **Refresh is safe to trigger at any time.** `bootstrap/refresh-knowledge` takes
  a `flock`, fast-forwards the checkout, then runs `qmd update` and `qmd embed`;
  a second caller waits and finds little to do. The review loop runs it at most
  hourly from a stamp file (`bootstrap/run-review-loop:111-120`).
- **Direct memory writes can fail on the personal instance.** `Zhachory1/docs`
  has a `main` ruleset requiring pull requests (id 26659401, no bypass actors);
  docs#16 is a memory note that went in as a PR for that reason.

## The outcome

A search for "what did we decide about X" returns the current decision, not its
predecessor, and the repository's shape matches the work's state: finished
projects sit in `archives/`, replaced notes say what replaced them, and indexes
list what is there. The owner reviews one pull request a week, and merging it is
the approval.

## Requirements

**Profile and permissions**

- **R1.** A new profile, `librarian`, exists because it needs a permission no
  other profile has: it edits documents and notes that other profiles wrote,
  across the whole docs repository, and nothing else. It does not judge the
  system (auditor) and does not write new documents (author).
- **R2.** The librarian changes the docs repository **only through one pull
  request per run**. It never pushes to the default branch, so it works whether
  or not the repository requires pull requests. It never merges.

**Archive finished projects**

- **R3.** A folder under `projects/` is finished when **both** hold:
  every document in it has `status: approved` or `superseded`, **and** the
  GitHub issue named in its documents' `issue:` front matter is closed. The
  librarian moves a finished folder to `archives/<same name>/` with `git mv`.
  A project whose documents name no issue is never archived automatically.
- **R4.** Moving a project does not break links. Document links are by id
  (nothing to fix); path links in memory notes are rewritten in the same pull
  request (R10).

**Fix front matter**

- **R5.** Every memory note carries `kind`, `source`, `profile` and `date`
  (`dd-ralph-knowledge` D6). The librarian adds what can be read from the file
  name, the folder or git history, and lists in the pull request body any note
  it could not complete.
- **R6.** A document whose own pull request (`prs:`) has merged, but whose
  `status` still says `in review`, gets `status: approved`. Merging is how the
  owner approves (`AGENTS.md`), so the merge is the evidence.

**Merge duplicates and mark superseded notes**

- **R7.** The librarian finds duplicate memory notes (same finding, different
  files) and writes one merged note that keeps every source's finding, and lists
  the source paths in a `merges:` front matter field.
- **R8.** Each merged source stays in git and in the tree, marked
  `status: superseded` with `superseded_by: <path of the replacement>`, and a
  first line in the body that says so. Nothing is deleted (R13).
- **R9.** A note whose finding a newer note contradicts or replaces (a decision
  overturned, a convention changed) is marked the same way, linked to the newer
  note. The librarian marks only what it can name a replacement for; it does not
  mark a note stale on age.

**Links and indexes**

- **R10.** Broken links between notes and documents are repaired where the
  target exists at a new path, and listed in the pull request body where it
  does not.
- **R11.** Each folder under `projects/`, `areas/`, `archives/` and `memory/`
  has an index that lists its documents by title. The librarian keeps them
  current.

**Reindex and run rules**

- **R12.** When a librarian pull request merges, the knowledge index refreshes
  at once, through the existing `refresh-knowledge`, instead of waiting for the
  hourly stamp. Search results for the moved and superseded paths are checked
  after the refresh (acceptance 1 and 2).
- **R13.** The librarian never deletes a file, and never rewrites the body of an
  approved document other than its front matter and links. History stays in git.

**Instance rules**

- **R14.** The librarian adds no privacy check of its own. It follows the
  instance-wide rule that every profile already has: with `RALPH_DOCS_PUBLIC=1`
  the repository is public and notes follow `PRINCIPLES.md`; otherwise they do
  not. Public or private is the owner's choice of repository, not the
  librarian's decision.
- **R15.** This PRD lifts N5 of `prd-ralph-knowledge` for summarisation and
  replacement of memories. Deletion stays out: R13 keeps it so.
- **R16.** A run is a weekly scheduled job on the librarian profile, per
  instance, reading only that instance's docs repository (same rule as
  `prd-ralph-knowledge` R6). A run that changes nothing opens no pull request.

**Installation**

- **R17.** The profile follows the repository's profile rules: `SOUL.md`,
  `config.yaml`, a row in the README profile table, a decision on toolsets and
  board access, and inclusion in `knowledge_profile` in `bootstrap/install` so it
  can search.

## Acceptance examples

1. On the personal instance, a first run archives `projects/ralph-knowledge/`
   to `archives/ralph-knowledge/` in one pull request. After the owner merges and
   the forced refresh runs, a search for "QMD service per instance" returns the
   design at its `archives/` path.
2. Two memory notes record the same root cause. After a run and a merge, one
   merged note answers the search, and each source shows `superseded_by` and a
   first-line notice when read.
3. A note with a missing `kind` or `date` is completed in the run's pull
   request, or named in its body when it cannot be.
4. The four projects marked `in review` whose pull requests merged get
   `status: approved` in the run's pull request, and none of them is archived
   (no closed issue named).
5. A run with nothing to change opens no pull request.
6. The librarian's token cannot push to the default branch, and a push attempt
   fails.
7. The auditor still cannot edit the docs repository outside
   `areas/agent-system/`.

## Non-goals

- **N1.** No deletion, ever (R13).
- **N2.** No change to the PRD, design and plan gates, or to who approves them.
- **N3.** No new database, service or second store. The docs repository and the
  existing QMD index are the whole system.
- **N4.** No judging of a document's quality or the system's health. That stays
  with the reviewer and the auditor.
- **N5.** Not a replacement for the direct memory write path, and no decision on
  the ruleset (see below).
- **N6.** No expiry by age. A note is marked only when something replaces it.

## Open owner decision

`Zhachory1/docs` requires pull requests on `main` (ruleset 26659401). GitHub
cannot grant a bypass to one person on a user-owned repository; the options are
a role-wide bypass, memory writes as pull requests (reverses decision 1 of
`prd-ralph-knowledge`), or an organisation. **The librarian does not depend on
the answer**: it reads a checkout and writes only through a pull request (R2).

## Unknowns to settle at design time

- **How search treats superseded notes.** A banner on the note, or a move to a
  folder the index leaves out. R8 keeps the file in the tree either way.
- **Folder indexes are indexed too.** They could add noise to search. The design
  decides whether the index excludes them.
- **How the librarian proves "duplicate".** Exact finding match, QMD similarity,
  or a model judgement over candidate pairs, and the confidence it needs before
  it merges.
- **How the forced refresh is triggered** (R12): clear the review loop's stamp,
  or run `refresh-knowledge` on the merge event the notifier already sees.
- **What the weekly job is**: Hermes cron on the librarian profile, or a card.

## Assumptions

- A librarian pull request is opened under the owner's account like other agent
  pull requests, so the owner can merge it.
- A project with no `issue:` is outside "finished" on purpose; a missing link is
  a gap to report, not to guess.
- A run skips when its previous pull request is still open, so two runs never
  edit the same notes. The owner can cut this.
- Setting a stale `status` to `approved` (R6) is safe because the merge is the
  approval; only the document's own merged pull request counts.
- The docs repositories stay small enough (hundreds to low thousands of files)
  for one run to read in full.

## Observations, out of scope

- Memory writes on the personal instance already reach the repository as pull
  requests (docs#16), so the quiet fallback exists today.
- The folders `areas/` and `resources/` are empty (`.gitkeep` only).
