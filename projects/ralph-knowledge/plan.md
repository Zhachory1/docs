---
id: plan-ralph-knowledge
title: Ralph knowledge ships in four pull requests, service first, so each one is usable and reversible on its own
status: approved
owner: Zhachory1
card: none (owner request in session, 2026-11-08)
revision: v1
date: 2026-11-08
prs:
  - Zhachory1/docs#11   # this plan
implements: dd-ralph-knowledge
issue: Zhachory1/ralph#121
supersedes: none
---

# Ralph knowledge ships in four pull requests, service first, so each one is usable and reversible on its own

Status: **approved** by the owner (merge of Zhachory1/docs#11, 2026-11-08).
Implements `dd-ralph-knowledge` (docs#10). Build follows one PR per section
below, exactly as written (Ralph#120).

All PRs are in `Zhachory1/ralph`, against `main` at `ff6ef4a` or later.

## Review header

**Decisions needed from you:** none; the design settled them.

**Order:** PR 1 → PR 2 → PR 3, then PR 4. PR 3 and PR 4 can run in parallel once
PR 2 is merged.

## PR 1: the `knowledge` service

- **Goal:** each instance runs QMD over a checkout of its docs repository, with
  nothing installed on the host.
- **Implements:** design D1, D2, D4.
- **Depends on:** nothing.
- **Size:** ~250 changed lines.
- **What:**
  - New `compose/Dockerfile.knowledge`: Node 22, SQLite, `@tobilu/qmd@2.8.3`
    pinned, CPU only.
  - A `knowledge` service in compose: `qmd mcp --http --host 0.0.0.0 --port 8181`,
    `QMD_ALLOWED_HOSTS=knowledge`, `mem_limit: 2g`, `QMD_FORCE_CPU=1`, a named
    `knowledge` volume, and a read-only bind of `../docs` at `/ralph/docs`. Not
    published to the host.
  - New `bootstrap/knowledge-init`, run by `init`: writes `index.yml` (collections
    `docs` and `ralph` with their contexts, embed and re-rank models, no
    expansion), clones `RALPH_DOCS_REPO` into the volume if missing, then
    `qmd pull`, `qmd update` and `qmd embed`.
  - `RALPH_KNOWLEDGE` in `.env.example`; `0` leaves the service out.
- **Planned files:**
  <planned_files>
  - compose/Dockerfile.knowledge
  - compose/docker-compose.yml
  - compose/.env.example
  - bootstrap/install
  - bootstrap/knowledge-init
  - evals/test_knowledge_service.py
  </planned_files>
- **Test plan:**
  - Unit: `knowledge-init` writes the expected `index.yml` and is idempotent.
  - Live, both instances: `qmd status` inside the service shows the two
    collections with documents and vectors; a `POST /query` returns a hit for a
    known PRD title; the work and personal indexes hold only their own repo.

## PR 2: the shared MCP block

- **Goal:** searching profiles reach the service as an MCP server; isolated
  profiles never do.
- **Implements:** design D5.
- **Depends on:** PR 1.
- **Size:** ~150 changed lines.
- **What:**
  - `bootstrap/write-profile-config.py` merges a `mcp_servers.knowledge` entry
    into a profile's existing `mcp_servers` (lens-tools stays untouched).
  - `bootstrap/install` adds it to the default profile and the searching
    profiles by name, only when `RALPH_KNOWLEDGE` is not `0`.
  - Tools limited to `query`, `get`, `multi_get`, `status`.
- **Planned files:**
  <planned_files>
  - bootstrap/write-profile-config.py
  - bootstrap/install
  - evals/test_profile_config.py
  </planned_files>
- **Test plan:**
  - Unit: merge keeps existing servers, adds `knowledge` once, and is idempotent.
  - Profile-config test: the searching profiles and `default` get `knowledge`;
    the lenses, review-synthesis and chat-summarizer do not.
  - Live: a dispatched researcher card calls `knowledge.query` and gets a cited
    hit; a lens card has no `knowledge` tools.

## PR 3: memories and principles

- **Goal:** agents search before they plan or write, and record what lasts.
- **Implements:** design D6, D7.
- **Depends on:** PR 2.
- **Size:** ~120 changed lines.
- **What:**
  - `PRINCIPLES.md`: "search before you plan or write, cite what you used" and
    "write down what lasts", with the note template, the `memory/` layout, and
    the single `gh api -X PUT` write.
  - The public-safety rule, keyed to `RALPH_DOCS_PUBLIC`, which install derives
    from `gh repo view --json visibility` and writes into the profile
    environment.
- **Planned files:**
  <planned_files>
  - profiles/PRINCIPLES.md
  - bootstrap/install
  - evals/test_profile_config.py
  </planned_files>
- **Test plan:**
  - Profile-config test: every SOUL carries both lines; install sets
    `RALPH_DOCS_PUBLIC` from the repository's visibility.
  - Live: a card asked to record a finding writes one note under `memory/` with
    the template header; on the personal instance, a note naming a person is not
    written and the card says why.

## PR 4: refresh, status and docs

- **Goal:** the index stays current hourly, its health is one command, and the
  docs and diagrams describe the system as built.
- **Implements:** design D3, D8.
- **Depends on:** PR 1 (and PR 2 for the docs to be accurate).
- **Size:** ~200 changed lines.
- **What:**
  - New `bootstrap/refresh-knowledge`: fetch and fast-forward the checkout,
    `qmd update`, `qmd embed`, write `status.json`. A failure keeps the previous
    index.
  - `run-review-loop` runs it at most hourly (stamp file, next to cleanup).
  - New `bootstrap/knowledge-status`.
  - `docs/infrastructure.md`, `workflows.md`, `operations.md` and the diagrams:
    the service, refresh, recovery; memory, shared MCP and document RAG move out
    of "planned".
- **Planned files:**
  <planned_files>
  - bootstrap/refresh-knowledge
  - bootstrap/knowledge-status
  - bootstrap/run-review-loop
  - evals/test_refresh_knowledge.py
  - docs/infrastructure.md
  - docs/workflows.md
  - docs/operations.md
  - docs/diagrams/system.dot
  - docs/diagrams/system.svg
  - docs/diagrams/system.png
  - docs/diagrams/tools.dot
  - docs/diagrams/tools.svg
  - docs/diagrams/tools.png
  </planned_files>
- **Test plan:**
  - Unit: refresh fast-forwards, re-indexes, and on a fetch failure leaves the
    index and `status.json` intact with the error recorded.
  - Live: a memory written in PR 3's check is searchable after the next refresh;
    `knowledge-status` shows the last refresh time; with the network off,
    search still answers.

## Acceptance (from the PRD)

| PRD example | Proven in |
|---|---|
| 1. A fresh chat answers from a past design, with a citation | PR 2 live check |
| 2. A memory written by one card is found by a later one | PR 4 live check |
| 3. Work and personal indexes do not mix | PR 1 live check |
| 4. Search works with the network off | PR 4 live check |
| 5. A fresh install has working search | PR 1 live check |
| 6. A public-unsafe memory is not written on personal | PR 3 live check |
| 7. A lens card has no QMD tools | PR 2 profile-config test and live check |

## Assumptions

- The design's assumptions hold, in particular that QMD 2.8.3 runs CPU-only on
  Linux arm64 and that Hermes' HTTP MCP client accepts QMD's endpoint. PR 1 and
  PR 2 verify each live before anything depends on it.
