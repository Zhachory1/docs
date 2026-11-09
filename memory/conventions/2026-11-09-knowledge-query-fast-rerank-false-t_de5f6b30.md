---
kind: convention
source: https://github.com/Zhachory1/ralph/pull/131
profile: ralph-researcher
date: 2026-11-09
---

# Search the knowledge index with typed lex and vec searches and rerank false

Query the knowledge index with typed `searches` — a `lex` entry holding the
exact terms and a `vec` entry holding the question in plain words — and pass
`rerank: false`. Never pass a plain-text `query`: that path runs query
expansion, which downloads a 1.3 GB expansion model and is slow.

Measured from the hermes container on three questions, comparing the fast
path against re-ranking:

| Question | rerank: false | re-ranked | Same top hit? |
| --- | --- | --- | --- |
| where do agents write memory notes | 13.6 s (first call, embed model load) | 238.5 s | yes |
| why was the knowledge memory limit raised | 0.8 s | 107.8 s | yes |
| who approves an implementation plan | 1.0 s | 143.0 s | yes |

All three questions returned the same top hit either way, so re-ranking cost
100+ seconds of CPU time without changing the answer. Reserve re-ranking for
cases where the fast results look wrong and the answer matters.

This is a follow-up to ralph#121 (defaults were too slow: personal-instance
median query 30 s, worst 278 s across 13 queries in 24 h; one work-instance
query took 117 s). The change landed in `PRINCIPLES.md`, `docs/infrastructure.md`,
and `docs/workflows.md` on the ralph repo.
