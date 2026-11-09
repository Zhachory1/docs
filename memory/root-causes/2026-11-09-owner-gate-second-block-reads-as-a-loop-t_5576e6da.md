---
kind: root-cause
source: t_5576e6da
profile: ralph-author
date: 2026-11-09
---

# A card that blocks twice for the same human approval is counted as a block loop and sent to triage — the kernel counts repeat blocks per card, not per reason, so a document that goes round the owner twice looks like an agent stuck in a circle.

## What happened

An implementation-plan card produced a pull request, took one review round,
fixed what the review found, and blocked for the owner with a reason naming the
open decisions. The owner answered them and sent back one further correction.
The card reopened, made the correction in a single push, and blocked for the
owner a second time — this time with nothing open at all, only "please merge".

That second block did not land in `blocked`. It landed in **triage**, with the
event `block_loop_detected` (recurrences 2, limit 2).

Triage means "a specifier should flesh this card out before work starts". The
card was the opposite of under-specified: it was finished work, reviewed, with
a pull request ready to merge, waiting on one action from one human. Anyone
reading the column would have drawn exactly the wrong conclusion about what the
card needed, and the one person who could move it was not told again.

## Root cause

The loop detector counts how many times a card has been blocked and compares
that count against a limit. It does not compare the reasons. That is the right
rule for the case it was built for: an agent that blocks, gets unblocked, and
blocks again on the same wall is looping, and a human should look at the card
rather than let it bounce.

But the approval workflow blocks for a *different* reason each time, and both
blocks are correct. Round one asks a question. The human answers. Round two
says the answer is applied and the artifact is ready. Two blocks is the normal,
healthy length of a document review with one round of owner feedback — not a
loop, a conversation. With a limit of two, the second and cleanest block of a
healthy card is the one that trips the alarm.

The deeper confusion is that two unrelated things share one mechanism. "I am
stuck and cannot proceed" and "this is done and needs your approval" are both
expressed as a block, so any counter over blocks mixes agent failure with human
gating.

## What to do about it

**As the author of such a card:** expect it, and say so on the card. When the
second block lands in triage, post a comment immediately stating that the
status is mechanical, that nothing is waiting on a specifier, and naming the
single human action outstanding with its link. A triage card with no
explanation gets re-specified by whoever picks it up, and the work is redone.

**Prefer a review handoff to a block where the lifecycle allows it.** A review
transition is explicitly not a blocker and never counts toward loop detection,
so a card can cycle through review as many times as the work honestly needs.
Reserve blocking for the final human gate, so a card spends at most one block
per approval.

**When designing a gate into a card graph:** a gate is a card whose whole
purpose is to wait for a human, so count its blocks separately from a worker's,
or do not count them at all. A counter that cannot tell "stuck" from "awaiting
approval" will always mis-fire on the second lap, and it mis-fires on the
healthiest cards — the ones that got their feedback and acted on it.

## How to check before relying on a block

Ask what the status will mean to the *next reader*, not to the kernel. If a
card can arrive in a column whose name describes a problem it does not have,
that column is carrying two meanings, and the card needs a comment saying which
one applies.

## Related, not verified here

A gate card created with the assignee string `"none"` is reported to be
rejected by the approval-gate check and stranded, needing completion by hand
(seen on `t_094ab531`, reported again on `t_082277be`). Same theme — gate cards
travelling the worker path — but a separate cause, recorded here only so that
the two are found together.

## Links

- The plan this happened on: `Zhachory1/docs#22`
- The design it implements: `Zhachory1/docs#14`, amended by `Zhachory1/docs#17`
