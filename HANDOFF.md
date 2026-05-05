# CaseHub Ledger — Session Handover
**Date:** 2026-05-05

## Current State

`casehub-ledger` v0.2-SNAPSHOT. Clean working tree. 345 tests, BUILD SUCCESS.
Group A (#49) fully closed: #57 ✅ #59 ✅ #56 ✅ #58 ✅. Group B was already done.

## What Landed This Session

**Group A — four independent features:**
- `AttestationAggregator` — consensus verdict before trust scoring; WEIGHTED_MAJORITY/
  UNANIMOUS_REQUIRED/FIRST_ATTESTOR; `casehub.ledger.trust-score.aggregation-strategy`
- `@ProvenanceCapture` interceptor — auto-attaches `ProvenanceSupplement` via existing
  enricher pipeline; `@SourceEntityId` param annotation; ThreadLocal stack context
  (not `@RequestScoped` — works in schedulers and `@QuarkusTest` without HTTP)
- `LedgerHealthJob` — scheduled gap detection (JPQL GROUP BY/HAVING) + reconciliation
  SPI; `LedgerGapDetected` CDI event; `casehub.ledger.health.*` config
- `LedgerComplianceReportService` — `reportForActor`/`reportForSubject`; `ComplianceReport`
  with `format(PLAIN_JSON/CSV/JSON_LD)`; `findBySubjectIdAndTimeRange` added to both
  repository SPIs

**Documentation audit:** 6 files — CLAUDE.md structure table fully corrected (new files,
wrong locations fixed), docs-wide config prefix `quarkus.ledger` → `casehub.ledger`,
EigenTrust key names fixed, integration-guide.md got 3 new consumer sections.

## Immediate Next Steps

1. **Group C / D** — check remaining open issues in epic #48 subtree:
   `gh issue list --repo casehubio/ledger`
2. **Milestone** — Group A+B done; consider `gh release create --generate-notes`

## References

| What | Path |
|---|---|
| Latest blog | `blog/2026-05-05-mdp02-the-proxy-and-the-bean.md` |
| Latest ADRs | `adr/0006` – `adr/0008` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
