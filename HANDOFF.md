# Handover — Cross-repo fix from qhorus #459

## What Was Done

Commit `ba1193c` on main — fix(#214): repair compilation errors after DecisionRecord/ComplianceReport record evolution and LedgerMerkleFrontierRepository package move.

Files changed:
- `ComplianceReportServiceCore.java` — updated to 9-param ComplianceReport and 11-param DecisionRecord constructors, added `reportForTenancy()` method
- `MerkleVerificationBundleService.java` — fixed import to `api.spi.LedgerMerkleFrontierRepository`
- `InMemoryLedgerEntryRepository.java` — same import fix
- `LedgerComplianceReportServiceIT.java` — fixed import to `io.casehub.ledger.jpa.*`

## Issue Filed

casehubio/ledger#214 — tracks these compilation fixes.

## Remaining

The `rest/` module tests still have compilation errors (DefaultLedgerEntryApiTest, DefaultLedgerTrustApiTest — constructor signature mismatches). These were not fixed because they're not on the critical path for qhorus CI.

## Context

Triggered by qhorus#459 — qhorus CI fails because the published ledger-core SNAPSHOT on GitHub Packages (Sep 16) predates the record evolution. The ledger CI needs to pass and deploy a new SNAPSHOT for qhorus CI to go green.
