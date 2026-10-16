# Docs

Durable knowledge: the designs, decisions and analyses that outlive the work that produced them. Organised with [PARA](https://fortelabs.com/blog/para/) — by how actionable something is, not by what kind of document it is.

| Folder | Holds |
| --- | --- |
| `projects/` | Short-term efforts with a goal and an end. One folder per feature: PRD, design doc, decisions, analyses, launch plan, postmortem |
| `areas/` | Ongoing responsibilities with no end date: services owned, runbooks, conventions |
| `resources/` | Reference material: standards, templates, research notes |
| `archives/` | Finished or paused items from the other three, moved whole |

Everything for one feature lives in one folder, because that is how it gets worked on.

## Why documents live here and not beside the code

A design document is reviewed by the person who asked for the work. A code repository's review rules are built for code: CODEOWNERS, required approvals from a team, checks that have nothing to say about prose. Putting a design doc through them means it cannot be approved by the one person whose approval the design actually needs.

So designs are proposed, reviewed and approved here, and the code repository receives the implementation once the design is settled. The design doc links to the code repository; the pull request there links back.

## Shape of a project folder

```
projects/<feature>/
  prd.md         the problem, the outcome, what success looks like
  dd.md          the design: the question, the answer, non-goals, assumptions
  decisions.md   decisions taken during the work
  launch.md      flags, stages, rollback, ownership
  postmortem.md  after the fact, if there is anything to say
```

A decision that outlives its feature moves up to the owning area.

## Frontmatter

Every document carries: an id, a status (`draft`, `in review`, `approved`, `superseded`), an owner, the Kanban card it came from, and any pull requests it relates to. Links between documents use ids rather than paths, so archiving a project never breaks one.

A replaced design is marked `superseded` and linked to its successor. It is never deleted.
