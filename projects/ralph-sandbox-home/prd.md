---
id: prd-ralph-sandbox-home
title: Ralph's large per-profile caches stop sharing a disk with the board, so two concurrent builders cannot break kanban and the gateway by filling it
status: draft
owner: Zhachory1
card: t_08c142ea
revision: v1
date: 2026-11-11
prs:
  - Zhachory1/docs#TBD   # this PRD
issue: Zhachory1/ralph#160
supersedes: none
---

# Ralph's large per-profile caches stop sharing a disk with the board, so two concurrent builders cannot break kanban and the gateway by filling it

> **This repository is public.** No employer, person, space, customer or message
> content appears here. "The work instance" and "the personal instance" are the
> two Ralph deployments; the repositories named are the public Ralph source tree
> and this documents repository.

Status: **draft**. Nothing is built. The design follows as its own pull request
once the owner merges this one.

## Review header

**Decisions needed from you:** four. They set the bounds the design works inside,
and three of them it cannot settle on its own.

1. **Is "the owner's own disk" the whole answer, or does Ralph still bound these
   caches wherever they land?** Measured from an author sandbox on 2026-11-11,
   the host disk reached by the sandbox bind mount is 855G used of 927G, 72G
   free (93%). Moving 7-10G of caches onto it relieves the VM disk, and does not
   by itself make the caches bounded. See R3.
2. **How much blast radius around the board database is acceptable?** One
   candidate direction moves the whole `hermes-home` volume, which also holds
   the Kanban database and every profile's `config.yaml`, onto a host bind
   mount. Whether that is on the table at all is yours, not the design's. See
   R5 and "Candidate directions".
3. **May a cache be evicted while that profile has a card running?** Today's
   policy never touches a busy profile
   (`bootstrap/cleanup-sandboxes:45,152`), and "caches grow while cards run" is
   precisely the failure mode (#160). Fixing it means either accepting that a
   running build can lose its cache mid-run, or accepting that nothing is
   bounded during a run. See R4.
4. **Is your stated preference for directions 1 or 2 over 3 a constraint or a
   lean?** The card records it as a constraint the design must weigh against
   real tradeoffs. If direction 3 turns out to be both sufficient and far
   cheaper, may the design propose it?

**Council dissent:** none convened. This profile's toolset has no
`delegate_task`, and inventing personas to disagree with would be theatre. The
critic pass is the agent reviewer, which runs on this document before you see
it.

**Scope:** S to M. Roughly one to three build PRs in the Ralph repository, plus
a docs update; the design and then the plan size them. The spread is wide
because decision 2 changes the shape: a cleanup-policy change is one small PR,
a volume-topology change is a migration with a runbook.

**What changed since the last revision:** first revision.

## Who this is for

- **The owner**, who runs both instances on one machine, lost Ralph twice in one
  session to a full disk, and had to recover a previous instance of this failure
  by hand inside the VM (#97). The outcome wanted: these caches live on the
  owner's own disk, where the owner's own disk space and whatever cleanup the
  owner already runs apply to them — not more VM-side pruning.

Not for: anyone else. There is one operator. This PRD adds no user, actor or
service, and names no profile that does not already exist.

## Current state, verified

Cited from Ralph `main` at `961f453` and this repository's `main` at `cac10e6`
(2026-11-11). Live figures measured from a `ralph-author` sandbox on
2026-11-11; figures from inside `hermes-home` are the owner's, carried with
provenance (see Assumptions 1 and 2).

- **The disk pressure is real and still present.** From inside a sandbox,
  `df -h` reports the filesystem backing the sandbox container's root (the VM
  disk, `/dev/vdb1`) at **86G used of 99G, 8.8G free (91%)**. The host
  filesystem reached through the bind mount reports **855G used of 927G, 72G
  free (93%)**.
- **Two instances share that one VM and one host machine.** Each is its own
  compose project with its own volumes (`docs/infrastructure.md:19`), and their
  concurrency settings add up — the compose file says so where it sets
  `RALPH_MAX_IN_PROGRESS`, default 6 per instance
  (`compose/docker-compose.yml:79-80`).
- **The sandbox's own home is already on host disk, and is small.** `#97` moved
  it: `bootstrap/install:98-99` sets `terminal.sandbox_dir` to
  `$RALPH_SANDBOX_ROOT/<profile>` for the global config and
  `bootstrap/install:209` does the same in each profile's `terminal` block, and
  compose binds that absolute host path into the containers at the same path
  (`compose/docker-compose.yml:173,470`). Verified from inside a live sandbox:
  both `/root` and `/workspace` are `virtiofs` mounts of the host share, not the
  container overlay. The owner measured a few hundred KB to tens of MB per
  profile. **This is not where the space went**, and
  `bootstrap/cleanup-sandboxes` already scans exactly this path.
- **The growth is in a different home: the Hermes agent process's own.** It sits
  at `/home/hermes/.hermes/profiles/<profile>/home/`, inside the `hermes-home`
  named volume on the VM disk. The owner measured 7.2G for `ralph-maintainer`
  alone on the work instance (#160, correcting comment, 2026-11-11):

  | Path inside the volume | Size |
  |---|---|
  | `profiles/ralph-maintainer/home/.cache` | 3.1G |
  | `profiles/ralph-maintainer/home/go/pkg` | 2.9G |
  | `profiles/ralph-maintainer/home/go-arm` | 709M |
  | `profiles/ralph-maintainer/home/.npm` | 518M |

  The original report named the sandbox home instead and measured the volumes
  whole: `ralph-work_hermes-home` 9.8G, `ralph-personal_hermes-home` 1.6G
  (#160, original body). Its comment corrects the directory, not the volume
  sizes.
- **That volume is shared by everything Hermes owns.** `hermes-home` is declared
  as a named volume, deliberately not a bind mount of the host's `~/.hermes`
  (`compose/docker-compose.yml:481-483`, `compose/README.md:98`), and it is
  mounted into `init`, `hermes` and `hermes-webui`
  (`compose/docker-compose.yml:90,167,466,468`). It holds the Kanban database,
  every profile's `config.yaml` and SOUL, and every profile's agent-side home,
  together. `docs/infrastructure.md:70` describes the `hermes` container's reach
  as "the board database, every profile's home".
- **Nothing bounds the agent-side home.** `bootstrap/cleanup-sandboxes` is
  invoked once an hour by the review loop, against the sandbox root only
  (`bootstrap/run-review-loop:104-109`). Its cache rule fires when a profile's
  **sandbox** home exceeds `HOME_LIMIT_BYTES` (5G) **and** that profile has no
  card in `todo`, `ready`, `running` or `blocked`
  (`bootstrap/cleanup-sandboxes:43,45,152-153`). Its evictable set is
  `CACHE_DIRS = (".cache", ".npm", "go/pkg/mod")` (`:44`). So: the same
  *category* of directory it already deletes, in a tree it never opens, under a
  gate that by construction never fires while the growth is happening.
- **The evictable set does not match what grew.** Of the four measured
  directories, `.cache` and `.npm` match `CACHE_DIRS`; `go/pkg` is a superset of
  `go/pkg/mod`, and `go-arm` matches nothing.
- **The trust boundary is documented and holds.** `docs/spikes/sandbox-isolation.md`
  records that Hermes constructs the sandbox's mounts, the agent does not choose
  them, and that from inside a live sandbox the Docker socket, the board
  database and `/<home>/.hermes/profiles` are all absent. Re-checked on
  2026-11-11 from this author sandbox: `/home/hermes/.hermes/profiles` does not
  exist there. `evals/test_sandbox_isolation.py` checks this against a live
  sandbox.

## The outcome

The caches that Ralph's heavy profiles fill are on the owner's own disk, under
the owner's own disk-space and cleanup policy, and they are bounded there. The
disk that holds the board and the gateway's state is no longer the place that
runs out when two cards build large repositories at once. Recovering from a
cache that grew too far does not require entering the VM.

**Success metric:** with the two heaviest profiles running concurrently for a
full card each on large repositories, the VM disk does not reach the point where
kanban reads fail (`sqlite3.OperationalError: disk I/O error`) or gateway writes
fail (`OSError: [Errno 28] No space left on device`), and neither failure
recurs across a week of ordinary operation on both instances.

## Requirements

- **R1.** The directories that grow without bound during a card — package
  manager and build caches under each profile's agent-side home — do not share a
  filesystem with the Kanban database or the gateway's writable state. Filling
  them must not be able to make the board unreadable.
- **R2.** Two cards on the heaviest profiles, running concurrently, can each
  clone and build a large repository for the whole of their run without
  exhausting the disk they write to.
- **R3.** Wherever those caches live, there is a stated budget and a mechanism
  that keeps them inside it. "Which disk" and "what cap" are two questions, and
  moving the caches answers only the first: the host disk measured 93% full on
  2026-11-11, so an unbounded cache on it relocates the failure rather than
  removing it. The budget may be a different number from the VM's.
- **R4.** The eviction policy states plainly what happens to a profile that has
  a card in flight. Either eviction can touch it and the policy says what a
  running build loses, or eviction cannot touch it and the policy says what
  bounds growth during a run instead. Silence here is what produced #160: the
  present gate avoids the question by never firing while the growth happens.
- **R5.** The Kanban database keeps the durability and locking behaviour it has
  today. If a change moves where it is stored, that change carries evidence that
  SQLite behaves the same on the filesystem it lands on.
- **R6.** Recovering from a full cache does not require `colima ssh` or a
  manual delete inside the VM. The usage is visible, and the space is
  reclaimable, from a path the containers can already see. (#97 recovery needed
  exactly that manual step.)
- **R7.** Usage is attributable per profile. "Which profile's caches are large
  right now" is answerable without entering the VM.
- **R8.** The sandbox trust boundary is unchanged, and the result does not
  contradict `docs/spikes/sandbox-isolation.md`. Nothing new becomes visible to
  code running in a sandbox.
- **R9.** The change installs on a running instance. `bootstrap/install` or
  compose applies it on the next `init`, and the migration does not strand or
  corrupt a card that is in flight when it runs.
  `compose/.env.example:21-23` already warns that changing the sandbox root on a
  running stack leaves a card started in between pointing at the old path; the
  same hazard class applies here.
- **R10.** Both instances get the fix from the same configuration, with no
  hand-editing of one of them.
- **R11.** Ralph's own documentation matches the result: the sandbox mount table
  in `docs/infrastructure.md:78-87` and, if any isolation property changes at
  all, `docs/spikes/sandbox-isolation.md`.

## Acceptance examples

1. A builder and a maintainer each clone and build a large repository
   concurrently for a full card. Both finish. Board reads and gateway writes
   succeed throughout, and the VM disk's free space at the end is within a
   stated margin of where it started.
2. A profile's caches exceed the stated budget. The budget is enforced, and the
   enforcement follows whatever R4 settles — including when that profile has a
   card running.
3. An operator asks which profile is using the most cache space, and answers it
   without `colima ssh`.
4. The caches are filled on purpose until the budget trips. Nothing about the
   board's behaviour changes: no corrupted database, no lost card.
5. `evals/test_sandbox_isolation.py` passes against a live sandbox after the
   change, with the same results as before it.
6. A fresh instance is installed from `compose/.env.example` and gets the same
   arrangement, with no step the second instance needed by hand.

## Candidate directions, as context only

**This PRD does not choose.** The directions are here because they are why the
problem is not a one-line mount change, and because the owner has a stated
preference the design has to weigh. From #160's correcting comment:

1. **Move the whole `hermes-home` volume to a host bind mount.** Simple to
   state, and it moves the Kanban database and every profile's config too —
   bigger blast radius, and it needs evidence that Hermes' SQLite usage is safe
   on whatever host filesystem backs it.
2. **Use a per-profile home-directory override, if Hermes has one distinct from
   `terminal.sandbox_dir`.** `terminal.sandbox_dir` controls only the sandbox's
   own home, not the agent process's. Whether such an override exists has to be
   checked against Hermes' own documentation and CLI, not assumed.
3. **Leave the volume where it is, and extend `cleanup-sandboxes` (or a sibling)
   to also bound
   `/home/hermes/.hermes/profiles/*/home/{.cache,.npm,go/pkg/mod,go-arm}`.**
   Narrower, stays on the VM disk, and addresses the growth directly rather than
   its location.

**The owner's stated preference is 1 or 2 over 3** (#160, "Next"; carried into
the card brief as a constraint). The design must weigh it against the real
tradeoffs above and say what it costs, not treat it as settled. Note that
whatever direction wins, R3 and R4 still have to be answered: direction 3 is
about bounding and directions 1 and 2 are about location, and the failure needs
both.

## Non-goals

Carried verbatim from the card brief:

- **N1.** No change to the sandbox security model (`terminal.backend: docker`
  stays; see `docs/spikes/sandbox-isolation.md`, which this must not
  contradict).
- **N2.** No change to which profiles get a sandbox.
- **N3.** No change to the sandbox's own workspace/home mount
  (`terminal.sandbox_dir`), which already works correctly.

And:

- **N4.** No pruning of the repository cache. Clones borrow its objects
  (#79), and #97 already ruled it out.
- **N5.** No cleanup agent profile. The rules are fixed and it would need delete
  rights over every profile's files; #97 settled this, and
  `bootstrap/cleanup-sandboxes`'s own docstring records why.
- **N6.** No change to card concurrency, memory ceilings or model limits.
  Lowering `RALPH_MAX_IN_PROGRESS` would reduce the symptom by doing less work,
  which is not the outcome wanted.
- **N7.** No change to where the Kanban database or any profile's `config.yaml`
  logically lives from Hermes' point of view. If a direction moves the volume
  underneath them, the paths inside the container stay as they are.

## Unknowns to settle at design time

- **Why the agent-side home holds build caches at all.** Builds run in the
  sandbox, whose home is elsewhere and small. A 2.9G `go/pkg` and a 709M
  `go-arm` under the *agent's* home mean something is installing and building
  outside the sandbox. Identifying what writes there could change the shape of
  the fix entirely — if those writes are avoidable or redirectable at their
  source, the directory may not need to move. This is the first thing the design
  should establish.
- **Whether Hermes has a per-profile agent home override** distinct from
  `terminal.sandbox_dir` (direction 2). Check Hermes' own documentation and CLI;
  do not infer it from the existence of `HERMES_HOME`, which compose sets once
  per container (`compose/docker-compose.yml:76,111,217,261`), not per profile.
- **Whether SQLite is safe for the board on the host filesystem** reached
  through the VM's share (direction 1). The containers see it as `virtiofs`.
  Locking semantics on that path are the question, and they decide whether
  direction 1 is viable at all.
- **What the budget is, and over what window** (R3): a per-profile cap, a
  per-instance cap, a floor of free space, or a combination.
- **Which directories belong in the evictable set.** The measured growth
  includes `go/pkg` and `go-arm`, neither of which today's `CACHE_DIRS`
  (`bootstrap/cleanup-sandboxes:44`) matches.
- **Whether the two instances need separate roots on host disk**, as they have
  separate sandbox roots today (#77), or can share one with per-instance
  accounting.
- **How the budget is observed** (R7): a line in the review loop's log, the
  morning digest, or something an operator runs.

## Assumptions

Every inference this PRD makes that the brief did not state.

1. **The 7.2G breakdown is the owner's measurement, not this run's.** An author
   sandbox cannot read `/home/hermes/.hermes/profiles` — verified again on
   2026-11-11, and expected, per `docs/spikes/sandbox-isolation.md`. The figures
   are carried with provenance (#160's correcting comment, 2026-11-11, work
   instance) and were not re-confirmed here. Requirements are written against
   the shape of the problem — one profile's caches reach several GB inside a
   shared volume — not against 7.2G exactly.
2. **The original report's volume sizes (9.8G, 1.6G) still describe real
   volumes.** The correcting comment corrects which *directory* grew, not the
   volume totals, so both are treated as consistent.
3. **"The VM disk" is the filesystem backing Docker's storage in the colima
   VM**, the 99G `/dev/vdb1` the issue names, which the sandbox container's own
   root also sits on. That is what this run measured at 91% full.
4. **The host filesystem measured at 93% full is the owner's own disk** — the
   destination the issue proposes. It is identified only as the bind-mount
   source visible from the sandbox; this run could not confirm what backs it.
5. **The PRD round was added on purpose.** #160's "Next" says the direction was
   already settled and "this starts at design rather than a PRD round". The
   correcting comment then unsettled it. This PRD assumes the orchestrator added
   the round for that reason. If the owner still wants to skip straight to
   design, merging this costs one round and loses nothing.
6. **Nothing has been built for #160 yet**, and no branch or pull request in the
   Ralph repository implements any part of it.
7. **Both instances run the same Ralph revision**, so a fix in `bootstrap` and
   `compose` reaches both on the next `init`.
8. **The failure is caches, not card checkouts.** Checkouts are already
   collected by the existing rules (`bootstrap/cleanup-sandboxes:124-150`), and
   the measurement names only cache directories.

## Observations, out of scope

- **The cleanup script may be scanning a path that does not exist, silently.**
  `bootstrap/run-review-loop:107` passes
  `--root "${RALPH_SANDBOX_DIR:-/workspaces/sandboxes}"`, but the `review-loop`
  service sets neither `RALPH_SANDBOX_DIR` nor `RALPH_SANDBOX_ROOT`, and mounts
  only `${RALPH_WORKSPACES}:/workspaces` (`compose/docker-compose.yml:247-280`).
  So the root is always the fallback, which is correct only while
  `RALPH_SANDBOX_ROOT` happens to be `<RALPH_WORKSPACES>/sandboxes` — the
  documented default (`compose/.env.example:17,23`). Point it elsewhere and
  `cleanup-sandboxes` prints "does not exist; nothing to do" and returns 0
  (`bootstrap/cleanup-sandboxes:167-169`): a silent success that cleans nothing.
  Worth a separate issue.
- **The isolation spike's mount table is pre-#97.**
  `docs/spikes/sandbox-isolation.md:21-23` still lists the sandbox mounts as
  `profiles/<profile>/sandboxes/docker/default/{workspace,home}`, inside
  `hermes-home`. #97 moved them to `$RALPH_SANDBOX_ROOT`. The document's
  conclusions about what a sandbox can reach are unaffected and were re-checked
  here; only the paths in that table are stale.
- **`CACHE_DIRS` names `go/pkg/mod` but the growth is in `go/pkg`**
  (`bootstrap/cleanup-sandboxes:44` against #160's table). Even applied to the
  right tree, today's set would leave most of the Go cache behind.
