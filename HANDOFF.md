# Session Handoff — 2026-07-13

## Branch closed: issue-179-named-query-migration

Migrated all remaining inline JPQL in `JpaLedgerEntryRepository` to
`@NamedQuery` declarations (#179). Added 6 tenant-scoped named queries to
`JpaLedgerEntry`, 4 tenant-scoped attestation queries (with JOIN) to
`LedgerAttestation`, and 2 supplement batch-loading queries to
`JpaComplianceSupplement` / `JpaProvenanceSupplement`. Zero `em.createQuery()`
calls remain in any production JPA repository — all queries are now validated
at Hibernate boot.

## Current state

- `casehubio/ledger` main: `448ca02` — pushed
- Issue #179 closed

## Paused branch

- `issue-137-artifact-trust-scoring` #137 — paused mid-brainstorm. No consumer exists yet.

## Immediate Next Step

Pick from What's Next — no trailing obligations.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #178 | Field-level GDPR erasure for metadata containing PII | M | Med | Deferred — only needed if PII enters metadata |
| #171 | Browser-based Vault OIDC flow (two-step auth URL + callback) | M | Med | Not needed until interactive admin tooling |
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | Paused — waiting for casehub-ops consumer |
| #96 | Code-generation for reactive service tier | L | High | Wait until service pair count >= 5 |
