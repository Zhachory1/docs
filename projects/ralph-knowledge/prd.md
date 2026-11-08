---
id: prd-ralph-knowledge
title: Ralph keeps its context, reports and memories in each instance's docs repository, and every agent searches them locally before it plans or writes
status: in review
owner: Zhachory1
card: none (owner request in session, 2026-11-08)
revision: v2
date: 2026-11-08
prs:
  - Zhachory1/docs#9   # this PRD
issue: Zhachory1/ralph#121
supersedes: none
---

# Ralph keeps its context, reports and memories in each instance's docs repository, and every agent searches them locally before it plans or writes

> **This repository is public.** No employer, person, space, message content or
> work document appears here. "The work instance" and "the personal instance" are
> the two Ralph deployments; repository names are the public Ralph source tree and
> this repository.

Status: **in review**. Merging this pull request approves the PRD and releases
the design (Ralph#115). Nothing is built.

## Review header

**Decisions needed from you:**

1. **Writes to the docs repository without a pull request.** Memories and run
   reports are frequent and low-stakes; a PR for each would bury the PRD,
   design and plan PRs you actually review. Proposed (R4): agents commit those
   two kinds directly to a dedicated folder on the default branch, and
   everything that changes a decision still goes through a PR. Accept, or keep
   PRs for everything?
2. **The personal docs repository is public.** Proposed (R7): on the personal
   instance, agents write memories and reports only when they pass the same
   public-safety rule this repository already has (no people, employers, spaces
   or message content); anything that fails it is not written. Or: keep a
   second, private repository for the personal instance's memories.
3. **Your previous private docs repository.** Its contents (about 1,200
   Markdown files, the source of `~/private-docs`) are already merged into a
   private docs repository. Proposed (R16): the work instance indexes it as a
   read-only source alongside its own docs repository; agents read it but never
   write to it. Accept, or keep it out?

**What changed since v1** (owner, 2026-11-08):

- The private docs are already in a docs repository, so they are a read-only
  source (R16), not a non-goal.
- QMD ships with Ralph and installs with it (R9, R17), instead of relying on
  the copy installed on the owner's Mac.

**Scope:** M. Roughly three to five build PRs in the Ralph repository; the plan
document sizes them.

## Who this is for

- **The owner**, who today re-explains context to agents that Ralph itself
  produced last week, and who reviews PRDs, designs and plans.
- **Every Ralph profile** that plans or writes: orchestrator, author, builder,
  maintainer, researcher, reviewer, analyst, experimenter, auditor, watcher,
  and the WebUI `default` profile.

Not for: the review lenses and review-synthesis, which must reason only from the
staged evidence of one PR (Ralph#79); chat-summarizer, which has no tools by
design; anyone other than the owner reading the stored knowledge.

## Current state, verified

Cited from Ralph `main` at `ff6ef4a` (2026-11-08) and the live instances.

- **Each instance already has a docs repository.** `RALPH_DOCS_REPO` is set on
  both: personal `Zhachory1/docs` (public) and a private one for work. It is a
  GitHub `owner/name`, not a local path
  (`compose/.env.example:67`). It receives PRD, design and implementation-plan
  PRs (`profiles/STANDARDS.md:38`, orchestrator SOUL), and generic maintenance
  skips it (`bootstrap/run-review-loop:91`).
- **Nothing searches it.** No checkout exists inside either instance. Agents read
  it one file at a time through `gh api`. The repository's own rules
  (`AGENTS.md`) already organise it by PARA and forbid secrets, customer data and
  copies.
- **Agent memory is off.** Every profile config sets `memory_enabled: false`.
  What a card learns survives only in its completion summary or a comment, and
  the board is a database on one machine that no later agent searches.
- **No shared MCP servers.** `bootstrap/write-profile-config.py` copies only
  `model`, `platform_toolsets`, `approvals`, `skills` and `terminal`; the one
  MCP server in use (lens-tools) is written by hand into each lens config.
- **A local search engine exists, but only on the owner's Mac.** QMD 2.8.3
  (`@tobilu/qmd`): BM25, vector search and LLM re-ranking with local GGUF models
  through `node-llama-cpp`; collections over folders, human-written collection
  context, `update --pull`, and an MCP server with a shared HTTP mode
  (`qmd mcp --http`). Its models (~1.2 GB) are cached under `~/.cache/qmd`.
  Neither instance has it: nothing in Ralph installs, configures or runs it.
- **The owner's previous docs live in a private repository** (about 1,200
  Markdown files), the origin of the host's `~/private-docs`. It is not
  `RALPH_DOCS_REPO` on either instance.
- **Ralph's own docs** (`docs/infrastructure.md`, `workflows.md`,
  `operations.md`, `docs/decisions/`) live in the Ralph repository, not the
  docs repository.

## The outcome

Before an agent plans or writes, it searches its instance's knowledge and cites
what it used. That knowledge is the docs repository: every PRD, design, plan
and decision, plus the reports and memories Ralph writes as it works. A fresh
WebUI chat asked "what did we decide about the work tracker?" answers from the
design document, with a citation, without being pointed to it.

## Requirements

**Storage**

- **R1.** The instance's docs repository (`RALPH_DOCS_REPO`) is the single home
  for Ralph's durable context: documents, decisions, run reports and memories.
  No second store.
- **R2.** Memories and reports live in a dedicated folder whose layout follows
  the repository's PARA rules, so they never mix with reviewed documents.
- **R3.** What may be stored follows the repository's existing rules: no
  secrets, credentials, tokens, customer data, personal data or raw message
  content. A note carries aggregates and links, not copies.

**Writing**

- **R4.** PRDs, designs, plans and decision records keep their pull-request and
  merge-to-approve flow (Ralph#115, #120). Memories and run reports are written
  without a PR (decision 1), only to the dedicated folder.
- **R5.** Each memory or report states what it is, which card or PR produced it,
  the date, and the profile that wrote it, so a reader can check and expire it.
- **R6.** A profile writes only to its own instance's docs repository.
- **R7.** On an instance whose docs repository is public, an agent writes a
  memory or report only if it passes the public-safety rule (decision 2);
  otherwise it does not write it, and says so on the card.

**Reading and search**

- **R8.** Each instance keeps a local checkout of its docs repository, refreshed
  automatically and at least hourly.
- **R9.** That checkout, plus Ralph's own `docs/`, is indexed with QMD running
  inside the instance, not on the host. Search, embeddings and re-ranking run locally; no document text
  leaves the machine.
- **R10.** Agents reach the index through QMD's MCP tools, read-only, pinned to
  their instance's index.
- **R11.** The profiles listed under "Who this is for" get search; the lenses,
  review-synthesis and chat-summarizer do not.

**Use**

- **R12.** The shared principles tell every searching profile to search the
  knowledge index before planning or writing, and to cite what it used.
- **R13.** A card that learns something durable (a decision, a root cause, a
  convention, a rejected approach) writes it as a memory at completion.

**Operation**

- **R14.** Index freshness, size and last refresh are visible in one command and
  in `docs/operations.md`.
- **R15.** Turning the feature off is one setting per instance, and leaves the
  docs repository untouched.

**Sources and installation**

- **R16.** An instance can index additional read-only source repositories
  (decision 3). Agents search them like the docs repository but never write to
  them. A source is indexed only on the instance whose credentials can read it.
- **R17.** QMD, its models and its index are part of Ralph: a new instance gets
  working search from `docker compose up` and `init`, with no step on the host.
  The index and models live in that instance's volumes and survive restarts and
  upgrades.

## Acceptance examples

1. Fresh WebUI chat, personal instance: "what did we decide about the work
   tracker?" The answer quotes the merged PRD or design and cites its path.
2. A card's completion records a root cause. Within the next refresh, a new card
   on the same instance finds it by searching, without being told where it is.
3. The work instance's index contains no personal documents, and the personal
   instance's none of the work ones.
4. With the network off, search still answers from the index.
4a. A fresh instance stood up from `docs/operations.md` has working search
    after `docker compose up`, without installing anything on the host.
4b. On the work instance, a question answered only by the previous private
    docs is answered with a citation to that repository; an attempt to write
    there is refused.
5. A memory that would name a person or contain message content is not written
   on the personal instance; the card says why.
6. A lens card has no QMD tools.

## Non-goals

- **N1.** No new database, vector store, sidecar service or second document
  store. The docs repository and a QMD index are the whole system.
- **N2.** No cloud embeddings or hosted RAG.
- **N3.** Ralph does not read the host's `~/private-docs` folder directly; its
  contents reach Ralph only through the repository in R16.
- **N4.** No change to the PRD, design and plan approval gates.
- **N5.** No automatic deletion or summarisation of old memories in this
  version.
- **N6.** Hermes' built-in memory stays off.

## Unknowns to settle at design time

- Where QMD runs inside an instance: a small per-instance compose service using
  QMD's HTTP MCP mode, the hermes container, or each sandbox; how its 1.2 GB of
  models are downloaded once and cached in a volume.
- How the MCP server reaches profiles, given that install does not copy
  `mcp_servers` today.
- How agents commit memories (`gh api` contents writes or a checkout), and how
  concurrent writers avoid conflicts.
- The exact folder layout and note template for memories and reports.

## Assumptions

- QMD's local models fit the VM (12 GB) alongside current load.
- The docs repositories stay small enough (hundreds to low thousands of files)
  to re-index hourly.
- The token each profile already has can write to its docs repository.
- "Reports" means run reports and research notes Ralph produces, not the
  Chat digest, which already has its own delivery.

## Observations, out of scope

- The diagram's "Planned / not wired" band lists memory, a shared MCP set and
  document RAG; this PRD covers all three.
