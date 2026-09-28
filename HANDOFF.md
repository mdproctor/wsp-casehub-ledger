# Handover — Cross-repo fix from qhorus #459

## What Was Done

Four commits on ledger main for ledger#214:

1. `ba1193c` — fix stale constructor calls in ComplianceReportServiceCore, add reportForTenancy(), fix stale LedgerMerkleFrontierRepository imports
2. `0a9f9e3` — fix REST API test constructors for constructor injection
3. `e500cf9` — exclude signing-spring from reactor (depends on unimplemented modules)
4. `dd0e7ad` — compute ComplianceSummary in reportFor* methods (was null, tests expected non-null)

## CI Status

Run 36499706727 on casehubio/ledger in progress. Once it passes, SNAPSHOT will be published to GitHub Packages. Qhorus CI will then resolve the updated version.

## Remaining

- `signing-spring` excluded from reactor until the 4 signing backend modules are created
- Once SNAPSHOT is published, qhorus CI should go green without any qhorus code changes
