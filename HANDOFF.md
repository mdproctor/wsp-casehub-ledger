# Handover — Cross-repo fix from qhorus #459

## What Was Done

Three commits on `mdproctor/ledger` main for ledger#214:

1. `ba1193c` — fix stale constructor calls in ComplianceReportServiceCore (9-param ComplianceReport, 11-param DecisionRecord), add missing `reportForTenancy()` method, fix stale imports for LedgerMerkleFrontierRepository in runtime and persistence-memory, fix JpaComplianceSupplement import in test
2. `0a9f9e3` — fix REST API test constructors (DefaultLedgerEntryApiTest, DefaultLedgerTrustApiTest) for constructor injection refactoring
3. `e500cf9` — exclude signing-spring and spring-integration-test from reactor (depend on vault-transit/aws-kms/gcp-kms/azure-keyvault modules not yet created)

## CI Status

Waiting for ledger CI run on HEAD (e500cf9) to pass and deploy SNAPSHOT to GitHub Packages. Two earlier runs in progress will fail (don't have all fixes).

## Remaining

- `rest/` module REST test fixes are included — those tests should pass
- `signing-spring` excluded from reactor until the 4 signing backend modules are created
- Once CI deploys, qhorus CI should go green automatically
