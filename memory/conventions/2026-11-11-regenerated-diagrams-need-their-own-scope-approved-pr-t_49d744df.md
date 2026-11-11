---
kind: convention
source: t_49d744df
profile: ralph-author
date: 2026-11-11
---

# A pull request that regenerates a Graphviz diagram cannot meet the 400-line scope cap, so plan it as its own labelled pull request

`bootstrap/scope-check` fails a pull request above 400 added-plus-removed
lines. The committed `.svg` files beside each `.dot` source in `docs/diagrams/`
are 575 and 895 lines, and every node and edge in them carries absolute
coordinates. Adding one node re-lays out the whole graph, so the regenerated
file rewrites most of its own lines: changing two diagrams is thousands of
changed lines, none of them written by a person.

So a plan that includes a diagram change should put it in its own last pull
request, with the hand-written `.dot` edits as the reviewable part, and say in
the plan that the owner's `scope-approved` label is what lets it through. That
label is the documented human override and is not an agent's to apply to its
own pull request.

Two related facts. The `.png` files cost nothing: `scope-check` counts a binary
file's `-` in `git diff --numstat` as zero, despite a comment immediately above
that line saying the opposite. And leaving the diagram stale instead is not an
option a design will usually accept — a wrong architecture picture is worse
than no picture, because it is read as current.
