# CaseHub Ledger — Session Handover
**Date:** 2026-06-03

## Current State

604 tests, BUILD SUCCESS. Both repos on `main`.
**Issue #114 (OutcomeRecorder / lightweight outcome-tracking) shipped and merged to upstream/main.**
Issue-105-106-reactive-parity branch is open; its primary commit (TrustGateService.meetsThresholdAsync) was delivered as part of the #114 squash — the branch still needs a proper work-end.

## What Landed This Session

- **ledger#114**: OutcomeRecord, OutcomeRecorder/ReactiveOutcomeRecorder SPIs, DefaultOutcomeRecorder (non-@Transactional outer + @Transactional delegate), BlockingToReactiveOutcomeRecorder, PlainLedgerEntry (@DiscriminatorValue("PLAIN") + V1009 migration), EigenTrustStartupValidator, LedgerConfig.OutcomeConfig. 604 tests. Closed.
- **ledger#106**: TrustGateService.meetsThresholdAsync Uni<Boolean> bridge — delivered inside the #114 squash to upstream/main.
- **Design routing bug fixed**: `design → workspace` routing in CLAUDE.md caused work-end to look for DESIGN.md in the workspace (doesn't exist) instead of the project. Fixed in ledger + parent + connectors + claudony + qhorus. All `.meta` files now correctly set `design-repo: project`.
- **ADRs**: 0016 (EigenTrust inapplicability for star-graph/single-attestor deployments), 0017 (PlainLedgerEntry as domain-agnostic concrete LedgerEntry).
- **Garden**: GE-20260602-6941d6 (separate @Transactional delegate controls commit timing), GE-20260602-298736 (Uni.<Void>item() type witness for void Mutiny lambdas).
- **Protocol**: PP-20260602-a44c4e (ledger write API outer methods must not be @Transactional).
- **Deferred issues filed**: #115 (incremental trust pipeline), #116 (JPA sequenceNumber gap), #117 (in-memory store memory growth), #118 (on-read trust computation), #119 (EigenTrustStartupValidationIT).
- **Blog**: `2026-06-02-mdp01-the-write-that-commits-before-returning.md` published.

## Immediate Next Step

Update the quarkmind HANDOFF.md to clear the ledger#114 blocking dependency. Then `/work` to resume `issue-105-106-reactive-parity` and run work-end on it (the meetsThresholdAsync feature is already in upstream/main; the branch just needs to be officially closed).

## Cross-Module

**We unblock:**
- `quarkmind` — ledger#114 shipped; OutcomeRecorder + TrustWeightedAgentStrategy trust routing now available. QuarkMind can now wire in `OutcomeRecorder.record()` with `casehub.ledger.trust-score.routing-enabled=true`.

## What's Left

- Close `issue-105-106-reactive-parity` via work-end (feature already delivered; branch needs work-end scaffold) · XS · Low
- Local project `main` is behind `upstream/main` (force-push during squash diverged local/upstream). Run `git -C /path/to/ledger fetch upstream && git -C /path/to/ledger reset --hard upstream/main` on local main to resync. · XS · Low
- Backup branch `backup/pre-squash-issue-114-lightweight-mode-YYYYMMDD` exists locally — delete after 14 days if no issues. · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #115 | Incremental per-actor trust recomputation | L | High | Requires casehub-engine TrustScoreCache per-actor refresh |
| #116 | JPA sequenceNumber gap in JpaLedgerEntryRepository | M | Med | Pre-existing; blocks JPA consumers of OutcomeRecorder |
| #119 | EigenTrustStartupValidationIT (@QuarkusTestResource log capture) | S | Med | Integration test verifying CDI wiring of EigenTrustStartupValidator |
| — | QuarkMind: wire OutcomeRecorder into game loop | M | Low | Consumer change in quarkmind; unblocked by #114 |

## References

| Artifact | Where |
|----------|-------|
| Blog entry | `blog/2026-06-02-mdp01-the-write-that-commits-before-returning.md` |
| ADR 0016 | `adr/0016-eigentrust-inapplicability-single-attestor-deployments.md` |
| ADR 0017 | `adr/0017-plain-ledger-entry-generic-concrete-subclass.md` |
| Spec | `specs/issue-114-lightweight-mode/2026-06-02-lightweight-outcome-tracking-design.md` |
| Design routing fix | `git show 5fc284a` (project) — also applied to parent/connectors/claudony/qhorus |
