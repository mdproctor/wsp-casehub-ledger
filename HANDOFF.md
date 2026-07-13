# Session Handoff — 2026-07-13

## Branch closed: issue-175-txn-requires-new-and-examples-pom

Fixed `saveAttestation()` transaction isolation (#175) — `REQUIRES_NEW` so
attestation failures don't roll back the caller's entry save. Also removed
the `@Transactional` wrapper from `OutcomeRecordSaveService` so each write
commits independently. Added `@NamedQuery("LedgerEntry.findByIdAndTenancyId")`
for protocol compliance. Added `examples/pom.xml` aggregator (#177) for
casehub-examples integration. Filed #179 for remaining inline JPQL migration.

## Current state

- `casehubio/ledger` main: `330bf1f` — pushed
- Issue #175 closed, #177 closed
- Issue #179 filed (remaining inline JPQL → @NamedQuery migration)

## Paused branch

- `issue-137-artifact-trust-scoring` #137 — paused mid-brainstorm. No consumer exists yet.

## Immediate Next Step

Pick from What's Next — no trailing obligations.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #179 | Migrate remaining inline JPQL to @NamedQuery in JpaLedgerEntryRepository | S | Low | Mechanical — move inline queries to entity annotations |
| #178 | Field-level GDPR erasure for metadata containing PII | M | Med | Deferred — only needed if PII enters metadata |
| #171 | Browser-based Vault OIDC flow (two-step auth URL + callback) | M | Med | Not needed until interactive admin tooling |
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | Paused — waiting for casehub-ops consumer |
| #96 | Code-generation for reactive service tier | L | High | Wait until service pair count >= 5 |
