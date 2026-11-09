---
id: dd-ralph-librarian
title: The librarian runs from a weekly card on `queues`, changes the docs repository through one pull request, and the review loop re-indexes as soon as that pull request lands
status: approved
owner: Zhachory1
card: t_563fd4f6
revision: v1
date: 2026-11-09
implements: prd-ralph-librarian
issue: Zhachory1/ralph#130
supersedes: none
prs:
  - Zhachory1/docs#21   # this design
---

# The librarian runs from a weekly card on `queues`, changes the docs repository through one pull request, and the review loop re-indexes as soon as that pull request lands

> **This repository is public.** No employer, person, space or message content
> appears here. The work instance's docs repository is "the work docs
> repository".

Status: **approved** by the owner (merge of Zhachory1/docs#21, 2026-11-09).
Implements `prd-ralph-librarian` (approved, merged at `Zhachory1/docs#19`).
Nothing is built. This document settles the five unknowns that PRD left to
design time, specifies the profile and its installation, and names the pull
requests. The implementation plan is a separate card and a separate pull
request.

**Citations.** Ralph claims are from `Zhachory1/ralph` `main` at
`f679e098` (2026-11-09), read through `gh api`, line by line. Docs-repository
claims are from `Zhachory1/docs` `main` at `863b48e` (2026-11-09). Live GitHub
state (the `main` ruleset, pull-request merge states, issue states, document
front matter) was queried on 2026-11-09 and is quoted below where it is
load-bearing. I read the code and the API; I ran nothing.

Documents are cited by id, as `AGENTS.md` requires. Paths appear only for code
in `Zhachory1/ralph`, and for docs-repository paths the librarian itself acts
on — those paths are data here, not links.

---

## Review header

### Decisions needed from you: none, both settled

**Owner, 2026-11-09:** accepted both recommendations below. D-A: the librarian reuses `RALPH_MAINTAIN_TOKEN`. D-B: the forced re-index fires on any docs default-branch change. The table keeps the reasoning.

| # | Decision | Why it is yours | Recommendation |
|---|---|---|---|
| **D-A** | **Which credential the librarian pushes with.** `bootstrap/install:228-250` writes `RALPH_MAINTAIN_TOKEN` into exactly three profiles' `.env` (`ralph-maintainer`, `ralph-builder`, `ralph-author`); every other profile gets the read token instead (`:379-402`, skip list at `:384`). The librarian pushes a branch, so it needs one of the two. Acceptance example 6 wants a token that "cannot push to the default branch" — **no GitHub token can be scoped that way on a user-owned repository**; the control is the `main` ruleset, and on `Zhachory1/docs` that ruleset is already active with an empty bypass list (verified below, §4.4). A second token buys a separate audit trail and a separate thing to rotate, and buys no extra protection. | Which credential may push where is a permission boundary, not a file layout. It is the one thing in this design an agent must not decide for you. | **Reuse `RALPH_MAINTAIN_TOKEN`**: add `ralph-librarian` to the two lists in `install`. Keep acceptance 6 as a statement about the ruleset, and say so in the README rather than implying the token enforces it. |
| **D-B** | **The forced re-index fires on any docs-repository change, not only on a librarian merge.** R12 asks for a re-index when a librarian pull request merges. The mechanism that can see a merge cheaply (§3.4) sees *commits*, not authors, so restricting it to librarian pull requests means adding an author check that buys nothing. Taking the wider version also cuts memory-note search latency from up to an hour to one review-loop poll — a change to `dd-ralph-knowledge` D3's hourly cadence, which is why it is yours and not mine. | It relaxes a cadence an approved design chose deliberately, and it costs one `gh api` call per review-loop poll (about 96 a day) plus one incremental re-index per docs commit. | **Take the wider version.** It is less code, and the narrower one leaves the thing memory notes actually need — fast search of what was just written — unfixed. |

### Decisions I made: the five unknowns the PRD left open

Each is settled in §3 with its reasoning. Say so if one is wrong; none of them
is hard to reverse.

| Unknown | Decision | § |
|---|---|---|
| Superseded notes and search | Stay indexed. A banner as the first body line, plus `status: superseded` and `superseded_by:` in front matter. No change to `index.yml`. | 3.1 |
| Are folder indexes indexed? | Yes. They are `README.md`, they hold titles and one-line summaries only, and they are not documents (no id, no status). | 3.2 |
| Proving "duplicate" | Three gates, all required: a search-derived candidate pair of the same `kind`; a shared concrete anchor (path, issue, pull request, card id); and one sentence true of both findings with no "and". No numeric threshold. At most five merges per run. Your merge is the confidence mechanism. | 3.3 |
| Forced re-index | The review loop compares the docs repository's default-branch head against the head it last indexed, and bypasses its own hourly stamp when they differ. | 3.4 |
| What the weekly run is | The review loop creates one card a week on `queues`, assigned to `ralph-librarian`. The dispatcher runs it like any other card. No new scheduler, no Hermes cron job. | 3.5 |

### Council dissent

**None ran.** No council or lens critique was invoked for this document; the
card's finishing sequence is one agent reviewer, then your gate. If the
reviewer disagrees with anything here, that disagreement and its resolution
belong in a v2 of this section, not in a comment thread.

### Scope rating

**M — three build pull requests.** Roughly: the profile and its installation
(~250 lines, mostly prose); the review loop's two blocks and their tests (~120
lines); the documentation and README updates (~80 lines). The plan sizes and
orders them; §5 names them so the plan has something to copy.

---

## 1. What the librarian is, and what it is not

The librarian is the "docs gardener" the README's build order has named since
the beginning and never built (`README.md:104`, step 4). This design builds it.

It is justified as a profile by one permission no other profile has: **it edits
documents and notes that other profiles wrote, anywhere in the docs
repository.** That is the whole of its reason to exist (R1).

The two profiles nearest to it are separated from it by one line each. Both
lines belong in `profiles/librarian/SOUL.md` verbatim, because the cost of
confusing them is a profile doing work that is not its own.

| | auditor | author | librarian |
|---|---|---|---|
| Writes | `areas/agent-system/` only | one new document per pull request | front matter, links, indexes and memory notes, anywhere |
| Judges | the system's behaviour | nothing | **nothing** |
| Creates | a report | a PRD, design or plan | **no new document** |
| Applies its own findings | never ("Propose; never apply", `profiles/auditor/SOUL.md:19`) | n/a | **yes — that is the point** |
| Merges | n/a | never | **never** |

So:

- **Against the auditor.** The auditor judges and proposes; it "cannot edit a
  skill, a prompt, an invariant, or a profile's memory"
  (`profiles/auditor/SOUL.md:9`). The librarian is its mirror image: it edits,
  and it judges nothing. It never says a document is good, bad, stale-on-age,
  or wrong. It only repeats what another document already said — this merged,
  so it is approved; this names that as its replacement, so it is superseded;
  this issue is closed, so this project is finished. Acceptance example 7 stays
  true: nothing here widens the auditor's write scope.
- **Against the author.** The author writes new documents, one per pull request
  (`profiles/author/SOUL.md:13`). The librarian writes no new document. The one
  file it creates that did not exist before is a merged memory note (R7), and
  that is a consolidation of findings other profiles already recorded, not a
  new finding. It must contain nothing that was not in its sources.

Both share the author's hard boundary, and the SOUL repeats it: write only
through a pull request, never push to the default branch, never merge, and run
`git` and `gh` in the terminal tool rather than `execute_code`, because
`execute_code` scrubs `GH_TOKEN` (`profiles/maintainer/config.yaml:6-11`).

### 1.1 What the librarian never touches

- **`Zhachory1/ralph`, or any other code repository.** Its scope is one
  repository: the one `RALPH_DOCS_REPO` names.
- **The body of an approved document**, other than its front matter and its
  links (R13). It does not rewrite prose, fix typos, or restructure a section.
- **Any file, by deleting it** (R13, N1). A `git mv` is the only move it makes,
  and the file stays in the tree and in history.
- **Cards.** It does not create them, comment on other profiles' cards, or
  route work. Everything it has to report goes in its own pull request body.

---

## 2. The run

### 2.1 Trigger

The review loop creates one card a week on `queues`, assigned to
`ralph-librarian` (§3.5). The dispatcher claims it, gives it a sandbox
workspace, and runs it the way it runs every other card, with the usual three
attempts.

### 2.2 The sequence

Order matters: step 6 reads the `status` that step 3 writes, and step 7 reads
the paths that step 6 moves. The run is one pass in this order.

1. **Skip if a run is still open.** `gh pr list --repo "$RALPH_DOCS_REPO"
   --state open` and look for a head branch starting `librarian/`. One found:
   complete the card with "previous run still open, nothing done". This is the
   PRD's assumption that two runs never edit the same notes, made mechanical.
2. **Clone and branch.** Clone the default branch into the card workspace;
   branch `librarian/<YYYY-MM-DD>`. **The checkout is the truth for every
   decision below.** The knowledge index is used only to *find* candidates
   (§3.3), never to decide, because it can be up to an hour behind the
   repository (`bootstrap/run-review-loop:111-120`).
3. **Status (R6).** A document whose `prs:` entries are all merged but whose
   `status` still reads `in review` becomes `approved`. The evidence is
   `gh pr view <n> --json state,mergedAt` per entry, recorded in the pull
   request body. Four rules keep this mechanical:
   - Only the document's own pull requests count — the entries under `prs:`,
     not `supersedes:`.
   - `prs:` is a history, so **the newest entry decides**. An older entry that
     was closed without merging (as `Zhachory1/docs#7` was) does not block a
     document whose latest pull request merged.
   - Any entry still open: leave the document alone.
   - `draft` is never promoted. Only `in review` becomes `approved`.
4. **Memory front matter (R5).** Every note under `memory/` carries `kind`,
   `source`, `profile`, `date` (`dd-ralph-knowledge` D6). Fill what is
   derivable and nothing else:
   - `kind` from the folder (`memory/decisions/` is `decision`,
     `memory/root-causes/` is `root-cause`, `memory/conventions/` is
     `convention`, `memory/reports/` is `report`).
   - `date` and `source` from the file name, which the shared principles fix as
     `<YYYY-MM-DD>-<slug>-<card>.md`.
   - `date` from `git log --diff-filter=A --format=%aI -1 -- <path>` when the
     name carries none.
   - `profile` from the author of that first commit, where it maps to a known
     `ralph-*` profile. Otherwise leave it and list the note.
   - Anything it cannot derive goes in the pull request body under "could not
     complete", with the field name. **It never guesses a value.**
5. **Merge duplicates and mark superseded notes (R7-R9).** §3.3. Capped at five
   merged pairs and five supersede marks per run.
6. **Archive finished projects (R3).** A folder under `projects/` moves to
   `archives/<same name>/` with `git mv` when **both** hold:
   - every document in the folder has `status: approved` or `superseded`
     (after step 3), **and**
   - every distinct issue named in those documents' `issue:` front matter is
     closed (`gh api repos/<owner>/<repo>/issues/<n> --jq .state`).

   A folder whose documents name no issue is **never** archived, and is listed
   in the pull request body as "finished-looking, no issue named". Checked
   against live state on 2026-11-09: `projects/ralph-knowledge/` qualifies
   (three approved documents, `Zhachory1/ralph#121` closed) and is the only one
   that does. `fleet-into-ralph`, `hermes-webui-evaluation`, `ralph-redmine`
   and `ralph-skills` name no issue at all, and `ralph-librarian`'s issue
   (`Zhachory1/ralph#130`) is open.
7. **Links (R4, R10).** In the same commit as the move, rewrite every path
   reference whose target moved in step 5 or step 6. Scope: memory notes and
   pull-request text are the only places that cite paths; documents link by id
   and need nothing (`AGENTS.md`). A reference whose target exists nowhere is
   listed in the body — not guessed at, not deleted.
8. **Indexes (R11).** Regenerate the folder indexes (§3.2).
9. **Open the pull request, or do not.** An empty diff means no pull request
   and no pushed branch; the card completes saying nothing needed changing
   (acceptance 5). Otherwise: one commit per requirement group (status, front
   matter, merges, archive and links, indexes), so the diff can be read one
   concern at a time, then one pull request.

### 2.3 Caps, so one run stays reviewable

The outcome the PRD wants is "the owner reviews one pull request a week". That
holds only if the pull request is readable in one sitting. Three caps, each
stated in the pull request body when it bites:

| Cap | Value | Why |
|---|---|---|
| Merged duplicate pairs | 5 per run | A wrong merge cannot be deleted back out (N1). Five is reviewable in full. |
| Supersede marks | 5 per run | Same reason. |
| Changed lines | stop adding work above ~400 | The engineering standard for a reviewable pull request (`profiles/STANDARDS.md`). The first run on an instance with a backlog hits this. |

When a cap bites, the run stops *between* steps, never inside one: a `git mv`
and its link rewrites are never split across two weeks. Whatever it did not
reach is listed in the body as deferred, and next week's run picks it up,
because the repository still shows the same state.

### 2.4 The pull request

Title: `librarian: <n> changes — <short list>`. The body is the whole report,
because the librarian has no other place to report:

1. **What changed**, grouped by requirement, each line naming the file.
2. **Merges**, one block each: the two source paths, the shared anchor that
   proved them the same subject, and the merged finding sentence (§3.3).
3. **Could not complete**: notes missing a field it could not derive, broken
   links with no target, finished-looking projects naming no issue.
4. **Deferred**: what a cap pushed to next week.
5. **Assumptions**: every inference the run made that the repository did not
   state, one line each — the rule every agent pull request follows
   (`README.md`, "Agents record what they guessed").

---

## 3. The five settled unknowns

### 3.1 A superseded note stays in the index, and says so in its first line

`bootstrap/knowledge-init:18-34` writes one collection over the whole checkout
with `pattern: "**/*.md"` and per-folder contexts. Excluding superseded notes
means changing that pattern, which means a QMD negation pattern I have not
verified, a change to `knowledge-init`, and a re-index on every instance.

**Decision: keep them indexed.** Mark them in the file:

```markdown
---
kind: root-cause
source: <unchanged>
profile: <unchanged>
date: <unchanged>
status: superseded
superseded_by: memory/root-causes/2026-11-16-<slug>-<card>.md
---
# <the original finding sentence, unchanged>

> **Superseded** by `memory/root-causes/2026-11-16-<slug>-<card>.md`
> on 2026-11-16. Kept for history; do not act on it.

<the original body, unchanged>
```

Three reasons, heaviest first:

- **The warning has to travel with the hit.** QMD returns the file's text. The
  banner is the first body line, so it is in the chunk a search returns and in
  what `get` reads. A note moved out of the index cannot warn anybody, because
  the reader never sees it at all — and the trail from a decision to the
  decision that replaced it disappears with it. The index already takes this
  position for finished work: the `/archives` context reads "Finished work;
  check status before trusting it" (`bootstrap/knowledge-init:27`).
- **R8 keeps the file in the tree either way**, so the only question is what
  `qmd` sees, and `qmd` seeing it is what makes the banner worth writing.
- **It needs no change to the knowledge service.** No new coupling between the
  librarian and `index.yml`, which is the spirit of N3.

The cost is real and I will not hide it: a superseded note can still outrank
its replacement for a query. The mitigations are the banner and `merges:` on
the replacement, so a reader is one hop from the right file. If you see this
happen in practice, excluding superseded notes by pattern is a later,
reversible change to `knowledge-init` alone.

**This extends `dd-ralph-knowledge` D6.** That note template has no `status`
field; R8 requires one. Two optional fields join the template, `status` and
`superseded_by`, both absent unless the librarian wrote them. One consequence
belongs in the SOUL: R6's status promotion applies to *documents* with `prs:`,
never to memory notes, so a note marked `superseded` is never "promoted" to
`approved` by a later run.

### 3.2 Folder indexes are `README.md`, and they are indexed

**Decision: one `README.md` per folder, and they stay in the index.**

- **One per folder, mechanically:** a folder that directly holds `.md`
  documents gets an index listing them; a folder that holds only folders gets
  an index listing those. Every folder gets exactly one. No judgement call, and
  both readings of R11 ("each folder under `projects/`…") are covered.
- **`README.md`, not `index.md`**, because GitHub renders it when you open the
  folder. The index is then useful to a person browsing the repository and not
  only to search.
- **An index is not a document.** It carries `kind: index` and
  `updated: <date>` and nothing else — no `id`, no `status`, no `owner`. So
  nothing links to it by id, R6 never promotes it, and it never looks like
  something to trust.
- **Content is titles and links only:** the folder's one-sentence purpose, then
  one line per entry — `- [<title>](<path>) — <status>, <one line>`. The one
  line comes from the document's own title, never from reading its body.

**Why they stay indexed.** The noise worry is real but small: at today's shape
that is about eight extra files among a few dozen, each holding only titles
that already exist in the documents it lists. The worst case is one extra hit
that points at the right document — one hop, with the title visible. The win is
the opposite case: "what work exists on X" becomes one query instead of a
directory walk, which is the question a folder index exists to answer.
Excluding them needs the same unverified pattern negation as §3.1 for a smaller
benefit.

Keeping them cheap is what keeps this true, so the format above is a rule and
not a suggestion. An index that grows prose starts competing with the documents
it lists; that is the failure mode to watch for.

### 3.3 Three gates prove a duplicate, and your merge is the confidence

A wrong merge cannot be deleted back out (N1). So the question is not "what
score is high enough" but "what evidence does a reader of the pull request need
in order to reject it". A similarity number is not that evidence: QMD's score
is not calibrated for this question, and a threshold on it would look like
proof while being a guess with a decimal point.

**Decision: three gates, all required, and no numeric threshold.**

1. **Candidate (mechanical).** For each memory note, run `knowledge.query` with
   the note's `# <finding>` heading as a `vec` sub-query, restricted to
   `memory/`, and take the top five hits. A pair is a candidate only when both
   notes have the same `kind`. The index may be stale here, because this stage
   only proposes; step 2 of §2.2 says the checkout decides.
2. **Anchor (mechanical, and this gate does the work).** The pair must share at
   least one concrete anchor named in both notes: the same file path, the same
   issue or pull-request number, the same card id, or the same named script,
   profile or configuration key. **No shared anchor, no merge** — no
   exceptions, and no model judgement overrides this gate. It is what turns
   "these feel similar" into something a reader can check in the pull request
   body in five seconds.
3. **One sentence (judgement, conservative).** Read both notes in full from the
   checkout and try to write one sentence true of both findings. **If the
   sentence needs an "and", they are two findings and they stay two files.**
   This is the test the note template already applies: one finding per note,
   stated as one sentence.

Then: at most five merges per run (§2.3); every merge itemised in the pull
request body with both source paths, the anchor and the merged sentence; and
nothing takes effect until you merge. **That review is the confidence
mechanism.** It is also why the gates are conservative rather than clever. A
librarian that merges only the obvious pairs leaves work for next week; a
librarian that merges the plausible ones spends your review budget on undoing
things.

**The merged note:**

```markdown
---
kind: <same as the sources>
source: <the librarian pull request URL>
profile: ralph-librarian
date: <the merge date>
merges:
  - memory/<kind>/<source-a>.md
  - memory/<kind>/<source-b>.md
---
# <the one sentence true of both>

<every source's finding, kept — nothing dropped, nothing added>

## Sources
- `<source-a>` — <its original `source:` and `date:`>
- `<source-b>` — <its original `source:` and `date:`>
```

Each source is then marked as in §3.1, pointing at this file.

**R9, supersede-on-replacement, uses gates 2 and 3 with the direction set by
`date:`:** the newer note wins, the older is marked. Equal dates, or a pair
where neither plainly replaces the other, means no mark and a line in the pull
request body. R9 and N6 are explicit that age alone is never a reason, so the
run never marks a note it cannot name a replacement for.

### 3.4 The review loop forces the re-index, by comparing heads

The two options the card names resolve against the live wiring like this:

| Option | Verdict |
|---|---|
| The notifier calls `refresh-knowledge` on the merge it sees | **No.** `notifier/approvals.py` looks only at cards it already polls — blocked and triage cards on `RALPH_BOARD` (`:176`) — and its merge check (`:268`) runs inside gate handling (`:375`). The librarian's card is on `queues`, which the notifier does not poll, by design (`bootstrap/install:273-290`). It would also need `RALPH_DOCS_REPO` added to its environment, which `compose/docker-compose.yml:213-228` does not set. This couples the re-index to a card shape the librarian deliberately does not have. |
| Something clears the review loop's `.knowledge-stamp` | **Yes in effect — but the review loop has to do it itself.** The stamp lives at `$WORK/.knowledge-stamp` under `/workspaces` (`run-review-loop:17,113`), and the only containers that mount `/workspaces` are `hermes` and `review-loop` (`compose/docker-compose.yml:165,275`). The notifier mounts `notifier-state` alone (`:232-236`). No outside actor can clear it. |

**Decision: the review loop decides for itself whether its stamp applies**, by
comparing the docs repository's default-branch head against the head it last
indexed. It already holds everything that needs: `RALPH_DOCS_REPO`
(`compose/docker-compose.yml:268`), `GH_TOKEN` and `gh` (`:261`), the stamp,
and the `docker exec` into the knowledge container (`run-review-loop:118`).

The shape, inside the existing block at `run-review-loop:111-120`:

- Read the remote head: `gh api repos/$RALPH_DOCS_REPO/commits/HEAD --jq .sha`.
- Compare it with `$WORK/.knowledge-head`, a file this block writes.
- Refresh when the hourly stamp has expired **or** the heads differ.
- Write `.knowledge-head` only after `refresh-knowledge` exits zero, so a
  failed refresh retries on the next poll instead of recording a lie.

Why not read `status.json`, which already records the indexed head
(`bootstrap/refresh-knowledge:29-31`)? It lives inside the knowledge volume, so
reading it costs a second `docker exec` every poll. A file in `$WORK`, beside
the two stamps already there, costs nothing and follows the pattern the loop
uses twice already (`:105`, `:113`).

**What this gives you, stated honestly:** a re-index within **one review-loop
poll** of the merge — 15 minutes by default (`RALPH_REVIEW_POLL_SECONDS`), not
"instantly" — and `run-review-loop --once` forces it immediately. That is the
PRD's "at once, instead of waiting for the hourly stamp". Concurrency needs
nothing new: `refresh-knowledge` takes a `flock` before it does anything
(`:20-21`), which is owner decision 3.

Acceptance examples 1 and 2 check the result after this refresh: search "QMD
service per instance" and get the archived path; search the merged finding and
get the merged note, with each source showing its banner.

### 3.5 The weekly run is a card the review loop creates

| Option | Verdict |
|---|---|
| A Hermes cron job running the librarian directly | **No.** It works — the daily digest is a Hermes cron job (`docs/workflows.md:114`) — but a cron session has no card: no dispatcher retry, no workspace lifecycle, no comment thread, and a failure visible only in a container log. It also cuts against the shared principle that work moves through cards. |
| A Hermes cron job that creates a card | **No.** Same outcome as below, with a second mechanism to install and an unverified assumption about what a cron `--script` can reach. `bootstrap/install-overlay-jobs` installs such scripts today, but only from a private overlay. |
| **The review loop creates the card** | **Yes.** |

**Decision: a weekly block in `run-review-loop`, beside the two periodic blocks
already there.** It is the smallest complete slice, and every part of it is
proven in that file already:

- A stamp for periodic work: `$WORK/.librarian-stamp`, `find -mmin -10080`
  (seven days) — the shape of `:105-109` and `:113-120`.
- Board commands through the hermes container:
  `docker exec "$RALPH_HERMES_CONTAINER" hermes kanban --board queues …` — the
  shape of `:62-64`.
- The guard: no card while an open pull request has a `librarian/` head, so two
  runs never overlap. This is the PRD's assumption made mechanical, checked
  here as well as inside the run (§2.2 step 1), so a card is not even created
  when the previous one is unmerged.
- `RALPH_DOCS_REPO` unset means no card. An instance with no docs repository
  gets no librarian, with no extra switch to set.

**What reads `RALPH_DOCS_REPO`:** two things, both wired already. The review
loop reads it from its own environment (`compose/docker-compose.yml:268`) to
build the card and run the guard, and writes it into the card body. The
librarian's sandbox reads it from the profile's `.env`, which `install` writes
for every profile (`:409-425`) and forwards into the sandbox (`:106`). So "per
instance, reading only that instance's docs repository" needs no new
configuration: an instance physically cannot point its librarian at another
instance's repository.

**Cadence, plainly:** every seven days *since the last run*, drifting by up to
one poll interval. Not a fixed weekday. The review loop has no calendar, and
the two periodic jobs already in it express their schedule the same way.

**Who tells you the pull request exists:** GitHub. The notifier polls
`RALPH_BOARD` only, so a card on `queues` never pages you
(`bootstrap/install:273-290`), which is the property that board exists to
protect. The librarian's output is a pull request in your own repository, and
GitHub already notifies you of those.

---

## 4. Installation (R17)

### 4.1 `profiles/librarian/SOUL.md`

Built like every profile's: `install` prepends `PRINCIPLES.md` and
`STANDARDS.md` (`bootstrap/install:313`). Its own text carries, in this order:

1. The job in one line, and the permission that justifies the profile (R1).
2. The two boundary paragraphs of §1 — against the auditor, against the author
   — close enough to the wording here that neither can be mistaken for the
   other.
3. The run sequence of §2.2, as the numbered order it is, with the rule that
   the checkout decides and the index only proposes.
4. The duplicate gates of §3.3, with "no shared anchor, no merge" stated as
   absolute.
5. The caps of §2.3 and the pull-request body shape of §2.4.
6. Explicit non-goals mirroring N1-N6: never delete; never judge quality or
   system health; no second store; no change to the approval gates; no expiry
   by age; no decision about the repository's ruleset. Plus the three in §1.1.
7. The finishing rule: open one pull request, complete the card with its link,
   never merge, never block waiting for approval. The merge is the approval and
   it arrives whenever it arrives.

The operating rules live in the SOUL rather than in a new skill, for now.
`SOUL.md` is assembled by `install` from a read-only mount, so an agent cannot
rewrite it, and adding a skill lane for one profile is more machinery than this
needs. If the procedure outgrows what a SOUL should hold, it becomes a skill
under `skills/write/` later.

### 4.2 `profiles/librarian/config.yaml`

| Setting | Value | Why |
|---|---|---|
| `model` | `anthropic` / `claude-sonnet-5` | The same call as `maintainer` and `auditor`. The work is mechanical with a few careful judgements, every one of which a human reviews. |
| `platform_toolsets.cli` | `terminal`, `file`, `kanban` | `terminal` because `git` and `gh` need `GH_TOKEN`, which `code_execution` scrubs (`profiles/maintainer/config.yaml:6-11`). `kanban` because it runs from a card. **No `code_execution`**: it has nothing to compute, and leaving it out removes a tool that would silently fail to push. |
| `approvals` | `approve` for both modes | Every card-dispatched profile carries this; the default refuses the tools a card worker needs, and no human is present to answer (`profiles/author/config.yaml`). |
| `memory`, `auxiliary` | disabled | As every Ralph profile. |
| `skills.external_dirs` | `/ralph/skills/voice`, `/ralph/skills/write` | Voice because people read its pull requests; write for document conventions. Not `code`, `debug`, `ui`, `plan` or `meta`. |
| `agent.max_turns` | 120 | It reads every file in the repository and opens one pull request. The author's 80 is sized for one document; the builder's 120 is the closer shape. |
| `agent.run_budget_seconds` | 3600 | Same reasoning. |

**Not added to `repo_cache_profile`** (`bootstrap/install:124-129`). The cache
spans every repository the instance has touched, across every owner
(`:118-123`); the librarian needs one small repository, and a plain clone
covers it.

### 4.3 `knowledge_profile`: yes

Add `librarian` to the case at `bootstrap/install:133-138`, so it gets the
shared MCP block (`:142-152`). It needs `query` for §3.3's candidate stage, and
"Search before you plan or write" applies to it like every other profile.

With one rule the SOUL states and §2.2 step 2 repeats: **the index proposes,
the checkout decides.** The index can be an hour behind, and acting on stale
text when the file is in the working tree is the mistake this profile is
uniquely placed to make.

### 4.4 Credentials, and what acceptance example 6 actually rests on

Decision **D-A** is yours. The recommended shape: add `ralph-librarian` to the
push-token loop (`bootstrap/install:235`) and to the read-token skip list
(`:384`).

What that gives, verified on 2026-11-09 against the live API: `Zhachory1/docs`
ruleset 26659401 ("Main") is `active`, targets `~DEFAULT_BRANCH`, has an
**empty `bypass_actors` list**, and enforces `pull_request`,
`non_fast_forward` and `required_status_checks` (with an empty check list and
`required_approving_review_count: 0`). So on this instance a direct push to
`main` fails for **any** token, the owner's included. Acceptance example 6
holds — **because of the ruleset, not because of the token** — and the README
should say it that way, in the spirit of its existing line: treat "agents don't
merge" as a repository setting you own.

An instance whose docs repository has no such ruleset does not get that
property from this design. The honest statement for the profile table and the
operations documentation is: *protect the docs repository's default branch
before installing the librarian.*

### 4.5 Board access: yes, `queues`

It runs from a card, so it needs the `kanban` toolset. The card is on `queues`
because the review loop creates it, and `queues` is "producer-created work …
No human cards here" (`bootstrap/install:285`). It creates no cards itself.

### 4.6 The README row

Inserted after `auditor` in the table at `README.md:34-45`, in the same
one-line style:

```
| librarian | Tidy the docs repo through one PR a week | Delete, merge, or judge |
```

Three further README edits, each small and each needed for the README to stay
true:

- `README.md:47` says "All ten exist". It becomes eleven, and the librarian
  joins the unexercised list until its first run merges.
- `README.md:104`, build order step 4, names "docs gardener". Name the
  librarian there and link the issue, so the roadmap and the profile are one
  thing rather than two.
- One line, under the table or in step 4, saying the default-branch protection
  of §4.4 is a prerequisite.

### 4.7 One operational step, for you only

After the profile and `install` changes merge and the image is pulled, `init`
has to run again on every instance before the profile exists there. No build
pull request and no card can do that, and the plan should not create one for
it. It belongs in this feature's launch note and in `docs/operations.md`.

---

## 5. What changes in `Zhachory1/ralph`

The plan sizes and orders these; the grouping is one concern each.

| # | Concern | Files | Rough size |
|---|---|---|---|
| **1** | The profile and its installation | `profiles/librarian/SOUL.md` (new), `profiles/librarian/config.yaml` (new), `bootstrap/install` (`knowledge_profile`, the two token lists), `evals/test_profile_config.py` | ~250 lines, mostly prose |
| **2** | The weekly card and the forced re-index | `bootstrap/run-review-loop` (two blocks), `evals/test_run_review_loop.py` | ~120 lines |
| **3** | Documentation | `README.md` (table row, the count, build order step 4, the protection prerequisite), `docs/workflows.md` (a "Librarian" section beside "Knowledge"), `docs/operations.md` (the `init` step, how to force a run) | ~80 lines |

Pull request 2 depends on nothing in 1: a card created for a profile that does
not exist yet simply sits in `ready`, which is visible and harmless. Shipping 1
first means the first card that appears can actually run, so the order is 1,
then 2, then 3. Nothing here parallelises usefully — the whole feature is three
small pull requests.

## 6. How the build proves itself

Deterministic tests, in the harness that exists.
`evals/test_run_review_loop.py:19-40` stubs `docker` on `PATH` and runs the
loop with `--once`; these follow that pattern and need no network and no model.

**`evals/test_run_review_loop.py`**

- A fresh `.librarian-stamp` (younger than seven days) creates no card.
- An absent or week-old stamp creates exactly one card, on `queues`, assigned
  to `ralph-librarian`.
- An open pull request with a `librarian/` head creates no card, whatever the
  stamp says.
- `RALPH_DOCS_REPO` unset creates no card.
- `.knowledge-head` differing from the stubbed remote head runs
  `refresh-knowledge` even when the hourly stamp is fresh.
- Matching heads and a fresh stamp run nothing.
- A `refresh-knowledge` that exits non-zero leaves `.knowledge-head` unchanged,
  so the next poll retries.

**`evals/test_profile_config.py`**

- `ralph-librarian` gets the knowledge MCP block.
- `ralph-librarian` gets the push token and not the read token (if D-A takes
  the recommendation).
- `ralph-librarian` has `terminal` and not `code_execution`.
- `ralph-librarian` does not get the `/repo-cache` mount.

**Not covered by a test, and named as such:** the run itself — the duplicate
gates, the archive rule, the index format. Those are a model's judgement over
real files, and the check on them is your review of the pull request, which is
the design (§3.3). The first run on the personal instance is the acceptance
test, and acceptance examples 1 to 5 are exactly what to look for in it.

---

## 7. Alternatives considered

| Option | Why not |
|---|---|
| Move superseded notes to a folder the index pattern excludes | The warning stops travelling with the hit, the replacement trail disappears from search, and it needs a QMD negation pattern I have not verified plus a `knowledge-init` change (§3.1). |
| A similarity threshold for duplicates | QMD's score is not calibrated for "same finding". A number would read as proof while being a guess (§3.3). |
| A Hermes cron job on the librarian profile | No card, so no retry, no workspace lifecycle, and a failure visible only in a log (§3.5). |
| The notifier triggering the re-index on the merge it sees | It polls only `RALPH_BOARD`, and the librarian's card is on `queues`; it also mounts no `/workspaces` and has no `RALPH_DOCS_REPO` (§3.4). |
| Lower the refresh stamp from an hour to the poll interval | Simpler, but it overrides an approved design's cadence on a cost claim I cannot verify. The head comparison keeps the hourly floor and bypasses it only when the repository actually moved (§3.4). |
| A second, narrower token for the librarian | GitHub cannot scope a user-repository token to "not the default branch", so it adds a credential to rotate and no protection (§4.4, D-A). |
| Let the librarian delete anything | N1, and the reason behind it: nothing it does is reversible by itself. |

## 8. Non-goals

Carried forward from the PRD, and binding on the build:

- **N1.** No deletion, ever. `git mv` and edits only.
- **N2.** No change to the PRD, design and plan gates, or to who approves them.
- **N3.** No new database, service or second store. The docs repository and the
  existing QMD index are the whole system.
- **N4.** No judging of a document's quality or of the system's health. That
  stays with the reviewer and the auditor.
- **N5.** Not a replacement for the direct memory write path, and no decision
  on the ruleset.
- **N6.** No expiry by age. A note is marked only when something replaces it.

Added by this design:

- **No change to `index.yml` or `bootstrap/knowledge-init`.** §3.1 and §3.2
  were both decided so the knowledge service needs no change at all.
- **No new label, board, container or environment variable.** Everything the
  trigger and the re-index need exists already.
- **No touching any repository but the one `RALPH_DOCS_REPO` names.**

## 9. Assumptions

Each is an inference the PRD and the card did not state. Any of them being
wrong changes something in this document.

- `hermes kanban … create` can be driven from `run-review-loop` through
  `docker exec` into the hermes container, with an assignee and a body. The
  loop already runs `hermes kanban … list` that way (`:62-64`); create is the
  same CLI, and I did not run it.
- QMD indexes a note's body text, so a banner as the first body line comes back
  with a search hit. Front-matter indexing is **not** assumed, which is why the
  banner is in the body and not only in `status:`.
- `qmd update` and `qmd embed` over an unchanged checkout are cheap enough that
  one extra incremental refresh per docs commit is not a load concern.
  `refresh-knowledge` embeds only what changed (`dd-ralph-knowledge` D3).
- Memory notes and pull-request text are the only places in the docs repository
  that cite paths; documents link by id (`AGENTS.md`). If a document does cite
  a path, step 7 rewrites it the same way and nothing breaks.
- The memory-note file name is `<YYYY-MM-DD>-<slug>-<card>.md`, per the shared
  principles, so `date` and `source` are derivable from it for notes written
  under that rule. Notes written before it are the ones step 4 reports.
- A docs repository stays small enough — hundreds to low thousands of files —
  for one run to read in full. The PRD assumes this; the caps in §2.3 keep the
  *pull request* bounded regardless.
- The dispatcher runs a `queues` card assigned to a profile that exists with no
  extra wiring, the way it runs review and maintenance cards today.
- `strict_required_status_checks_policy: true` with an empty required-check
  list does not block merging a branch that has fallen behind `main`. If it
  does, a librarian pull request left open while a memory note lands needs an
  "Update branch" click before it merges. I did not test this.

## 10. Risks

| Risk | Mitigation |
|---|---|
| **A wrong merge, unreviewable after the fact.** The librarian cannot delete, so the undo is a human's `git revert`. | Three gates, a five-merge cap, every merge itemised in the pull request body, and nothing takes effect before your merge (§3.3). |
| **The first run is huge** — a long backlog in one pull request nobody can read. | The ~400-line cap stops the run between steps and defers the rest (§2.3). On the personal instance today the first run is small: one archive, four status fixes, the indexes. |
| **A superseded note outranks its replacement.** | The banner and `merges:` put the reader one hop away. Pattern exclusion stays available as a reversible later change (§3.1). |
| **The run silently does nothing for weeks** — a broken guard, a stamp that never ages, a profile never installed. | The card lands on the board whether or not the run succeeds, with the dispatcher's three attempts; `knowledge-status` and the board show the state. Worth one line in `docs/operations.md` on how to force a run. |
| **Two librarian runs overlap** and edit the same notes. | Guarded twice: the review loop creates no card while a `librarian/` pull request is open, and the run checks again before it starts (§3.5, §2.2). |

## 11. Observations, out of scope

- The docs repository's root `README.md` folder table lists `projects/`,
  `areas/`, `resources/` and `archives/`, but not `memory/`, which R11 asks the
  librarian to index. Adding the row is a one-line change for whoever next
  edits that file; the librarian does not edit the root README under this
  design.
- `areas/` and `resources/` hold only `.gitkeep` today, so their indexes stay
  empty until something lands in them.
- Four documents currently read `status: in review` with every pull request
  merged, and none of them names an issue. So the first run fixes four status
  lines and archives nothing but `ralph-knowledge` — acceptance examples 1 and
  4 landing in the same pull request.
- `projects/ralph-knowledge/` uses `design.md` and `plan.md` while the root
  `README.md` names `dd.md` as the shape of a project folder. Nothing here
  depends on file names — the librarian reads front matter — but it is a
  convention drift worth a decision someday.
