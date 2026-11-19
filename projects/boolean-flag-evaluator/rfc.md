# RFC: Boolean Flag Evaluator for ralph-canary

## Status and source binding

Status: Revised design for fresh Scope, Architecture, and Security Council review. This document is not implementation, deployment approval, or authorization to begin implementation.

This revision responds to Architecture review of RALA-33 (issue UUID `8d8db705-daaa-43e4-a196-1f6ed90dc116`), native document key `rfc`, exact source revision `42346ef7-68d0-4308-a04d-8a8212cbf047`. The source remains unchanged; this document is the revised copy on RALA-37.

The design is for a local draft-PR MVP: a boolean flag evaluator, PostgreSQL as authoritative state and history, and an optional Redis read cache with a fixed five-minute TTL. It defines behavior and bounds, not production readiness or deployment prerequisites.

## Goals and non-goals

Goals:

- Read a global boolean value and its version by flag key.
- Create and update flags through an authenticated administrative API.
- Store the current value and attributable change history transactionally.
- Use Redis as a disposable optimization for reads, with explicitly bounded cache-entry lifetime.
- Make validation, concurrency, and database/cache failure behavior explicit.

Non-goals: traffic splitting, cohorts, percentage rollout, targeting rules, user or tenant attributes, multivariate values, scheduled changes, bulk import, delete/restore, SDK-side evaluation, a cache-invalidation outbox, retry worker, deployment/rollout feature, production-capacity plan, or migration program. A missing flag is not silently interpreted as `false`.

## MVP boundary and assumptions

The MVP is a local draft-PR design and verification target. PostgreSQL is authoritative. Redis is optional and disposable; correctness must not depend on Redis being available. The API/service is the supported writer. Database transactions provide atomic current-state and history updates. The local environment's configured database disk and resource limits are the storage boundary; this RFC makes no production volume, throughput, latency, availability, or capacity claim.

Production-scale deployment, capacity budgets, thresholds, operational rollout, and any future version-expansion or migration plan are separate decisions. They are not features or acceptance criteria of this MVP. No update rate or timeline to version exhaustion is asserted.

## Evaluator and API contract

All endpoints accept and return JSON and use authenticated caller identity. Errors use `{code, message, request_id}` and do not disclose SQL, Redis, credentials, or internal exception details. Versions are decimal strings in JSON and ETags so clients preserve the exact integer.

- `GET /v1/flags/{flag_key}` returns `200 {flag_key, value, version, updated_at}`. Unknown keys return `404 FLAG_NOT_FOUND`.
- `GET /v1/admin/flags` lists flags for an authorized reader, ordered by key, with opaque cursor pagination and a bounded page size (default 50, maximum 200).
- `POST /v1/admin/flags` creates `{flag_key, value}`. Success is `201` with the canonical record and `ETag: "<version>"`; duplicate keys return `409 FLAG_ALREADY_EXISTS`.
- `PUT /v1/admin/flags/{flag_key}` updates `{value}` and requires `If-Match: "<version>"`. Success is `200` with the canonical record and new ETag. Missing precondition returns `428 PRECONDITION_REQUIRED`; stale version returns `412 VERSION_MISMATCH` with current version; missing key returns `404`.
- `GET /v1/admin/flags/{flag_key}/history` returns newest-first, paginated history to an authorized audit reader.

Every successful create starts at version 1. Each successful update increments by exactly one. An explicitly submitted same-value update is still an update and is recorded. Failed requests do not change state or history. `updated_at` is a UTC database timestamp set in the transaction; cache hits may return an earlier timestamp, so clients compare versions rather than timestamps. There is no delete endpoint: deletion would change missing-key semantics and needs a separate design.

## Validation and injection defense

Validate `flag_key` at the HTTP boundary and service/domain boundary: 1–128 ASCII characters matching `^[a-z][a-z0-9_.-]{0,127}$`. Reject whitespace, slashes, control characters, invalid UTF-8, and Unicode lookalikes. Treat keys only as data, never as table names, SQL fragments, Redis commands, HTML, or log format strings. Use parameterized database access; never interpolate identifiers from requests. Reject oversized bodies before parsing and reject unknown fields in write requests.

`value` must be a JSON boolean, not a string, number, null, or coercible value. Apply server-side validation regardless of UI checks. Escape rendered values for their output context, and do not echo untrusted payloads in logs or validation errors. Bound pagination and request sizes. Administrative writes require authorization and should be rate-limited in the application; those controls do not replace authorization.

## PostgreSQL schema, transaction, and version limit

`flags`:

- `id UUID PRIMARY KEY`
- `flag_key VARCHAR(128) NOT NULL UNIQUE`
- `current_value BOOLEAN NOT NULL`
- `current_version BIGINT NOT NULL CHECK (current_version >= 1)`
- `created_at TIMESTAMPTZ NOT NULL`
- `updated_at TIMESTAMPTZ NOT NULL`

`flag_history`:

- `id UUID PRIMARY KEY`
- `flag_id UUID NOT NULL REFERENCES flags(id)`
- `flag_key VARCHAR(128) NOT NULL` (key snapshot for readable history)
- `value BOOLEAN NOT NULL`
- `version BIGINT NOT NULL CHECK (version >= 1)`
- `actor_id TEXT NOT NULL`
- `request_id UUID NOT NULL`
- `operation TEXT NOT NULL CHECK (operation IN ('create','update'))`
- `created_at TIMESTAMPTZ NOT NULL`
- `UNIQUE(flag_id, version)`

Use a unique index on `flags.flag_key` and an index supporting history lookup by flag and descending version. History rows are append-only to the application role. There is no application history-update/delete endpoint or automated history-pruning job in this MVP.

The version representation is PostgreSQL signed `BIGINT`, with valid positive versions from 1 through `9,223,372,036,854,775,807` (`2^63 - 1`). Create inserts the current record and version-1 history row in one transaction. Update is conditional on both key and expected version; it updates the current row, appends the matching history row, and commits both atomically. A stale expected version changes neither row. Database uniqueness and conditional updates are the concurrency controls; there is no last-write-wins update.

At the maximum version, reject a further update with `409 VERSION_EXHAUSTED`; leave current state and history unchanged. Never wrap, reset, or reuse a version. This RFC does not prescribe a production migration or wider-version program. The limit is stated as a representation bound, not a prediction about when it will be reached.

## History and data retention for the local MVP

Retain one immutable history row for every successful create and update, alongside the current flag row. Keep actor, request ID, operation, key snapshot, value, version, and database timestamp. Failed operations do not create history. The local MVP has no automatic time-based deletion or pruning: history remains for as long as the local database dataset is retained. Resetting or deleting that local dataset removes its flags and history together. This is a simple local-development assumption, not a compliance, backup, legal-retention, or production-retention claim.

History grows with successful writes and therefore consumes the configured local database storage. If storage is unavailable or exhausted, PostgreSQL rejects the transaction; the current value and its history remain unchanged because the transaction is atomic. The RFC does not claim a row-count, disk-capacity, or growth-rate budget. Any production retention policy and capacity planning are outside this MVP and require separate review.

## Redis read cache and consistency behavior

Cache key: `ralph-canary:flag:v1:<flag_key>`. A payload contains the key, boolean value, decimal-string version, and `updated_at`. Cache only found records; do not cache not-found responses. Set a fixed TTL of exactly 300 seconds on successful population. Cache hits do not extend or slide that TTL. Redis is disposable and may be flushed.

Read behavior:

1. Validate the key, then try Redis.
2. On a valid cache hit, return that snapshot. It may be stale relative to a database write whose eviction failed or was missed.
3. On miss, Redis error/timeout, or unusable payload, read PostgreSQL. If the database read succeeds, return that result and attempt to populate Redis for 300 seconds; cache population failure does not fail the read.
4. A successful database read concurrent with a write may race with its post-commit eviction. A snapshot populated after an eviction can itself be stale; any such stale cache entry still has a fixed maximum lifetime of 300 seconds from that successful population. Do not claim stronger cross-store ordering or linearizable reads.

The Director's cache decision is fixed for this design: after a PostgreSQL write commits, make a best-effort Redis eviction for the affected key. The write does not wait for eviction and does not fail solely because eviction failed. There is no transactional outbox, retry worker, pending-write response, or `202` invalidation-pending behavior. Do not make invalidation a condition for reporting a committed write.

If eviction fails or is missed, a cached stale value can be returned until the remaining TTL expires. Its maximum stale lifetime is 300 seconds from its last successful population; a later stale cache population starts its own fixed TTL. After expiry, a read must obtain current state from PostgreSQL before returning a value. This is a bounded-staleness cache contract, not immediate consistency.

## PostgreSQL and Redis outage behavior

PostgreSQL outage:

- A cache hit may still serve the cached snapshot until that entry's existing TTL expires. It cannot establish that the value is current; callers receive the cached version and should not infer freshness from a successful response alone.
- On a cache miss, expired entry, or unusable cache payload, the service cannot produce an authoritative value and returns a service-unavailable error rather than inventing a default or presenting a cache miss as `404`.
- Create and update require PostgreSQL. If the database transaction cannot commit, return an error and make no successful mutation; current state and history remain unchanged. Do not evict Redis for an uncommitted write.
- Once PostgreSQL is available, database-backed reads resume. No cross-store availability or recovery-time guarantee is made.

Redis outage:

- Reads fall back to PostgreSQL. If PostgreSQL is available, return its result; cache population is best-effort and may be skipped until Redis is usable.
- Writes continue through PostgreSQL. After commit, attempt eviction best-effort; Redis failure does not undo the commit or change the successful write response.
- An already populated entry may be served by Redis only if Redis itself still serves it. If Redis is unavailable, it is not a read source; the database fallback applies. Following Redis recovery, any surviving stale entry expires no later than 300 seconds after its last successful population. No immediate invalidation or linearizable-read guarantee is made during failure.

If both stores are unavailable, no authoritative read or write is available. The service returns an error and does not fabricate values. Missing keys remain distinct from store failures.

## Administration and security

All admin routes require authenticated identity and server-side authorization on each request; deny by default. Keep evaluator reads distinct from admin mutations so a machine identity permitted to evaluate cannot mutate flags. Support reader, auditor, and operator capabilities: readers see current flags; auditors additionally read history; operators may create/update. Role mapping is supplied by the application's existing identity/authentication integration; this RFC does not prescribe a provider product or require a new identity platform as an MVP feature.

Every successful mutation records actor and request ID in durable PostgreSQL history. Do not expose database or Redis credentials, internal exception details, or secrets in responses or logs. Protect browser-based admin mutations against CSRF when cookie sessions are used. Audit reads and administrative actions should be attributable. Redis must not be exposed as a public client interface; cache contents contain only flag state, not secrets or personal data. The exact production network, encryption, identity-provider, and credential-management setup is deployment-specific and outside this local design's readiness criteria.

## Verification scope

The draft-PR MVP should verify the behavioral contract with focused tests:

- create, update, history, uniqueness, atomic rollback, and actor/request attribution;
- boolean-only input, key validation, bounded pagination, and safe error envelopes;
- missing and stale `If-Match`, concurrent updates with only one winner for a given version, and the signed-BIGINT maximum/exhaustion guard;
- history append-only behavior and no history for failed writes;
- Redis hit, miss, fixed 300-second non-sliding TTL, corrupt/unusable payload fallback, cache-population failure, and stale-entry expiry;
- best-effort post-commit eviction, failed eviction that does not undo a committed write, and database fallback when Redis is unavailable;
- PostgreSQL outage behavior for cache hits, cache misses, and writes; Redis outage behavior; and missing-key versus backend-failure distinction;
- authentication/authorization denial, CSRF where applicable, and injection-resistant handling of untrusted keys and values.

These are correctness tests for the local design, not production load tests, capacity thresholds, rollout gates, or deployment-readiness requirements. No throughput target or operational scale claim is made.

## Decisions and limitations

This RFC specifies a boolean-only evaluator, PostgreSQL-authoritative storage, append-only local-MVP history, bounded signed-BIGINT versions, Redis read-through caching with fixed 300-second TTL, and best-effort post-commit eviction. It intentionally excludes traffic splitting, outbox/retry machinery, pending-write semantics, production deployment design, production capacity planning, and a migration program. Fresh Scope, Architecture, and Security review must assess this exact revision before any approval or implementation decision.