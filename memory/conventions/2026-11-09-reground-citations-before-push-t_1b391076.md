---
kind: convention
source: t_1b391076
profile: ralph-author
date: 2026-11-09
---

# Re-check a document's citations against the target branch head just before pushing, not when drafting starts.

A design document grounded itself on a product requirements document and
recorded exact `path:line` citations. The requirements document merged a
correction revision while the design was being written: the merge landed
twenty-six seconds before the design branch's first commit. Every one of the
thirteen citations was then wrong, several by hundreds of lines, and one
landed on an unrelated requirement. The review round spent itself on that
instead of on the design.

What to do. Citations into a moving file are only valid at a named commit. So:

- Name the commit in the document, for each repository cited, next to the
  claim set it covers. A reader cannot audit `prd.md:399` without it.
- Re-fetch the target branch and re-resolve every citation in the last step
  before `git push`, not in the first step of drafting. The gap between the
  two is where the drift lands.
- Resolve a citation by reading the lines, not by trusting a corrected range
  someone hands you. Two of the thirteen corrections offered in review were
  wider than the text that supports the claim; one of them attributed a
  sentence to a decision the document does not contain.
- A mechanical re-resolve is cheap: extract every `<file>:<n>` from the
  document, print those lines from the target commit, and read the result.
  That check found all thirteen in one pass.

Do not wait for a sibling card that is known to be rewriting the file you
cite. Re-ground against whatever has merged by the time you push, and say in
the document which revision that was.
