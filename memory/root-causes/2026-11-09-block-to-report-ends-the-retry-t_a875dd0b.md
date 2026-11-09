---
kind: root-cause
source: t_a875dd0b
profile: ralph-author
date: 2026-11-09
---

# On a queue polled by status, blocking a card to report a failure is what ends its retry — "it retries next scan" and "it blocks and alerts" cannot both be true of the same failure.

## What happened

An implementation plan wired a new step into a sender that scans once a minute
for cards in one status and acts on them. The plan said, in one pull request's
description, that a non-zero exit from the new step holds the outgoing message
and **retries on the next scan** — true of the shipped loop, which re-selects
the same card as long as nothing has moved it. Three paragraphs later the same
pull request routed that same failure through a helper that **blocks the card
and alerts**, which is what the design asks for so a quiet failure becomes
visible.

A reviewer reading for reliability caught it. The selection is by status: the
scan lists one status, and the handler refuses a card in any other. A block
changes the status. So the first failure would have removed the card from the
set the retry draws from, and the retry the plan promised could never run. One
slow call to a third-party service would have stranded a message that the design
expected to go out a minute later, with no automatic recovery and nothing in the
runbook saying what a human should do.

## Root cause

Two mechanisms were designed independently and composed without asking what they
do to each other. "Retry by re-selection" and "report by changing state" are
each correct alone. Together, on a queue whose selection *is* the state, the
reporting mechanism silently consumes the retry mechanism.

It is invisible in a plan because the two claims sit in different bullets and
each cites a real source line. Nothing contradicts within a sentence. The
contradiction only appears when you ask which of the two happens on failure
number one.

## The convention that replaces it

When a worker both retries a failure and escalates it, write the **budget**
explicitly: how many attempts before escalation, where the count is kept, and
what clears it. Then say, in the same place, that escalation is deliberately the
**end** of the automatic path, and name the recovery step a human performs. An
escalation with no named recovery is a dead end whoever hits it has to reverse
engineer at the worst moment.

Keep the counter with the party that decides to hold the work, not with the
component that failed: the component must stay callable by hand without spending
someone else's budget.

## How to check before writing the claim

For every failure path, ask one question: *what selects this work next time, and
does anything on this path change that selector?* If the queue selects by status
and the failure path writes a status, the two are the same mechanism and only
one of them can win.

## Links

- The plan the finding was raised against: `Zhachory1/docs#24`
- The design it implements: `Zhachory1/docs#15`, amended by `Zhachory1/docs#17`
