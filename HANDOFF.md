# Session Handoff — 2026-06-18

## Branch closed: issue-149-actor-identity-producer

Four issues addressed in one session across two bug fixes and one foundation feature promotion.

**#149** — `LedgerPrivacyProducer` injected `@LedgerPersistenceUnit EntityManager` eagerly.
In datasource-free deployments (e.g. apps using `casehub-qhorus` without a ledger JPA
datasource), CDI augmentation failed with `UnsatisfiedResolutionException` on
`ActorIdentityProvider`. Fix: `Instance<EntityManager>` — always satisfiable by Arc;
`.get()` only called when tokenisation is enabled.

**#150** — `LedgerSequenceAllocator` used `INSERT ON CONFLICT DO NOTHING` which requires
`MODE=PostgreSQL` in H2. Downstream modules (casehub-engine-ledger) using plain H2 got
`JdbcSQLSyntaxErrorException` on every ledger write. Fix: dialect detection via
`INFORMATION_SCHEMA.SETTINGS` (URL not reliable — Agroal strips connection properties);
H2 standard mode gets SQL-standard MERGE path.

**#126** — Closed as already done: casehubio/qhorus#257 delivered the telemetry field
decoupling; protocol `PP-20260608-054090` established STATUS as the canonical type for
content-bearing observe-channel messages.

**#140** — `ErasureReceiptLedgerEntry` promoted from devtown-specific to `casehub-ledger`
foundation. Opt-in: `casehub.ledger.erasure-receipt.enabled=true` (default false).
`ErasureReason` enum: GDPR_ART_17_REQUEST | RETENTION_EXPIRED | ACCOUNT_DELETION.
V1010 migration. `LedgerErasureService.erase()` now takes `ErasureReason`; `ErasureResult`
carries `Optional<UUID> receiptEntryId`. devtown migration tracked in casehub-devtown#82.

## Current state

- `casehubio/ledger` main: `38d9f20` — all 4 issues closed, PR #152 merged
- 810 runtime + 58 persistence-memory tests pass
- GE-20260618-73a023 submitted to garden: H2 `getMetaData().getURL()` drops connection properties
- parent#274 filed: sync PLATFORM.md and casehub-ledger.md for this session's changes

## Immediate Next Step

Pick from the backlog — no open obligations. `#102` (Cloud KMS AgentSigner) and `#101`
(Vault AppRole/OIDC) both had their blockers close and are ready.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #102 | Cloud KMS AgentSigner adapters (AWS, GCP, Azure) | L | Med | blocker #85 closed — ready |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | High | blocker #85 closed — ready |
| #96 | Code-generation for reactive service tier | L | High | wait until service pair count ≥ 5 |
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | wait for casehub-ops consumer |
