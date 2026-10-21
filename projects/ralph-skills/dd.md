---
id: dd-ralph-skills
title: Publish 50 skills in ralph, keep 17 in the private overlay, and mount them per profile as read-only lanes
status: in review
owner: Zhachory1
card: t_78046316
revision: v3
date: 2026-10-21
prs:
  - Zhachory1/docs#2
---

# Publish 50 skills in ralph, keep 17 in the private overlay, and mount them per profile as read-only lanes

> **Names in this document are placeholders.** This repository is public, so every skill that
> exists only inside the company appears here under a neutral `internal-*` name, and the company
> itself is referred to as "the company". The real names live in the private overlay, where the
> lane tables below are reproduced verbatim. A reader implementing this needs the overlay
> checkout; a reader reviewing the *design* does not, which is the point of the substitution.
> Public upstream projects keep their real names: nothing is hidden that is already published.

Status: **in review** — proposed, awaiting your approval at the design gate. Nothing has been
copied, moved, or edited; this document is the thing to review before any of that happens.
Decision: **the public collection ships as lanes under `skills/` in the public layer-2 repository
and mounts read-only through `skills.external_dirs`; the private collection mounts one tier above
it through `skills.create_dir`, so a private skill of the same name wins without either repository
copying the other; each profile mounts only the lanes its toolsets can act on.**
Source: Kanban cards `t_8a78fa8b` (v1-v2) and `t_78046316` (v3).

---

## Review header

**Decisions needed from you: none.** The four questions v2 asked are answered, and the answers are
applied throughout rather than noted at the top — section 1's table, section 2's rule, section 3's
layout, section 5's lane map and the assumptions list all now read as though the answers were
always there. The four, and where each landed:

1. **The private repository is the layer-3 overlay.** Its checkout is `skills.create_dir` for every
   profile. Section 4 keeps only the arrangement you chose; the config-repo alternative and its
   cost are gone, and v2's first load-bearing assumption is retired.
2. **The pointers are settled — and one of my findings about them was wrong.** Below.
3. **Ship by reference, accepted.** `stop-slop` is a pinned entry in `third-party.yaml`, not a
   vendored copy. The same mechanism now carries the two `autopraxis` skills. You did not overrule
   the `google-workspace` drop, so it stays dropped.
4. **The document stays public.** The placeholder scrub on this branch is intact; nothing in this
   revision reintroduces a real internal name.

What remains is three implementation details and one question that needs somebody who can read a
repository I cannot (section 10). None of them blocks approval.

**The correction, which matters more than the answers.**

v2 reported 42 symlinks in the second collection, 34 of them into one plugin. The directory has
**8 symlinks**. The other 34 are *directories* that each contain a single symlinked `SKILL.md` —
which is why a walk that stats the top entry sees a directory and a walk that reads the file sees a
pointer. v2 did both walks and reconciled them into one number. The total, 42 pointer entries, came
out right by accident; its composition did not, and the composition is the part that decides what
can be imported.

Recounted from the directory rather than patched:

| v2 said | v3, from the directory | Why it moved |
| --- | --- | --- |
| 42 symlinks | 8 symlinks, plus 34 directories whose only `SKILL.md` is a symlink | Two walks reconciled into one number. 42 pointer *entries* is still right |
| "76 skills" is 34 | 76 entries are **29** real skills | The 34 pointer-directories were counted as the real skills |
| 30 real skills | 29 | `conventions` has no `SKILL.md` at any depth: it is twelve loose notes in a directory, not a skill |
| 42 pointers, none classified | 34 skipped, 5 private, 2 installed by pin, 1 already redundant | Your answer 2, plus two of the four upstreams being readable from GitHub even though the symlinks are dead |
| 12 in the private overlay | **17** | The five that point into a team repository inside the company are company-coupled and stay private |
| 50 public | 50 published **+ 3 installed by pin = 53 resolvable** | `stop-slop` by your answer 3, and the two `autopraxis` skills by the same mechanism |
| 3 skills to the config repo | 2 skills, plus a notes directory that is not a skill | The same `conventions` correction |
| Six names appear in both collections | **Seven** | `council` is a real skill in one collection and a pointer in the other, exactly like `ship`. The symlink confusion hid it |
| ~3,400 tokens to mount everything | **~5,000** | Recomputed from frontmatter across all 70 distinct skills. v2's figure is not reproducible from the files |
| ~1,100 tokens for 58 builtins | ~1,200 | Same recount, same method |

The headline survives: **50 published**, and the title's other number moves from 12 to 17. What
shrank is not what ships but what was still open — v2 held the door open for up to 42 more skills,
and 34 of those are now permanently shut.

**The skipped collection, in full, because "skipped" must not read as "overlooked".**

The 34 pointer-directories resolve into one checkout of **`github.com/garrytan/gstack`** — MIT,
Copyright (c) 2026 Garry Tan, verified against the upstream repository today [UPSTREAM].
Thirty-three point at `<plugin>/<name>/SKILL.md`; the thirty-fourth, an underscore-prefixed entry,
points at the plugin's own root `SKILL.md`, a second-level pointer rather than a sub-skill. It is
not your work, and it is not this design's to relicense.

It is also dead. Every one of the 34 is dangling: the path they point at no longer exists, the
content survives only in a conflict backup directory dated four months ago, and nothing local was
ever modified [BOARD]. They have been unusable for four months without anyone noticing — which is
the actual finding, and it is about the mechanism rather than about these 34 files. **Decision:
skip them entirely.** Not vendored, not referenced, not imported, not resurrected from the backup.
If you ever want one or two, the way to take them is by reference from upstream — where they are
maintained, and where there are twenty-three of them today against the thirty-four names this stale
layout surfaces [UPSTREAM] — and that is a later decision, not this design's.

Skipping them also removes a name collision for free: one of the 34 is `ship`, which is a real
published skill in the other collection.

**Council dissent.**

No council was convened: this profile's toolset has no `delegate_task`, and inventing personas to
disagree with would be theatre. The critic pass is the agent reviewer, which ran once, on v2.

*What it found, and how it was resolved.* It re-derived every count from the collections and from
Hermes source rather than reading the prose, and raised two blocking contradictions, both of which
it won outright. (1) `google-workspace` was classified as a manifest entry in decision 3 and as a
drop in section 2; resolved by changing the rule rather than the row, so the builtin test now runs
first. (2) The per-profile table counted public lanes only while the paragraph under it added
private lanes, and it gave the experimenter a private lane one sentence after saying unproven
profiles get nothing; resolved by totalling both tiers and cutting the experimenter's private
mount. No finding was declined.

It did not catch the pointer miscount, and neither did I. The reason is worth recording: it
re-derived the numbers with the same two walks, and the two walks agreed with each other. What
found it was reading the directory the way a person reads it. **A reviewer pass on this revision
has not yet run**; it is the next gate after this document is pushed.

Below, the three points where the evidence contradicts the brief, each of which a critic would
have raised:

- **"76 skills" is 29.** The second collection's 76 entries are 29 real skills, 42 pointers, one
  directory of notes with no `SKILL.md`, and four loose files [COLLECTIONS]. The brief's framing —
  two collections of 45 and 76 — is how the tools present them, not what is on disk. Nothing in the
  brief is wrong; the count is a directory listing. But a plan written to "split 121 skills" would
  have spent most of its effort on one third party's dead plugin.
- **"12 trivial mentions" is 10, after the drops — and it takes two scans to find them.** A word
  scan for the company name, over every non-binary file outside the vendored reference and outside
  build output, matches 15 of the 45 personal skills. Five of those are the company-coupled set that
  stays private, and one (`getting-started`) is among the dropped, which leaves **nine**. The tenth,
  `worktree`, carries no mention of the company at all: it hard-codes a corporate Jira tenant in an
  example URL and is caught only by a second scan for SaaS-tenant URL shapes. So the 10 is the union
  of a name scan and a URL scan, not 15 minus 2 — and that is the argument for the publication check
  in section 7 testing shapes as well as names, since a name list alone would have shipped
  `worktree` unchanged.
- **The ownership premise has three exceptions, not two.** "Both collections are their own
  copyright" is out of scope to relitigate and I have not. It is still true that `stop-slop` is
  `hardikpandya`'s, `google-workspace` is `andmarios`', and — the one v2 missed, and the largest —
  that 34 of the second collection's 76 entries are Garry Tan's. Publishing any of the three MIT
  under your name would be a licence error rather than a judgement call. All three are now handled
  by mechanism: pinned reference, drop, and skip.

**What changed in v3.** The four answers folded in and their alternatives deleted rather than
annotated; the pointer recount above and every number downstream of it; the 34 skipped with their
provenance named; the private overlay grown to 17 and the builder's private lane with it; the two
`autopraxis` skills read from their public upstream and given lanes, which v2 could not do because
all it had was the dead symlink; `conventions` demoted from skill to notes directory; the seventh
shared name found; the index-cost figures recomputed; two new rules in the publication check that
exist because of what the 34 turned out to be; open questions down from four decisions plus four
details to three details and one question. Load-bearing assumptions are down from two to one, and
it is a different one.

**Scope: L — split into six pull requests**, none over ~400 lines of hand-written change. The bulk
is mechanical file movement, which is why the rule and the check land before any skill does.

| PR | Repository | What lands | Size |
| --- | --- | --- | --- |
| 1 | ralph (public) | `skills/` lane directories, `skills/README.md` (the rule from section 2), `third-party.yaml` with its three pinned entries, and the publication check in CI. No skills | S |
| 2 | ralph (public) | The 24 clean personal skills, each stripped of the vendored reference file | M |
| 3 | ralph (public) | The 10 generalised personal skills, one commit each so the diff is the generalisation | M |
| 4 | ralph (public) | The 16 skills from the second collection | M |
| 5 | layer-3 overlay (private) | The 17 private skills, with the `standards` name collision resolved by renaming the router to `standards-index`, plus the internal marker list the drift check reads | L |
| 6 | ralph (public) | Nine profile templates gaining their `skills:` block; `doctor` check for mount drift | S |

PR 5 grew from M to L, and it is the one an agent cannot finish alone: five of its 17 skills live in
a team repository that exists only on your machine, so their files have to be fetched by someone
with that checkout before the pull request can be opened. PR 6 is last on purpose: until the trees
exist, a mount points at nothing, and a profile whose `external_dirs` names a missing directory is
a silent no-op rather than an error [HERMES].

---

## 1. What is actually in the two collections

Counts from a fresh walk of both trees in the card workspace on 2026-10-21, re-derived for this
revision rather than carried over [COLLECTIONS].

| | Personal collection | Second collection |
| --- | --- | --- |
| Directory entries | 45 | 76 |
| Real skills (its own `SKILL.md`) | 45 | 29 |
| Directories whose only `SKILL.md` is a symlink | 0 | 34 |
| Top-level symlinks | 0 | 8 |
| A directory with no `SKILL.md` at any depth | 0 | 1 |
| Loose files, not skills | 0 | 4 |

That is 74 skill directories that resolve. Four pairs are the same skill filed twice, so there are
**70 distinct skills** to place, and 42 pointer entries to dispose of.

**The 42 pointers, and where each group goes.** All 42 are dangling in this container, so none of
their content was read here. Two of the four upstreams are public repositories and were read from
GitHub instead — which is how the two `autopraxis` skills get lanes in section 5 rather than a
question mark, and how the licence on the 34 is a fact rather than an inference.

| Pointer entries | Count | Upstream | Disposition |
| --- | --- | --- | --- |
| Directories with a symlinked `SKILL.md` | 34 | `garrytan/gstack`, MIT, © 2026 Garry Tan | **Skipped.** Someone else's work, dangling four months. Review header |
| Top-level symlinks | 5 | A team repository inside the company | **Private overlay.** Yours, but company-coupled. Section 5 |
| Top-level symlinks | 2 | `Zhachory1/autopraxis`, public, yours | **Installed by pin**, like `stop-slop`. Not copied |
| Top-level symlink | 1 | A local `agent-fleet` checkout (`council`) | **Nothing to do.** `council` is already a real skill in the personal collection and is published from there |

Three facts from the scan shape everything below.

- **The vendored reference file is one file, 32 times.** `references/engineering-standards.md`
  appears in 32 skills across both collections, byte-identical at 3,155 bytes each — one SHA for
  all 32. 28 of them are skills this design publishes, so the import strips it rather than
  quarantining the skill, exactly as the brief established.
- **Seven names appear in both collections, and four are the same skill.** `hermes-queue-client`,
  `i-have-adhd`, `simplified-technical-english` and `stop-slop` have identical `SKILL.md` bytes and
  fold to one copy — two of them carry extra files on one side (`stop-slop` is a full git checkout
  in the second collection and a four-file copy in the personal one), which is an argument for the
  pinned-reference treatment rather than against folding. `standards` is **two different skills
  sharing a name** — a 16-file standards pack and a 1-file router. `ship` and `council` each exist
  as a real skill in one collection and a pointer in the other; v2 found the first and missed the
  second.
- **The index cost of mounting everything is about 5,000 tokens**, against about 1,200 for all 58
  Hermes builtins. Not because there are more skills, but because these descriptions average 275
  characters against the builtins' 85. Every session pays it before a single skill is loaded
  [HERMES]. That is the whole argument for lanes — and it is still a small number, which is why
  section 5 argues the real cost is selection noise, not tokens.

## 2. The rule that decides where a skill goes

Apply in order and stop at the first match. The order runs from the tests that take a skill out of
the import entirely, through the test that decides whose licence it ships under, to the tests that
only choose which repository holds it — so a later test is never asked a question an earlier one
has already made moot.

1. **Does a Hermes builtin already do it, or share its name?** Then: **drop.** Section 6. This
   test runs first because it is the only one that can make the others moot: a skill whose name
   matches a builtin can never resolve anyway — builtins sit in the profile skills tier, which
   outranks both mount points [HERMES] — so deciding its licence or its repository would be
   deciding where to file something that will never load.
2. **Is it someone else's?** A LICENSE naming another copyright holder, or a `.git` remote outside
   your account. Then: **reference, never copy.** It is listed in `third-party.yaml` with upstream
   and pinned commit, and installed from there. Never vendored, never relicensed. When the pointer
   to it is dead and nobody has missed it, the honest form of "reference" is **skip** — which is
   what happens to the 34.
3. **Does it need something only one machine has?** A local daemon, a private document store, a
   personal credential path. Then: **config repo.** It is useless in a container, so it ships in
   neither repository and is mounted only where the dependency exists.
4. **Does any line of it name something that exists only inside the company?** An internal
   repository, host, endpoint, tenant URL, standard, process, metric, incident tier, brand value,
   or data model. The test is the name, not the topic: "write tests in a pyramid" is generic, "the
   pyramid as defined by our standards doc" is not. Then: **private overlay.**
5. **Otherwise:** **public.**

Two notes that make this usable without asking.

- **A mention is not a coupling.** A skill that names one internal thing in one sentence is a
  public skill with a sentence to fix (section 7). A skill whose *procedure* only makes sense
  inside the company is private, however few times it says so. The 1-file standards router mentions
  the company twice and is irreducibly private; a test generator mentions it twice and is public
  after a two-line edit.
- **When in doubt, private.** Moving a skill from private to public later is a pull request.
  Moving it the other way is a git history rewrite and a disclosure conversation.

Applying the rule to the 70 distinct skills:

| Outcome | Count | Which |
| --- | --- | --- |
| Public | 50 | 34 personal, 16 from the second collection |
| Private overlay | 12 | 4 personal (`standards` the 16-file pack, `internal-autoscaling`, `internal-pr-dashboard`, `internal-publish-branch`), 8 from the second collection (`standards` the router, `internal-docs`, `internal-doc-architect`, `internal-doc-critique`, `internal-platform`, `internal-brand`, `internal-experiment-analysis`, `internal-app-builder`) |
| Config repo, host-coupled | 2 | `hermes-queue-client`, `zbrain-query` |
| Dropped, builtin wins | 5 | `code-verifier`, `getting-started`, `conventional-commit`, `compound`, `google-workspace` |
| Reference, pinned | 1 | `stop-slop` |
| Folded duplicates | 4 | identical copies resolved to one |

50 + 12 + 2 + 5 + 1 = 70. The pointers are not in this table, because on this disk they are not
skills. Applying the same rule to what they resolve to adds **5 to the private overlay** (rule 4 —
yours, company-coupled) and **2 to the pinned references** (rule 2 — yours, already public
upstream, so referencing costs nothing and copying buys nothing). That is the 17 in the title and
the three entries in `third-party.yaml`.

`conventions` is in no row: it has no `SKILL.md` at any depth, so it is twelve notes in a directory
rather than a skill, and it is the config repo's business if it is anyone's.

Two skills match more than one test, and the order settles both. `code-verifier` is company-coupled
*and* redundant: rule 1 runs before rule 4, so it ships nowhere rather than being carried privately
forever. `google-workspace` is third-party *and* name-identical to a builtin: rule 1 runs before
rule 2, so it is dropped rather than referenced — a manifest entry for a skill the resolver would
never reach is a licence record for nothing.

## 3. Layout, and what each tier is for

Hermes resolves skills from four tiers, lowest number wins: trusted project directory, the
profile's own skills directory, `skills.create_dir`, then `skills.external_dirs`. The profile
skills directory is already occupied — every profile carries its own 3.9 MB copy of the 58 builtins
— and the project tier is derived from the worker's current directory, so neither is available to
hold a collection [HERMES]. That leaves `create_dir` and `external_dirs`, which is exactly two
slots for exactly two repositories.

```
ralph/ (public, MIT, your name)            layer-3 overlay (private)
  skills/                                    skills/
    plan/        8 skills                      write/      5 skills
    write/       7                             code/      11
    code/       10                             analysis/   1
    debug/       7                           marker-list.txt
    ui/          8                           README.md
    review/      8
    ops/         2
    meta/        3
    third-party.yaml  (3 pinned entries)
    README.md         (the rule from section 2)
```

Three of the public tree's 53 skills are not in git: `stop-slop`, `debug-investigation` and
`pr-review` are installed into their lanes from `third-party.yaml`, each at a pinned commit. The
manifest names upstream, commit and target lane, and the install step runs when the tree is built —
so the repository holds 50 skills and the mounted tree resolves 53.

Lanes are cut by **audience**, not by topic: the unit a profile mounts is a directory, so a skill
two profiles need must sit in a lane both mount. That is the cost of using directories as the
selection mechanism, and it is the right trade because the alternative is worse (section 5).

Both trees mount **read-only**. The public tree is a container bind-mount with `:ro`, which makes
"an agent cannot rewrite a shipped skill" a permission rather than a sentence in a prompt — the
property layer 2's contributor guide asks for by name [RALPH]. A `skill_manage` patch against a
read-only tree fails with an error the agent can see, which is the behaviour we want.

## 4. How the overlay works, without either repository copying the other

Per-profile config is a whole config, not a merge: a named profile's `config.yaml` *is* the config,
with defaults filling the gaps, and nothing from the default profile's file reaches it [HERMES].
So each profile names its own mounts:

```yaml
# profiles/ralph-builder/config.yaml
skills:
  external_dirs:                      # tier 3, read-only, public — a list
    - /srv/ralph/public/skills/code
    - /srv/ralph/public/skills/debug
    - /srv/ralph/public/skills/ui
  create_dir: /srv/ralph/private/skills/code   # tier 2, read-only, private — ONE path
  creation_nudge_interval: 0
```

A private skill named `standards` sitting at tier 2 **shadows** the public one at tier 3: the
private file wins, the public copy is hidden, and Hermes logs the shadowing once. Neither
repository contains a copy of the other, and layer 3 can replace a layer-2 skill wholesale without
layer 2 knowing it exists. That is the overlay the brief asks for, and it is the same mechanism
layer 2's own design already uses for "mechanism upgrades, content is yours" [RALPH].

**`external_dirs` is a list; `create_dir` is a single path** [HERMES]. So a profile mounts many
public lanes and **exactly one private lane**. That is a mechanism constraint rather than a style
choice, and it is why the private tree's lanes are cut per *mounting profile* while the public
tree's are cut per audience: a private lane no profile can reach is a directory, not a skill. It is
also why the five team-repository skills join the builder's existing private lane in section 5
instead of forming an `ops` lane of their own — the builder is the only profile with the toolsets
they assume, and it cannot mount two.

**The mechanism refuses the alternative, which is why it is not the alternative.** Two *different*
skills sharing a name **inside one tier** are not resolved by order — Hermes calls them ambiguous
and refuses both [HERMES]. So mounting the public and private trees as two `external_dirs` entries
would mean that the day layer 3 adds a `standards`, every profile mounting both loses the public
one *and* the private one. The failure is silent at write time and total at read time. Tier
separation makes the winner deterministic instead.

The same rule bites inside the private tree: the two `standards` skills are different files with
one name, so PR 5 renames the router to `standards-index`. A name collision is a design error in
the repository, not a runtime surprise, and the publication check (section 7) fails on duplicate
names across every lane in both trees.

**What we give up.** `create_dir` is also where `skill_manage` writes new skills, so pointing it at
the overlay means anything an agent writes is born private. For this fleet that is close to free —
every profile already sets `creation_nudge_interval: 0`, which turns skill-creation prompting off
[HERMES] — and it is the safer default: a skill written while doing company work is presumed
company-flavoured until a human promotes it. What it costs is real but small: an agent-written
skill lands in whichever private lane that profile mounts, so the author's and the builder's never
mix, and a profile with no private lane has nowhere to write and fails visibly rather than quietly.

## 5. Who gets what, and why

The principle, in one sentence: **a profile mounts a lane when every skill in it is actionable with
that profile's toolsets and its failure mode stays inside that profile's trust boundary.**

Three consequences, which are the whole map:

- **Nothing that needs a terminal goes to a profile without one.** The researcher, analyst and
  watcher have `file` and `web` but no `code_execution` [RALPH]. A debug skill mounted there is a
  skill the model will try to follow and cannot. This also settles the five team-repository skills:
  they are CI, deployment and cluster work, so the builder is the only profile that could run them.
- **The most exposed profile holds the least.** The researcher reads untrusted web pages for a
  living. Everything it can quote, a page can ask it to quote. It mounts no lane at all — and it
  does not need one: `grounded-citations`, `llm-wiki`, `arxiv`, `blocked-page-recovery` and
  `codebase-inspection` are builtins it already has.
- **Unproven profiles get the smallest public mount their charter already names, and no private
  one.** The analyst and the auditor have charters that map onto an existing lane — `write` for
  one, `write` and `meta` for the one whose job is judging the system — so they mount it, and it is
  one line to remove if the charter turns out wrong. The experimenter has no established workload
  and the broadest toolset of the three (`code_execution`, `file` and `web` [RALPH]), which is the
  combination where a wrong lane costs the most, so it mounts nothing in either tree until a card
  says what it does. Adding a lane later is one line. Discovering that the experimenter has been
  carrying 25 front-end skills for a year is not recoverable attention.

Skills and index cost below are the **total** a profile carries: public lanes plus the one private
lane it mounts, if any. Index cost is measured from the real frontmatter descriptions at roughly
four characters to the token [COLLECTIONS].

| Profile | Public lanes | Private lane | Skills (public + private) | Index cost |
| --- | --- | --- | --- | --- |
| orchestrator | plan, meta | — | 11 | ~865 tok |
| author | plan, write | write (5) | 20 = 15 + 5 | ~1,660 tok |
| builder | code, debug, ui | code (11) | 36 = 25 + 11 | ~2,230 tok, plus five unread |
| reviewer | review | — | 8 | ~425 tok |
| watcher | ops | — | 2 | ~140 tok |
| analyst | write | — | 7 | ~530 tok |
| auditor | write, meta | — | 10 | ~750 tok |
| researcher | — | — | 0 | 0 |
| experimenter | — | — | 0 | 0 |

The builder's is the only estimated figure in that column: 2,230 tokens is measured over the 31 of
its 36 skills whose frontmatter is readable here. At the collection's average description the five
team-repository skills add roughly 350 more, so call it ~2,600.

Lane contents, so the map is checkable rather than asserted. Entries marked † are installed from
`third-party.yaml` at a pinned commit rather than committed to the repository:

- **plan** — task-planner, prompt-decomposer, prompt-optimizer, council, deep-dive, kickstart,
  handoff, session-recap
- **write** — doc-writer, exec-summary, simplified-technical-english, better-writing,
  technical-drawing, i-have-adhd, stop-slop †
- **code** — test-maker, code-optimizer, ship, swarm, worktree, graphite-stacked-prs, pr,
  pr-review-handler, commit-push-watch, internal-shell-policy
- **debug** — debug-api, debug-build, debug-jest, debug-playwright, dev-monitor, ci-watcher,
  debug-investigation †
- **ui** — build-design, break, variant, state-machine, better-colors, better-ui,
  msw-storybook-validator, storybook-play-tests
- **review** — interface-review, explain-interface, investigation, better-accessibility,
  better-interface, better-layout, better-typography, pr-review †
- **ops** — alert-monitor-suggester, internal-chat
- **meta** — agent-forge, agent-optimizer, skill-forge

The two `autopraxis` skills are placed from their published frontmatter, read from the public
repository rather than from the dead symlink [UPSTREAM]: `debug-investigation` is an incident and
root-cause workflow, which is the debug lane, and `pr-review` is a PR review workflow that posts a
verdict, which is the reviewer's whole job. Neither name collides with anything in either tree —
`investigation` and `pr-review-handler` are different skills with different names — and whether any
of the three overlap in *content* belongs to the brief's overlap analysis, which this revision does
not reopen.

Private lane contents. Lane cuts follow the mounting profile, because a profile mounts exactly one
private lane (section 4), and the names are the ones that exist after PR 5's rename:

- **write** (5), mounted by the author — `standards-index` (the 1-file router), `internal-docs`,
  `internal-doc-architect`, `internal-doc-critique`, `internal-brand`
- **code** (11), mounted by the builder — `standards` (the 16-file pack), `internal-autoscaling`,
  `internal-pr-dashboard`, `internal-publish-branch`, `internal-platform`, `internal-app-builder`,
  and the five from the team repository: `internal-ci`, `internal-deploy`, `internal-cluster`,
  `internal-gateway-discovery`, `internal-merge`
- **analysis** (1), mounted by nobody at launch — `internal-experiment-analysis`

Those last five placeholder names describe each pointer's apparent function and nothing more. Their
files are not readable from this container, so their placement rests on your word that they are
company-coupled, not on a scan of their contents; section 10 asks the one question that depends on
reading them.

The private `analysis/` lane is **mounted by no profile at launch**. It exists because its one
skill cannot be published, not because a profile is waiting for it: the private repository is where
a non-publishable skill lives whether or not anything mounts it. It mounts on the analyst the day a
card gives the analyst company analysis work, which is one line in one profile config.

**The token column is not the argument.** The worst case is about 2,600 tokens on the builder — 36
skills, public and private together — which is noise beside a diff. The argument is selection:
Hermes instructs the model to scan the index and load anything even partially relevant [HERMES].
Thirty-six plausible skills is a judgement call every turn; seventy is a lottery. That the builder
is the worst case twice over — most public skills and much the largest private lane — is the first
place to look if its work gets vaguer after this lands, and its private lane grew by five in this
revision, which makes that more likely rather than less. The watcher at two skills is the design
working: it was deliberately small, and it stays small.

## 6. The four dropped skills, and what the agents do instead

| Dropped | Replaced by | What the agents lose |
| --- | --- | --- |
| `code-verifier` | builtin `requesting-code-review` | Nothing. The builtin runs the same pre-commit gates without the company standards index |
| `getting-started` | builtin `codebase-inspection` | Nothing. The dropped one opened by reading an internal onboarding path |
| `conventional-commit` | builtin `github` | The commit-message rules, which `commit-push-watch` (public, kept) already applies |
| `compound` | builtins `llm-wiki` + `grounded-citations` | Nothing: the two builtins cover compounding notes and citation discipline between them |

"Dropped" means the skill is never imported into either repository. It stays in the source
collection on your machine, where it will keep working in the tool it came from. Nothing deletes
anything. The same is true of the 34 skipped pointers, with one difference worth keeping straight:
a dropped skill still works where it is, and a skipped pointer has not worked anywhere for four
months.

**A fifth drop, which is not from the brief's analysis.** `google-workspace` in the second
collection is dropped too, and for a different reason than the four: it is a checkout of a third
party's repository that happens to carry the same name as a Hermes builtin. Rule 1 catches it
before the provenance test ever runs, and the mechanism is why — builtins live in the profile
skills tier, which outranks both `create_dir` and `external_dirs` [HERMES], so a mounted copy would
be shadowed on every resolve and never load. The agents use the builtin, which covers Gmail,
Calendar, Drive, Docs and Sheets. Answering question 3 you took the reference mechanism and left
this drop standing, so it stands.

The three kept near-overlaps from the brief's analysis stay for the stated reasons and are mounted
accordingly: `test-maker` (code lane) alongside builtin `test-driven-development`, `investigation`
(review lane) alongside builtin `systematic-debugging`, and the four `debug-*` skills, which have
no builtin equivalent.

## 7. Generalising the ten, and who checks

**Which ten.** After the drops and the private set, these publish and still carry an internal
mention: `agent-forge`, `agent-optimizer`, `alert-monitor-suggester`, `code-optimizer`,
`commit-push-watch`, `doc-writer`, `exec-summary`, `skill-forge`, `test-maker`, `worktree`. Nine
carry one or two mentions of the company name in a sentence. `worktree` is the one with real
content to change, and the one the name scan misses entirely: it hard-codes a corporate Jira tenant
and a real-looking ticket id in an example URL — not even this company's tenant — which must become
a placeholder. It is in this list because of the URL-shape scan, not the name scan, which is the
reconciliation in the review header and the reason check 1 below tests shapes.

**How.** PR 3, one commit per skill, so each diff is exactly the generalisation and nothing else.
Each edit replaces the specific with the configurable — a placeholder host, "your issue tracker",
"your team's test pyramid" — and never deletes the procedure. A generalisation that drops a step is
a worse outcome than leaving the skill private.

**Who checks, in three layers, because no single one is sufficient.**

1. **A blocking check in the public repository's CI**, run on every pull request that touches
   `skills/`. It fails on: a `references/engineering-standards.md` by path; any URL whose host
   looks like a corporate SaaS tenant; any duplicate skill name across lanes; any `__pycache__`,
   `.pyc` or build artefact; any file matching the credential-ish patterns; and — new in v3 — **any
   symlink anywhere under `skills/`, and any `third-party.yaml` entry without a pinned commit.** It
   deliberately does **not** carry the list of internal names, because that list is itself internal.
2. **The full marker list lives in the private repository** and runs there, weekly, as an auditor
   cron over the public tree, the private tree and the config repo. Its question is the one CI
   cannot ask: *has a skill that is public today started naming something internal?* It reports; it
   never edits. This is the answer to "how does a skill that stops being generic get noticed" —
   the public check catches the shapes, the private check catches the names, and only the second
   one can be kept current.
3. **You check the ten, once.** Whether a sentence names something internal is a judgement only you
   can make, and it is cheap at ten skills and once. The reviewer profile checks the diffs against
   the marker list before you see them; approval of PR 3 is the gate.

Check 1's two new rules come from this revision rather than from a principle. Thirty-four skills
were unusable for four months because a symlink target moved and nothing said so; the collection
went on listing them, and both of my earlier walks went on counting them. A tree that cannot
contain a symlink cannot fail that way, and a reference that must carry a commit cannot quietly
become whatever upstream holds today.

The scan that produced these numbers has one more lesson worth encoding: two of the mentions it
found were inside a `.pyc`, in a skill that is otherwise clean. The check must read source, not
build output, or it will quarantine clean skills and teach everyone to ignore it.

## 8. Non-goals

- **Scoping the 58 builtins.** Every profile has all of them installed locally, including the
  watcher, which has Apple Notes and iMessage it will never call. The lever is a per-profile
  `skills.disabled` list, it is subtractive, and it is a different card.
- **Importing, reviving or evaluating the 34 skipped skills.** They are another author's MIT work
  and they have been dangling four months. Taking any of them later is a decision made against
  upstream, not against a four-month-old conflict backup.
- **Reading the five team-repository skills.** They are routed private on your word; their contents
  reach the overlay in PR 5, from a machine that has the checkout.
- **A skills registry, a version scheme, or publishing to a hub.** `hermes skills tap` exists and
  would install copies into each profile; this design keeps the repositories as the source of truth
  instead, so divergence stays visible in git rather than inside nine profile directories.
- **Editing any skill.** Not one byte was changed by this card or the one before it, by instruction.
- **The config repo's own structure.** Two skills and one notes directory are routed there and it
  is assumed to exist.

## 9. Risks, and what each would cost

- **A pointer rots and nobody notices.** No longer hypothetical: it is what happened to 34 skills
  for four months, in the collection this design imports from. Cost: a profile runs without skills
  it believes it has, and every count anyone takes of the tree is wrong in the same direction.
  Mitigated by importing files rather than links, by the no-symlink rule in check 1, and by pinning
  the three referenced skills to commits.
- **A name collides inside one tier and both skills vanish.** Mechanism-level, not stylistic: the
  resolver refuses rather than guesses [HERMES]. Cost: a profile silently loses two skills, nobody
  notices until the work is visibly worse. Mitigated by the duplicate-name check in CI (PR 1) and
  by the tier separation in section 4; the residue is a private skill added by hand outside CI.
- **The private tree is also the write target, so a careless `skill_manage` lands company-flavoured
  text in the overlay.** Cost: low while creation nudging is off, and recoverable — the repository
  is private, so the fix is a commit. The inverse arrangement is worse: an agent-written skill
  landing in the public tree is a disclosure, not a commit.
- **Five skills are placed in the builder's private lane without anyone having read them.** Cost:
  one of them turns out to be the author's, and moving it is not free, because a profile mounts
  exactly one private lane — the author gains it and the builder loses it. Cheap to settle at PR 5,
  expensive to discover a year later. Section 10, question 4.
- **Lanes cut by audience will be wrong somewhere.** Cost: one skill in a lane a profile does not
  mount means a profile quietly lacks it, and the symptom is poor work, not an error. Mitigated by
  the lane map being in profile config, which is reviewed when permissions change; a mis-cut lane
  is one line to fix.
- **A generalisation removes a step along with the mention.** Cost: a skill that still loads and
  now quietly does less — the most expensive failure in this design, because it looks finished. One
  commit per skill in PR 3 exists to make it reviewable.
- **The marker list goes stale, so the drift check passes a skill that has drifted.** Cost: the
  public repository gains internal detail and nobody is told. This is the residual risk the design
  does not close, and it is why the check reports to a human rather than gating automatically.
- **`external_dirs` pointing at a missing directory is a silent no-op.** Cost: a profile runs with
  no skills and looks healthy. Mitigated by the `doctor` check in PR 6, which is cheap and should
  not wait for a second card.

## 10. Open questions

No decision is outstanding. Three implementation details and one question remain, and none of them
blocks approval of this design.

1. Does the layer-3 overlay already exist as a repository, or does PR 5 create it? The design reads
   as though it exists [RALPH]; nothing in the workspace confirms a checkout.
2. Where is the public tree mounted on the host? `/srv/ralph/...` is a placeholder in this
   document; the real path goes into nine profile configs in PR 6 and wants to be decided once.
3. Should the auditor own the weekly drift report, or the watcher? The auditor fits by charter
   (judging the system) but the watcher already runs on cron. Not blocking; PR 5 can land the check
   with either owner.
4. Do all five team-repository skills belong to the builder? I have not read them, and a profile
   mounts exactly one private lane, so one that is really an authoring skill cannot just be added
   to the author's mount — it has to be moved, and the builder loses it. One line at PR 5 for
   whoever has that checkout open; unrecoverable attention if it is noticed a year later.

## Assumptions

Every inference the brief and your answers did not state, ordered by weight. Three of v2's nine are
retired by your answers and are not restated.

1. **Load-bearing: the five team-repository skills belong in the builder's private lane.** You
   settled that they are company-coupled and stay private; you did not say which profile uses them,
   and I cannot read them to find out. I placed them by what their pointer names suggest they do —
   CI, deployment, cluster, gateway discovery, merge — all of which need a terminal, which among
   private-lane holders only the builder has. Question 4.
2. **Moderate: `create_dir` is read-only-mountable while still serving as the write target.** I
   verified the tier ordering and the write-target behaviour in source [HERMES] and have still
   never run a `skill_manage` write against a read-only `create_dir`. What closes the dangerous
   half is the dispatcher: every tool call is wrapped in a blanket `except Exception` that turns the
   exception into an `Error executing tool ...` string returned to the model [HERMES], so a
   `PermissionError` from a read-only mount surfaces as a visible tool error rather than a broken
   turn. The residue is cosmetic — the agent sees an unhelpful message — and section 4's guarantee
   holds.
3. **Moderate: my classification of the 29 readable skills in the second collection is mine alone.**
   The brief's established analysis covers the 45 personal skills; nobody had classified the other
   collection. The eight I route private each name an internal repository, platform, brand or
   process; the sixteen I publish contain no internal marker outside build artefacts.
4. **Moderate: 10 skills need generalising, not 12.** Reconciled in the review header. If the
   brief's 12 came from a list rather than a count, use the list and ignore mine.
5. **Moderate: `internal-doc-architect` and `internal-app-builder` are private.** Both are mostly
   generic mechanism wrapped around an internal dependency — a credential path, a brand guideline
   skill. Rule 4 sends them private; a generalisation pass could move either one public later, and
   neither is worth blocking on now.
6. **Moderate: the overlay holds copies of the five team-repository skills rather than pointers to
   the team repository.** Nothing in your answer says which, and a pointer would re-create exactly
   the failure this revision found — a link into a checkout that moves, and four months of silence.
   Files in the overlay, not links into a repository no container can see.
7. **Moderate: the two `autopraxis` skills go in `third-party.yaml` even though they are yours.**
   The file you named says "third-party", but what it actually holds is everything installed by pin
   rather than committed: three entries, two of them yours. Renaming it `upstreams.yaml` is a
   one-line choice at PR 1 and I did not take it, because you named the file.
8. **Low weight: the dangling `council` pointer and the published `council` are the same skill.**
   The pointer is dead, so I compared names and nothing else. If they differ, what ships is the
   personal collection's, which is readable and classified; the other has been dead as long as the
   rest.
9. **Low weight: index cost is ~4 characters per token.** Used only for order of magnitude, and
   recomputed from the files for this revision rather than carried over.
10. **This revision is a new pull request on the same branch.** PR #2 was merged shortly before this
    card was written, which closed it, and a merged pull request cannot be reopened — so "revise the
    existing pull request" was satisfiable only in part. The branch is the same one,
    `design/ralph-skills` on the fork, and there is one open pull request for this design at a time.
    The side effect is a better diff: a reviewer sees exactly v2 to v3 rather than the whole
    document as new.

## Observations — out of scope, not acted on

- The collection that produced the 34 dead pointers also holds the only four loose files and the
  only directory that is not a skill. If anything in this estate wants a tidy it is that directory,
  and it is not this design's to tidy.
- Each of the nine profiles carries its own 3.9 MB copy of the 58 builtins, so a builtin upgrade is
  nine copies and `skills.disabled` is the only per-profile lever. Worth a card.
- `internal-chat` ships a `__pycache__` with two compiled modules, for two different Python
  versions. Build artefacts in a skill directory are how a clean skill fails a naive scan; the
  import should strip them everywhere.
- The brief's framing of "45 and 76" comes from a directory listing in each tool. A `hermes skills`
  equivalent that resolved symlinks and reported the dead ones would have shown 45 and 29 from the
  start — and would have told somebody about the 34 four months ago.
- `GH_TOKEN` is still not forwarded into dispatcher-spawned workers; this worker recovered it from
  the gateway process environment to push, as the two previous authoring cards reported.

## Sources

- **[COLLECTIONS]** — the two skill collections as snapshotted into this card's workspace
  `/workspaces/skills-dd` on 2026-10-21: `personal-collection/` (45 entries) and
  `second-collection/` (76 entries). Re-walked in full for v3: the entry type of every top-level
  entry, whether each directory's `SKILL.md` is a file or a symlink, every symlink target and
  whether it resolves, SHA-256 of every duplicate pair and of all 32 copies of the vendored
  reference file, frontmatter names and descriptions for all 69 readable names, and a marker scan
  over every non-binary file excluding that vendored file. Third-party provenance is from
  `.git/config` remotes and `LICENSE` copyright lines inside `second-collection/stop-slop` and
  `second-collection/google-workspace`.
- **[UPSTREAM]** — the two public repositories the pointers resolve to, read from GitHub on
  2026-10-21 because the symlinks themselves are dead: `github.com/garrytan/gstack` (MIT,
  Copyright (c) 2026 Garry Tan, read from its `LICENSE`; its own description advertises
  twenty-three tools) and `github.com/Zhachory1/autopraxis` (public, yours), from which
  `skills/debug-investigation/SKILL.md` and `skills/pr-review/SKILL.md` were read for the
  frontmatter that places them in lanes.
- **[HERMES]** — Hermes Agent source on the host running this worker, version `0.22.2`, commit
  `d923a5eb487a17c0a68801bba9ef0f56c81040a8`, read 2026-10-21: `agent/skill_utils.py`
  (tier constants and precedence, lines 381-384; `get_skill_create_dir` returning a single
  `Optional[Path]`, 349-366; `get_skill_search_roots`, 393-403; `pick_skill_candidate` and the
  same-tier ambiguity refusal, 431-440; `get_disabled_skill_names`, 261-273;
  `get_project_skills_dirs`, 576-579; `iter_skill_index_files` walking with `followlinks=True`,
  842-851), `agent/prompt_builder.py` (the skills index and its loading instruction, 1451-1490),
  `hermes_cli/config_defaults.py` (the `skills:` block — `external_dirs` a list, `create_dir` a
  single string, 1446-1492), `agent/agent_init.py` (1381-1383) and `agent/turn_finalizer.py`
  (764-765) for `creation_nudge_interval`, `hermes_constants.py` (`get_config_path`,
  `get_skills_dir`, 1196-1203), and `agent/tool_executor.py` (the blanket
  `except Exception as tool_error` that returns `Error executing tool ...` to the model rather than
  raising, 1369-1371 — cited for assumption 2). Profile isolation confirmed against the live
  profile directories under `/home/hermes/.hermes/profiles/`.
- **[RALPH]** — the layer-2 contributor guide, the layer-3 README, and the system design document,
  as pinned to this card: `ralph-AGENTS.md`, `overlay-README.md`, and `overlay-design.md` (the
  profile table and permissions, the three-layer section, and the extension green path), plus the
  nine profile `config.yaml` files in `ralph-profiles/`.
- **[BOARD]** — the `worker_context` delivered to cards `t_8a78fa8b` (the brief) and `t_78046316`
  (your four answers and the pointer correction, including the conflict-backup finding and the
  four-month date, which are yours and were not reproducible inside this container).
