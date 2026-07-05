# Session Handoff — 2026-07-05

## Branch closed: issue-168-extract-ledger-write-spi

Extracted ledger write SPI to api tier (#168). Two-tier JPA model hierarchy:
`api.model.LedgerEntry` (`@MappedSuperclass`, all persistent fields, `canonicalBytes()`)
and `runtime.model.jpa.JpaLedgerEntry` (`@Entity`, JPA machinery). Supplement JOINED
inheritance eliminated — each type independent `@Entity`. `LedgerEntryRepository` and
`ReactiveLedgerEntryRepository` moved to `api/spi`. New `LedgerAppender` SPI with
`AuditRecord` value type. Dead api model duplicates deleted, `ScoreType` extracted.

Design review: 4 rounds, 18 issues, all resolved. ADR 0017 recorded.
Garden entry GE-20260705-7c0e86 (Hibernate bytecode enhancement strips @Transient from @MappedSuperclass).

Also closed #161, #165 (already implemented in prior sessions).
Filed #173 (engine NoOp consolidation), #174 (platform persistence unification).

## Current state

- `casehubio/ledger` main: `e021fd4` — pushed
- Issues #168 closed; #161, #165 closed; #173, #174 filed

## Paused branch

- `issue-137-artifact-trust-scoring` #137 — paused mid-brainstorm. No consumer exists yet.

## Immediate Next Step

Pick from What's Next — no trailing obligations.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #173 | Consolidate engine NoOpLedgerEntryRepository copies into shared test-support artifact | S | Low | Unblocked by #168 |
| #162 | REST endpoints for ledger entry query + Merkle proof | M | Med | Gates casehub-aml workbench UI |
| #171 | Browser-based Vault OIDC flow (two-step auth URL + callback) | M | Med | Not needed until interactive admin tooling |
| #137 | Artifact trust scoring (content-hashed artifacts) | L | High | Paused — waiting for casehub-ops consumer |
| #96 | Code-generation for reactive service tier | L | High | Wait until service pair count >= 5 |
