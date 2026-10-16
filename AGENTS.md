# Working in this repository

This is a documents repository. Nothing here is built, deployed, or imported by anything: it holds the designs, decisions and analyses that explain why the code looks the way it does.

## What belongs here

PRDs, design documents, decision records, analyses, research notes, runbooks, launch plans, postmortems. Markdown only.

Organised by [PARA](https://fortelabs.com/blog/para/): `projects/` for work with an end, `areas/` for ongoing responsibilities, `resources/` for reference, `archives/` for anything finished. One folder per feature under `projects/`, holding everything about that feature.

## What does not belong here

- **Code.** A design document describes an implementation; it does not contain one beyond the snippets an argument needs.
- **Secrets, credentials, tokens, customer data, or anything personally identifying.** Analyses carry aggregates; the underlying data stays in the platform that holds it, referenced by path.
- **A second copy of something.** Link to the code repository, the pull request, or the dashboard. A copy goes stale silently.

## Writing a design document

State the answer in the title, not the topic. Open with the decisions the reader has to make, the scope, and the pull requests you propose, so the document can be triaged in two minutes.

Cite what you verified: real paths, real line numbers, and the commit they are from. A reader must be able to check any claim you make about existing code.

Carry two sections without exception:

- **Non-goals.** What this deliberately does not do. A design without them grows in review.
- **Assumptions.** Every inference you made that the brief did not state, one line each. "None" is valid and should be rare. An unflagged wrong assumption is the thing most likely to reach production looking finished.

## Frontmatter and status

Every document carries an id, a status (`draft`, `in review`, `approved`, `superseded`), an owner, the Kanban card it came from, and related pull requests. Link between documents by id, never by path, so archiving never breaks a link.

Check the status before trusting a document: a superseded design still reads as confident. A replaced design is marked and linked to its successor, never deleted.

## If you are an agent working here

- The request is the card brief you were given. Stop when its acceptance criteria are met.
- Write the document and open a pull request. Never merge it: the approval is a human's, and merging is how they give it.
- Out-of-scope problems you notice go under observations, one line each. Report them; do not fix them.
- If you cannot finish without broader changes, stop and say what is needed and why.
