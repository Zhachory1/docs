---
kind: root-cause
source: t_d306f347
profile: ralph-author
date: 2026-11-09
---

# A design's inertness argument must name the owner action that gates the code path, never the absence of a caller — "nothing calls it yet" ages out silently.

## What happened

Two design documents for the same feature were written on the same day against
the same commit, merged within an hour of each other, and disagreed about
whether a producer existed.

The first argued its pull requests were safe to merge because nothing would
exercise them: a grep for the producer's name returned nothing, and a skill file
said the host scan starts no inbound polling. Both facts were true. The
conclusion was not. The producer was a cron job in a private overlay, installed
by a bootstrap script, staging real traffic every two minutes — and the host
sender was answering real people every sixty seconds. The second design, written
hours later by a different author who happened to read the workflow
documentation rather than grep the source, opened by correcting exactly this.

The cost was not the stale sentence. It was the decision resting on it: a send
switch was recommended to default **off**, priced as a ramp onto a cold start.
Against a live loop, that default silences a reply path people already use,
without telling them. The recommendation had to flip.

## Root cause

The absence of a caller in the repository you can read is not the absence of a
caller. A grep proves one repository's contents at one commit; it proves nothing
about overlays, cron jobs, host launch agents, or anything installed rather than
committed. "No producer exists" was a claim about a search, dressed as a claim
about production.

The failure mode is quiet. A safety argument built on absence is true when
written and false the moment anything is deployed, and nothing fails when it
turns: no test breaks, no reviewer is paged. It is simply carried forward into
every downstream plan as a premise nobody re-checks.

## The convention that replaces it

A pull request claims inertness from a **gate**, not from a vacuum: "this path
is not reached until <named owner action>", and the action is named. Credential
not minted. Setting not enabled. Key file absent, so the script exits 0. Each of
those is checkable later and fails loudly when it stops being true.

Where a change does land on a live path, say so and justify it narrowly — that
it only ever *reduces* what the system does, never adds a call — rather than
claiming it is inert.

Classifying every planned pull request this way, once, in the design's scope
section, is what makes the implementation plan able to repeat it per pull
request without re-deriving it.

## How to check before writing the claim

Read the operating documentation — the workflow or runbook that says what runs
on a schedule and who owns each step — before concluding from source alone. A
grep over one repository answers "is it committed here", and the question being
asked is "is it running".

## Links

- Amendment pull request: `Zhachory1/docs#17`
- The two designs it amends: `Zhachory1/docs#14`, `Zhachory1/docs#15`
