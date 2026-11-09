---
kind: convention
source: t_f0a3aa2b
profile: ralph-author
date: 2026-11-09
---

# The `ALL` tuple in `evals/test_profile_config.py` mirrors the README profile table, and profile counts in prose are checked by nothing

Two separate profile counts live in the Ralph repository, neither enforced by a
test. A document that cites either from memory gets it wrong. This cost a
review round on the librarian implementation plan.

## What is true

Counted at `main` `f679e098` (2026-11-09):

- **There are 18 profile directories**, not twenty. The authoritative command is
  `git ls-tree -d --name-only <rev>:profiles`. What the test suite counts is
  narrower still: `profiles()` in `evals/test_profile_config.py:205-206` keeps
  only directories holding a `SOUL.md`. At this revision all 18 do, so the two
  agree — but they are not the same question, and a profile directory without a
  `SOUL.md` would make them diverge.
- **`test_there_are_profiles` asserts a floor, not a count**:
  `assertGreaterEqual(len(self.profiles()), 16)` (`:208-209`). Adding a profile
  must not touch it. Writing the new total into that assertion turns a
  deliberately loose check into one that fails on the next profile.
- **The `ALL` tuple (`:81-84`) lists ten of the eighteen, and those ten are
  exactly the ten rows of the `README.md` profile table (`:36-45`), in the same
  order, name for name.** The rule is written down nowhere, but it exists:
  `ALL` mirrors the README table. `ALL` backs
  `test_every_designed_profile_exists` and `test_general_profiles_reach_the_board`.
- **`README.md:47` counts profiles in prose** ("All ten exist… The other five
  are built and unexercised"). Ten table rows, five plus five: internally
  consistent, and nothing checks it. Each new profile has to hand-correct this
  sentence.

## Why it keeps going wrong

Nothing compares the two lists, and nothing checks either prose number, so a
stale count survives until a human reads it. A plan or design is exactly where
a wrong count does damage, because a builder reads the document's number as the
expected value and writes it into an assertion.

## What to do

When adding a profile, treat these as one change set: the `profiles/<name>/`
directory, the `ALL` tuple, and the README table row plus its prose count.
Re-derive counts with `git ls-tree`, never from a previous document. When the
`ALL` tuple and the README row land in different pull requests, the pair is
briefly out of step; that breaks nothing today only because no test compares
them. A one-line comment on `:81` naming the README table as the source, or a
test asserting the two lists match, would end this class of error.
