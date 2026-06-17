# Session Handoff — 2026-06-17

## Branch: issue-149-actor-identity-producer (open)

Three issues addressed in one session on branch `issue-149-actor-identity-producer`.

**#149** (XS/Low) — `LedgerPrivacyProducer` injected `EntityManager` eagerly, causing
`UnsatisfiedResolutionException` on `ActorIdentityProvider` in datasource-free deployments
(e.g. apps using casehub-qhorus without a ledger JPA datasource). Fixed: changed class-level
`EntityManager em` to `Instance<EntityManager> emInstance` — always satisfiable by Arc;
`emInstance.get()` only called when tokenisation is enabled. Two unit tests added.
PR #152 open, 804 runtime tests green.

**#140** (M/Med) — `ErasureReceiptLedgerEntry` (tamper-evident GDPR Art.17 erasure record).
Deferred by user decision — wait for a real-world second consumer to surface before
promoting from devtown-specific to foundation.

**#126** (M/Med) — Decouple MCP tool telemetry from `MessageType.EVENT` content. Closed
as already done: casehubio/qhorus#257 delivered the `telemetry` field on `MessageDispatch`
and routed `LedgerWriteService.populateTelemetry()` to `dispatch.telemetry()`. Protocol
`PP-20260608-054090` settled the architectural question — EVENT stays content-free; STATUS
is the canonical type for content-bearing observe-channel messages.

## Current state

- Project branch `issue-149-actor-identity-producer`: 1 commit ahead of main, PR #152 open
- Workspace branch: `issue-149-actor-identity-producer`
- casehubio/ledger main: `19634f3`
- All 804 runtime tests pass

## Immediate Next Step

Merge PR #152 (trivial, single clean commit). Then run `work-end` to close the branch.

## What's Left

- PR #152 merge → `work-end` · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #102 | Cloud KMS AgentSigner adapters (AWS, GCP, Azure) | L | Med | blocker #85 closed — ready |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | High | blocker #85 closed — ready |
| #96 | Code-generation for reactive service tier | L | High | wait until service pair count ≥ 5 |
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | wait for casehub-ops consumer |
| #140 | ErasureReceiptLedgerEntry | M | Med | deferred — wait for second consumer |
