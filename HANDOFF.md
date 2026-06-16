# Session Handoff — 2026-06-17

## Branch closed: issue-144-identity-binding-merkle-and-tenancy

Two correctness bugs in `ActorIdentityBindingRepository` fixed:

**#144 — Merkle frontier gap**
- `JpaActorIdentityBindingRepository.save()` never called `frontierRepo.replace()`
- `InMemoryActorIdentityBindingRepository.save()` was assigning `sequenceNumber=0`, `digest=null` — no pipeline at all
- Root cause: dedicated `save()` method duplicated the `LedgerEntryRepository` pipeline incompletely
- Fix: `ActorIdentityBindingRepository` is now read-only (no `save()`). Observer calls `ledgerRepo.save(entry, tenancyId)` — full pipeline runs

**#145 — Cross-tenant reads**
- `latestBindingFor(actorId)` and `bindingHistoryFor(actorId)` had no `tenancyId` parameter
- Named queries ordered by `occurredAt` (non-deterministic within same millisecond)
- Fix: both methods now take `tenancyId`; named queries filter by it and order by `sequenceNumber`

**Design decisions:**
- `ActorDIDEnricher` gets `instanceof ActorIdentityBindingEntry` guard — prevents event loop (makes it unconditionally impossible, not cache-bounded) and prevents `boundDid`/`actorDid` discrepancy
- `InMemoryActorIdentityBindingRepository` delegates reads to `InMemoryLedgerEntryRepository.allEntries()` (mirrors `InMemoryKeyRotationRepository`)
- Protocol `PP-20260616-7d4171` formalised: LedgerEntry subclass repositories must be read-only

## Current state

- `casehubio/ledger` main: 2 squashed commits ahead of previous state
- All H2 tests pass (BUILD SUCCESS, 3m 41s); PostgreSQL tests require Docker
- Issues #144 and #145 closed on GitHub
- Blog entry published: `2026-06-16-mdp02-the-repository-that-stopped-short.md`

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #146 | `KeyRotationRepository` tenancy gap — same structural issue as #145; requires design decision on cross-tenant SUSPECT detection semantics | S | Med | Filed this session; may be intentionally actor-scoped |
