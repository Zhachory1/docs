---
id: dd-ralph-skills
title: Publish 50 skills in ralph, keep 12 in the private overlay, and mount them per profile as read-only lanes
status: in review
owner: Zhachory1
card: t_8a78fa8b
revision: v1
date: 2026-10-21
prs:
  - Zhachory1/docs#2
---

# Publish 50 skills in ralph, keep 12 in the private overlay, and mount them per profile as read-only lanes

Status: **in review** — proposed, awaiting your approval at the design gate. Nothing has been
copied, moved, or edited; this document is the thing to review before any of that happens.
Decision: **the public collection ships as lanes under `skills/` in the public layer-2 repository
and mounts read-only through `skills.external_dirs`; the private collection mounts one tier above
it through `skills.create_dir`, so a private skill of the same name wins without either repository
copying the other; each profile mounts only the lanes its toolsets can act on.**
Source: Kanban card `t_8a78fa8b`, "Design: give Ralph's profiles their skills".

**This document is public.** `Zhachory1/docs` is a public repository, so internal identifiers found
in the collections are described here, never quoted: "names an internal knowledge-base repository",
not the repository. See decision 4 if you would rather have the evidence quotable.

---

## Review header

**Decisions needed from you (4).**

1. **Which repository is "the private one" — the layer-3 overlay, or your personal config repo?**
   The brief says "a public repository and a private one". The layer-2 contributor guide says a
   user's own skills live in their config repo, and the layer-3 README repeats it [RALPH]. Those
   are different repositories with different audiences: a Rokt org can read layer 3, only you can
   read your config repo. This design puts the 12 Rokt-coupled skills in layer 3 and makes that
   checkout the `skills.create_dir` for every profile, which also makes it the target for anything
   an agent writes. If you want agent-written skills to land somewhere only you can see, say so and
   the write target moves to the config repo — at the cost described in section 4, "What we give up".
2. **42 of the 76 "Rokt-flavoured" skills are pointers, not skills. Which upstreams do you want
   imported?** They are symlinks into four places on your Mac: 34 into one `gstack` plugin, 5 into
   a Rokt team repository, 1 into a local agent-fleet checkout, 2 into autopraxis [COLLECTIONS].
   I could not read any of them — the targets do not exist in this container — so I have classified
   nothing in that 42. The two autopraxis ones are already public upstream and should be mounted,
   not copied. The 5 from a Rokt team repository are the only things in either collection whose
   copyright plausibly is not yours, and I have not assumed otherwise.
3. **Two skills are someone else's MIT work and cannot ship under your name.** `stop-slop` is a
   checkout of `hardikpandya/stop-slop` and `google-workspace` is a checkout of
   `andmarios/google-workspace-skill`, both MIT with their own copyright lines [COLLECTIONS].
   Proposal: neither is vendored; they are listed in a `third-party.yaml` manifest with upstream URL
   and pinned commit, and installed rather than copied. Confirm, or say you would rather vendor them
   with their LICENSE files intact.
4. **Keep this design public, or move it?** Every internal identifier above is paraphrased, which
   costs you the ability to check my classification against the actual line. If you want the
   evidence quotable, this file moves to a private repository and only the rule (section 2) stays
   here.

**Council dissent.**

No council was convened: this profile's toolset this run has no `delegate_task`, and inventing
personas to disagree with would be theatre. The critic pass is the agent reviewer, which runs on
this card before you see it. In its place, the three points where the evidence contradicts the
brief, each of which a critic would have raised:

- **"76 skills" is 34.** The second collection has 34 directories, of which 4 are loose files, so
  30 are real skills; the other 42 entries are symlinks into four upstreams [COLLECTIONS]. The
  brief's framing — two collections of 45 and 76 — is how the tools present them, not what is on
  disk. Nothing in the brief is wrong; the count is a directory listing. But a plan written to
  "split 121 skills" would have spent most of its effort on things that are one plugin.
- **"12 trivial mentions" is 10, after the drops.** My mechanical scan finds 15 of the 45 personal
  skills carrying an internal marker outside the vendored file; 5 of those are the Rokt-coupled set
  and 2 more are in the four being dropped, leaving 10 that both ship publicly and need a sentence
  changed. Named in section 7. The difference is bookkeeping, not disagreement.
- **The ownership premise has two exceptions.** "Both collections are their own copyright" is out
  of scope to relitigate and I have not. It is still true that two directories are git checkouts of
  other people's repositories, and publishing them MIT under your name would be a licence error
  rather than a judgement call. Decision 3.

**Scope: L — split into six pull requests**, none over ~400 lines of hand-written change. The bulk
is mechanical file movement, which is why the rule and the check land before any skill does.

| PR | Repository | What lands | Size |
| --- | --- | --- | --- |
| 1 | ralph (public) | `skills/` lane directories, `skills/README.md` (the rule from section 2), `third-party.yaml`, and the publication check in CI. No skills | S |
| 2 | ralph (public) | The 24 clean personal skills, each stripped of the vendored reference file | M |
| 3 | ralph (public) | The 10 generalised personal skills, one commit each so the diff is the generalisation | M |
| 4 | ralph (public) | The 16 skills from the second collection | M |
| 5 | layer-3 overlay (private) | The 12 private skills, with the two `standards` collisions renamed, plus the internal marker list the drift check reads | M |
| 6 | ralph (public) | Nine profile templates gaining their `skills:` block; `doctor` check for mount drift | S |

PR 6 is last on purpose: until the trees exist, a mount points at nothing, and a profile whose
`external_dirs` names a missing directory is a silent no-op rather than an error [HERMES].

---

## 1. What is actually in the two collections

Counts verified by walking both trees in the card workspace on 2026-10-21 [COLLECTIONS].

| | Personal collection | Second collection |
| --- | --- | --- |
| Directory entries | 45 | 76 |
| Real skills (a `SKILL.md` that resolves) | 45 | 30 |
| Loose files, not skills | 0 | 4 |
| Symlinks into another tree | 0 | 42 |

The 42 pointers resolve to four upstreams: 34 to one `gstack` plugin whose sub-skills are each
surfaced as a top-level name, 5 to a Rokt team repository, 1 to a local agent-fleet checkout, 2 to
autopraxis. All 42 are dangling in this container, so their content was not read and nothing in
this design classifies them. Decision 2.

Three facts from the scan shape everything below.

- **The vendored reference file is one file, 32 times.** `references/engineering-standards.md`
  appears in 32 skills across both collections, byte-identical at 3,155 bytes each — one SHA for
  all 32. 28 of them are skills this design publishes, so the import strips it rather than
  quarantining the skill, exactly as the brief established.
- **Six names appear in both collections, and four are the same bytes.** `hermes-queue-client`,
  `i-have-adhd`, `simplified-technical-english` and `stop-slop` are byte-identical duplicates and
  fold to one copy. `standards` is **two different skills sharing a name** — a 16-file standards
  pack and a 1-file router — and `ship` exists as a real skill in one collection and a pointer in
  the other. This matters more than it looks: see section 4.
- **The index cost of mounting everything is about 3,400 tokens**, against about 1,100 for all 58
  Hermes builtins. Not because there are more skills, but because these descriptions average 295
  characters against the builtins' 75. Every session pays it before a single skill is loaded
  [HERMES]. That is the whole argument for lanes, and it is a small number — which is why section 5
  argues the real cost is selection noise, not tokens.

## 2. The rule that decides where a skill goes

Apply in order and stop at the first match. The order is deliberate: the earlier tests remove a
skill from publication entirely, so they must run before the test that only chooses a repository.

1. **Is it someone else's?** A LICENSE naming another copyright holder, or a `.git` remote outside
   your account. Then: **manifest.** It is listed in `third-party.yaml` with upstream and pinned
   commit, and installed from there. Never vendored, never relicensed.
2. **Does a Hermes builtin already do it?** Then: **drop.** Section 6.
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

Applying the rule to the 75 readable skills:

| Outcome | Count | Which |
| --- | --- | --- |
| Public | 50 | 34 personal, 16 from the second collection |
| Private overlay | 12 | 4 personal (`standards`, `keda-scaling`, `pr-dashboard`, `publish-branch`), 8 from the second collection (`ads-success-docs`, `doc-architect`, `doc-critique`, `rokt-ai-platform`, `rokt-brand-guidelines`, `rokt-experiment-analysis`, `spa-builder`, `standards`) |
| Config repo, host-coupled | 3 | `hermes-queue-client`, `zbrain-query`, `conventions` |
| Dropped, builtin wins | 5 | `code-verifier`, `getting-started`, `conventional-commit`, `compound`, `google-workspace` |
| Manifest, third-party | 1 | `stop-slop` |
| Folded duplicates | 4 | identical copies resolved to one |

`code-verifier` is both Rokt-coupled and redundant. Dropping wins: rule 2 runs before rule 4, so it
ships nowhere rather than being carried privately forever.

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
    plan/        8 skills                      write/     5 skills
    write/       6                             code/      6
    code/       10                             analysis/  1
    debug/       6                           marker-list.txt
    ui/          8                           README.md
    review/      7
    ops/         2
    meta/        3
    third-party.yaml
    README.md   (the rule from section 2)
```

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
  external_dirs:                      # tier 3, read-only, public
    - /srv/ralph/public/skills/code
    - /srv/ralph/public/skills/debug
    - /srv/ralph/public/skills/ui
  create_dir: /srv/ralph/private/skills/code   # tier 2, read-only, private overlay
  creation_nudge_interval: 0
```

A private skill named `standards` sitting at tier 2 **shadows** the public one at tier 3: the
private file wins, the public copy is hidden, and Hermes logs the shadowing once. Neither
repository contains a copy of the other, and layer 3 can replace a layer-2 skill wholesale without
layer 2 knowing it exists. That is the overlay the brief asks for, and it is the same mechanism
layer 2's own design already uses for "mechanism upgrades, content is yours" [RALPH].

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
the private overlay means anything an agent writes is born private. For this fleet that is close to
free — every profile already sets `creation_nudge_interval: 0`, which turns skill-creation
prompting off [HERMES] — and it is the safer default: a skill written while doing company work is
presumed company-flavoured until a human promotes it. If you would rather it land in your config
repo, decision 1, and then the private tree needs globally unique names enforced by CI in both
repositories, because it would be sharing tier 3 with the public one.

## 5. Who gets what, and why

The principle, in one sentence: **a profile mounts a lane when every skill in it is actionable with
that profile's toolsets and its failure mode stays inside that profile's trust boundary.**

Three consequences, which are the whole map:

- **Nothing that needs a terminal goes to a profile without one.** The researcher, analyst and
  watcher have `file` and `web` but no `code_execution` [RALPH]. A debug skill mounted there is a
  skill the model will try to follow and cannot.
- **The most exposed profile holds the least.** The researcher reads untrusted web pages for a
  living. Everything it can quote, a page can ask it to quote. It mounts no lane at all — and it
  does not need one: `grounded-citations`, `llm-wiki`, `arxiv`, `blocked-page-recovery` and
  `codebase-inspection` are builtins it already has.
- **Unproven profiles mount nothing until a card proves otherwise.** Adding a lane later is one
  line. Discovering that the experimenter has been carrying 24 front-end skills for a year is not
  recoverable attention.

| Profile | Lanes | Skills | Index cost |
| --- | --- | --- | --- |
| orchestrator | plan, meta | 11 | ~880 tok |
| author | plan, write | 14 | ~1,155 tok |
| builder | code, debug, ui | 24 | ~1,560 tok |
| reviewer | review | 7 | ~330 tok |
| watcher | ops | 2 | ~145 tok |
| analyst | write | 6 | ~500 tok |
| auditor | write, meta | 9 | ~730 tok |
| researcher | — | 0 | 0 |
| experimenter | — | 0 | 0 |

Lane contents, so the map is checkable rather than asserted:

- **plan** — task-planner, prompt-decomposer, prompt-optimizer, council, deep-dive, kickstart,
  handoff, session-recap
- **write** — doc-writer, exec-summary, simplified-technical-english, better-writing,
  technical-drawing, i-have-adhd
- **code** — test-maker, code-optimizer, ship, swarm, worktree, graphite-stacked-prs, pr,
  pr-review-handler, commit-push-watch, bash-last-policy
- **debug** — debug-api, debug-build, debug-jest, debug-playwright, dev-monitor, ci-watcher
- **ui** — build-design, break, variant, state-machine, better-colors, better-ui,
  msw-storybook-validator, storybook-play-tests
- **review** — interface-review, explain-interface, investigation, better-accessibility,
  better-interface, better-layout, better-typography
- **ops** — alert-monitor-suggester, google-chat
- **meta** — agent-forge, agent-optimizer, skill-forge

Private lanes mount on top for the profiles that do company work: `write/` on author, `code/` on
builder, `analysis/` on analyst and experimenter. The researcher never mounts a private lane, which
is the point of the third consequence.

**The token column is not the argument.** The worst case is 1,560 tokens on the builder, which is
noise beside a diff. The argument is selection: Hermes instructs the model to scan the index and
load anything even partially relevant [HERMES]. Twenty-four plausible skills is a judgement call
every turn; fifty is a lottery. The watcher at two skills is the design working — it was
deliberately small, and it stays small.

## 6. The four dropped skills, and what the agents do instead

| Dropped | Replaced by | What the agents lose |
| --- | --- | --- |
| `code-verifier` | builtin `requesting-code-review` | Nothing. The builtin runs the same pre-commit gates without the company standards index |
| `getting-started` | builtin `codebase-inspection` | Nothing. The dropped one opened by reading an internal onboarding path |
| `conventional-commit` | builtin `github` | The commit-message rules, which `commit-push-watch` (public, kept) already applies |
| `compound` | builtins `llm-wiki` + `grounded-citations` | Nothing: the two builtins cover compounding notes and citation discipline between them |
| `google-workspace` | builtin `google-workspace` | Nothing, and it resolves a name collision: the vendored copy is a third party's and shares a name with the builtin |

"Dropped" means the skill is never imported into either repository. It stays in the source
collection on your machine, where it will keep working in the tool it came from. Nothing deletes
anything.

The four kept near-overlaps from the brief's analysis are kept for the stated reasons and mounted
accordingly: `test-maker` (code lane) alongside builtin `test-driven-development`, `investigation`
(review lane) alongside builtin `systematic-debugging`, and the four `debug-*` skills, which have
no builtin equivalent.

## 7. Generalising the ten, and who checks

**Which ten.** After the drops and the private set, these publish and still carry an internal
mention: `agent-forge`, `agent-optimizer`, `alert-monitor-suggester`, `code-optimizer`,
`commit-push-watch`, `doc-writer`, `exec-summary`, `skill-forge`, `test-maker`, `worktree`. Nine
carry one or two mentions of the company name in a sentence. `worktree` is the one with real
content to change: it hard-codes a named corporate Jira tenant in an example URL, which must become
a placeholder.

**How.** PR 3, one commit per skill, so each diff is exactly the generalisation and nothing else.
Each edit replaces the specific with the configurable — a placeholder host, "your issue tracker",
"your team's test pyramid" — and never deletes the procedure. A generalisation that drops a step is
a worse outcome than leaving the skill private.

**Who checks, in three layers, because no single one is sufficient.**

1. **A blocking check in the public repository's CI**, run on every pull request that touches
   `skills/`. It fails on: a `references/engineering-standards.md` by path; any URL whose host
   looks like a corporate SaaS tenant; any duplicate skill name across lanes; any `__pycache__`,
   `.pyc` or build artefact; any file matching the credential-ish patterns. It deliberately does
   **not** carry the list of internal names, because that list is itself internal.
2. **The full marker list lives in the private repository** and runs there, weekly, as an auditor
   cron over the public tree, the private tree and the config repo. Its question is the one CI
   cannot ask: *has a skill that is public today started naming something internal?* It reports; it
   never edits. This is the answer to "how does a skill that stops being generic get noticed" —
   the public check catches the shapes, the private check catches the names, and only the second
   one can be kept current.
3. **You check the ten, once.** Whether a sentence names something internal is a judgement only you
   can make, and it is cheap at ten skills and once. The reviewer profile checks the diffs against
   the marker list before you see them; approval of PR 3 is the gate.

The scan that produced these numbers has one lesson worth encoding: two of the mentions it found
were inside a `.pyc`, in a skill that is otherwise clean. The check must read source, not build
output, or it will quarantine clean skills and teach everyone to ignore it.

## 8. Non-goals

- **Scoping the 58 builtins.** Every profile has all of them installed locally, including the
  watcher, which has Apple Notes and iMessage it will never call. The lever is a per-profile
  `skills.disabled` list, it is subtractive, and it is a different card.
- **Importing the 42 pointers.** Blocked on decision 2 and on content I cannot read.
- **A skills registry, a version scheme, or publishing to a hub.** `hermes skills tap` exists and
  would install copies into each profile; this design keeps the repositories as the source of truth
  instead, so divergence stays visible in git rather than inside nine profile directories.
- **Editing any skill.** Not one byte was changed by this card, by instruction.
- **The config repo's own structure.** Three skills are routed there and it is assumed to exist.

## 9. Risks, and what each would cost

- **A name collides inside one tier and both skills vanish.** Mechanism-level, not stylistic: the
  resolver refuses rather than guesses [HERMES]. Cost: a profile silently loses two skills, nobody
  notices until the work is visibly worse. Mitigated by the duplicate-name check in CI (PR 1) and
  by the tier separation in section 4; the residue is a private skill added by hand outside CI.
- **The private tree is also the write target, so a careless `skill_manage` lands company-flavoured
  text in the overlay.** Cost: low while creation nudging is off, and recoverable — the repository
  is private, so the fix is a commit. The inverse arrangement is worse: an agent-written skill
  landing in the public tree is a disclosure, not a commit.
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
- **Symlink loops.** Hermes walks skill directories with symlinks followed and no depth bound
  [HERMES]. A bundle-style layout of cross-linked directories could hang a scan. One reason this
  design uses plain directories, and a reason not to "fix" a mis-cut lane with a symlink.

## 10. Open questions

1. Decisions 1-4 above; 1 and 2 block PR 5 and the second collection's import respectively.
2. Does the layer-3 overlay already exist as a repository, or does PR 5 create it? The design reads
   as though it exists [RALPH]; nothing in the workspace confirms a checkout.
3. Where is the public tree mounted on the host? `/srv/ralph/...` is a placeholder in this
   document; the real path goes into nine profile configs in PR 6 and wants to be decided once.
4. Should the auditor own the weekly drift report, or the watcher? The auditor fits by charter
   (judging the system) but the watcher already runs on cron. Not blocking; PR 5 can land the check
   with either owner.

## Assumptions

Every inference the brief did not state, ordered by weight.

1. **Load-bearing: the private repository is the layer-3 overlay.** The brief says "a private one"
   and the layer-3 README claims the overlay role, but it also says a user's own skills belong in
   their config repo. I picked layer 3 because the 12 private skills are company-coupled rather
   than personal. Decision 1 exists because this is a guess about intent, not about mechanism.
2. **Load-bearing: nothing in the 42 pointers needs to ship.** I classified none of them. If the
   `gstack` plugin is the operator's own and is meant to come along, the import is a different
   shape — one plugin, not 34 skills — and section 3's lane counts change.
3. **Load-bearing: `create_dir` is read-only-mountable while still serving as the write target.** I
   verified the tier ordering and the write-target behaviour in source [HERMES], but did not run a
   `skill_manage` write against a read-only `create_dir`. If that errors in a way that breaks a
   turn rather than returning a tool error, the private mount must be writable and section 4's
   guarantee weakens to convention.
4. **Moderate: my classification of the 30 readable skills in the second collection is mine alone.**
   The brief's established analysis covers the 45 personal skills; nobody had classified the other
   collection. The eight I route private each name an internal repository, platform, brand or
   process; the sixteen I publish contain no internal marker outside build artefacts.
5. **Moderate: 10 skills need generalising, not 12.** Reconciled in the review header. If the
   brief's 12 came from a list rather than a count, use the list and ignore mine.
6. **Moderate: `doc-architect` and `spa-builder` are private.** Both are mostly generic mechanism
   wrapped around an internal dependency — a credential path, a brand guideline skill. Rule 4 sends
   them private; a generalisation pass could move either one public later, and neither is worth
   blocking on now.
7. **Low weight: the two autopraxis-sourced pointers are already public upstream.** Read from the
   symlink target path, not from the repository.
8. **Low weight: index cost is ~4 characters per token.** Used only for order-of-magnitude.
9. **This pull request comes from a fork.** The credential available to me authenticates as
   `zhach1`, which has `push: false` on `Zhachory1/docs`, so the branch lives on a fork.

## Observations — out of scope, not acted on

- Each of the nine profiles carries its own 3.9 MB copy of the 58 builtins, so a builtin upgrade is
  nine copies and `skills.disabled` is the only per-profile lever. Worth a card.
- Four files in the second collection are not skills at all (a README, two filing-rule files, an
  output-rules file) but sit in the skills directory as though they were. They are the config
  repo's business if anything's.
- `google-chat` ships a `__pycache__` with two compiled modules. Build artefacts in a skill
  directory are how a clean skill fails a naive scan; the import should strip them everywhere.
- The brief's framing of "45 and 76" comes from a directory listing in each tool. A `hermes skills`
  equivalent that resolved symlinks would have shown 45 and 34 from the start.
- `GH_TOKEN` is still not forwarded into dispatcher-spawned workers; this worker recovered it from
  the gateway process environment to push, as the previous authoring card reported.

## Sources

- **[COLLECTIONS]** — the two skill collections as snapshotted into this card's workspace
  `/workspaces/skills-dd` on 2026-10-21: `mewrite-skills/` (45 entries) and `roktcode-skills/`
  (76 entries). Walked in full: entry types, symlink targets, `SKILL.md` presence, SHA-256 of every
  duplicate pair and of all 32 copies of the vendored reference file, frontmatter names and
  descriptions, and a marker scan over every non-binary file excluding that vendored file. The
  third-party provenance is from `.git/config` remotes and `LICENSE` copyright lines inside
  `roktcode-skills/stop-slop` and `roktcode-skills/google-workspace`.
- **[HERMES]** — Hermes Agent source on the host running this worker, version `0.22.2`, commit
  `d923a5eb487a17c0a68801bba9ef0f56c81040a8`, read 2026-10-21: `agent/skill_utils.py`
  (tier constants and precedence, lines 381-384; `get_skill_search_roots`, 393-403;
  `pick_skill_candidate` and the same-tier ambiguity refusal, 431-440; `get_disabled_skill_names`,
  261-273; `get_project_skills_dirs`, 576-579; `iter_skill_index_files` walking with
  `followlinks=True`, 842-851), `agent/prompt_builder.py` (the skills index and its loading
  instruction, 1451-1490), `hermes_cli/config_defaults.py` (the `skills:` block, 1448-1492),
  `agent/agent_init.py` (1381-1383) and `agent/turn_finalizer.py` (764-765) for
  `creation_nudge_interval`, `hermes_constants.py` (`get_config_path`, `get_skills_dir`, 1196-1203).
  Profile isolation confirmed against the live profile directories under
  `/home/hermes/.hermes/profiles/`.
- **[RALPH]** — the layer-2 contributor guide, the layer-3 README, and the system design document,
  as pinned to this card: `ralph-AGENTS.md`, `rokt-it-ralph-README.md`, and
  `rokt-it-ralph-design.md` (the profile table and permissions, the three-layer section, and the
  extension green path), plus the nine profile `config.yaml` files in `ralph-profiles/`.
- **[BOARD]** — the `worker_context` delivered to card `t_8a78fa8b`, carrying the brief, its
  acceptance criteria, constraints and out-of-scope list.
