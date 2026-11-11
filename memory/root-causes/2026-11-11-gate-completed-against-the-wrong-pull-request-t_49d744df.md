---
kind: root-cause
source: t_49d744df
profile: ralph-author
date: 2026-11-11
---

# An approval gate card can complete against the merge of a different pull request, releasing the next card early

The `Approve design: queues board split` gate (`t_27ba9139`) completed with the
summary "Approved in GitHub: https://github.com/Zhachory1/docs/pull/27 merged".
Pull request 27 was the **PRD** for that feature, merged about forty minutes
earlier. The design pull request the gate was waiting on, `Zhachory1/docs#30`,
was still open at the time and is still open now.

The effect is not loud. The gate completed, its child plan card promoted to
ready, and a worker started planning from a design the owner had not approved.
Nothing failed; the work simply rested on an unapproved document. The plan card
(`t_49d744df`) worked around it by planning against #30's head and saying so in
the document, in its pull request body, and in a decision row.

What to check when this shows up again: a gate's completion summary names a
pull request number, and nothing compares that number to the pull request the
gate's parent card actually opened. Two gates for the same feature, minutes
apart, are two chances to match the wrong one. The parent card's handoff
carries the right URL, so the comparison is available where the gate is
resolved.

Do not treat a released card as proof that its gate's document is merged. Read
the document's pull request state before grounding work on it, which costs one
`gh pr view`.
