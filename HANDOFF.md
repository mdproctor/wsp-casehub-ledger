# CaseHub Ledger — Session Handover
**Date:** 2026-06-03

## Current State

All S-scale issues closed. Both repos on `main`, clean.
Local `main` in sync with upstream/main (reset to upstream earlier this session).
Pause stack empty.

## What Landed This Session

- **#99**: `LedgerProcessor.registerMigrationResources()` — `NativeImageResourcePatternsBuildItem` for `db/ledger/migration/*.sql`. Resolves known gap in PP-20260528-flyway-ext-reg. `parent#150` filed to update protocol; `qhorus#241` filed to remove redundant ledger glob from `QhorusProcessor`.
- **#104**: `InMemoryAgentSigner` in persistence-memory — `@Alternative @Priority(1)`, `ConcurrentHashMap<String,KeyPair>`, `register(actorId, keyPair)` + `clear()`.
- **#105**: `InMemoryReactiveAttestationTest` + `ReactiveProfile` in persistence-memory — verifies reactive `saveAttestation()` delegation path. `qhorus#242` filed for consumer-side implementation.
- **#111**: `ActorIdentityValidationEnricherObserverTest` in `io.casehub.ledger.runtime.service.identity` — calls `onKeyRotated()` directly (package-private). `platform#62` filed for ScimActorDIDProvider `requireHttps` + authToken fixes.
- **#117**: `InMemoryLedgerEntryRepository.clear()` — Javadoc documenting session-boundary lifecycle use; two tests verifying all state reset and sequence counter restart.
- **#119**: `EigenTrustStartupValidationIT` — CDI wiring + config resolution; startup log capture not viable (JBoss LogManager handler chain reset during bootstrap — garden GE-20260603-83883c).
- **Dangling branches**: `issue-114-lightweight-mode` cleared from pause stack (already had EPIC-CLOSED); `issue-105-106-reactive-parity` properly closed (local main reset to upstream/main).
- **CLAUDE.md**: `InMemoryAgentSigner` added to persistence-memory section; `AgentKeyProvider`→`AgentSigner` stale refs fixed.
- **Blog**: `2026-06-03-mdp01-the-log-that-wouldnt-be-caught.md`
- **Garden**: GE-20260603-83883c (JBoss LogManager handler chain reset during Quarkus bootstrap)
- **ledger#121**: filed — CLAUDE.md had stale `AgentKeyProvider`/`ConfiguredAgentKeyProvider` refs (now fixed).

## Immediate Next Step

Update the quarkmind HANDOFF.md to clear the ledger#114 blocking dependency (OutcomeRecorder + TrustWeightedAgentStrategy now available — this was deferred from the previous session).

## Cross-Module

**We unblock:**
- `quarkmind` — ledger#114 shipped; `OutcomeRecorder.record()` + trust routing available. QuarkMind can wire in with `casehub.ledger.trust-score.routing-enabled=true`.
- `qhorus` — `saveAttestation()` ledger-side complete; qhorus#242 filed for consumer-side impl.

## What's Left

- Update quarkmind HANDOFF.md (prior-session deferred) · XS · Low
- Backup branch `backup/pre-squash-issue-114-lightweight-mode-20260602` — delete after 2026-06-16 · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #115 | Incremental per-actor trust recomputation | L | High | Needs casehub-engine `TrustScoreCache.refreshForActor` companion |
| #116 | JpaLedgerEntryRepository.save() sequenceNumber gap | M | Med | Blocks JPA consumers of OutcomeRecorder |
| #119 | EigenTrustStartupValidationIT | S | Med | ✅ Done this session |
| #120 | ARC42STORIES.MD Foundation tier | L | High | Dedicated session; no casehubio deps |
| — | QuarkMind: wire OutcomeRecorder into game loop | M | Low | Unblocked by #114; quarkmind-side change |

## References

| Artifact | Where |
|----------|-------|
| Blog entry | `blog/2026-06-03-mdp01-the-log-that-wouldnt-be-caught.md` |
| Garden entry | GE-20260603-83883c (jvm/) |
| Peer issues | parent#150, qhorus#241, qhorus#242, platform#62, ledger#121 |
