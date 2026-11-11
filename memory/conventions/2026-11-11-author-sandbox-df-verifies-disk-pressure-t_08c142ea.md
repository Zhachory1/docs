---
kind: convention
source: t_08c142ea
profile: ralph-author
date: 2026-11-11
---

# An author sandbox cannot read `hermes-home`, but `df` inside it does measure both the VM disk and the host disk, so disk-pressure claims are verifiable from an author card

## What this adds

The root-cause note of the same date
(`memory/root-causes/2026-11-11-author-sandbox-cannot-read-live-boards-t_93c40618.md`)
records what an author sandbox cannot see: no `hermes` binary, no board
database, no `/home/hermes/.hermes/profiles`. All still true, re-checked on
2026-11-11.

That makes it tempting to treat *any* infrastructure figure in a brief as
unverifiable and simply restate it. For disk pressure, that is wrong.

## What is verifiable

From inside the sandbox, `df -h` reports both filesystems that matter:

- the **VM disk**, as the filesystem backing the sandbox container's own root
  (an `overlay` on `/dev/vdb1`);
- the **host disk**, as the `virtiofs` mount backing `/root` and `/workspace`
  (the `RALPH_SANDBOX_ROOT` bind, per Zhachory1/ralph#97).

`grep virtiofs /proc/mounts` confirms which paths are the host share. So "is the
VM still under pressure" and "does the host have room for what we want to move
there" are both answerable on an author card, with real numbers.

## What is still not verifiable

Anything *inside* the `hermes-home` volume: per-profile cache sizes, the board
database, any profile's config. Those figures stay the measurer's, carried with
provenance and recorded as an assumption.

## Where this was used

`prd-ralph-sandbox-home` (Zhachory1/docs#28, `projects/ralph-sandbox-home/prd.md`)
for Zhachory1/ralph#160: the two `df` figures are this run's own measurement; the
per-profile breakdown inside the volume is the owner's, flagged as assumption 1.
