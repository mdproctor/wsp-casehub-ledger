# Session Handoff — 2026-06-17

## Branch closed: issue-148-h2-sequence-concurrent

Four issues closed in one session covering H2 concurrency, API design, and engine migration.

**#148** — `LedgerSequenceAllocator` H2 MERGE race condition fixed. H2 2.x MVStore
evaluates `MERGE INTO` non-atomically; replaced with `INSERT ON CONFLICT DO NOTHING +
UPDATE` (identical to PostgreSQL path). H2 2.2+ in `MODE=PostgreSQL` supports `ON
CONFLICT DO NOTHING` (no column list). Dialect split eliminated. Unblocked casehubio/aml CI.

**#136** — Batch scoring API added to `TrustScoreSource` (api/spi): `scoresFor()` and
`decisionCountsFor()` as `default` methods with loop fallback. `MaterializedTrustScoreSource`
overrides with single `WHERE actorId IN (...)` query. `TrustGateService` exposes blocking
+ `Uni<>` async variants. Unblocks engine `ImplementationRoutingStrategy`.

**#142** — `ActorIdentityProvider` moved from `runtime/privacy/` to `api/spi/` per
`consumer-spi-placement` protocol. `tokeniseForQuery()` now returns `Optional<String>`:
empty = null input only; present = always query (token or raw actorId). Protocol
`PP-20260617-4d345f` formalised.

**#123** — `TrustScoreCache` deleted from casehub-engine-ledger. `TrustCandidateClassifier`,
`TrustWeightedAgentStrategy`, `WorkerDecisionEventCapture` now inject `TrustScoreSource`.
`CachedTrustScoreSource` selected in engine test config.

## Current state

- `casehubio/ledger` main: `19634f3` — 4 squashed commits ahead of `9184e8a`
- casehub-engine branch `issue-123-trust-score-source-migration`: committed (`90697a72`), not merged
- All tests pass; 802 runtime + 58 persistence-memory + 65 engine-ledger
- All 4 GitHub issues closed

## What's Next

No open issues identified for this branch. Next work is discretionary — pick from the backlog.

**Pending:** casehub-engine `issue-123-trust-score-source-migration` branch needs its own
PR or merge into casehub-engine main — not yet done.
