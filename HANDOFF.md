# Handover — Slot 194

## Branch
- Ledger: `issue-208-class-mcpdomain-panache` (1 commit, not pushed)
- Engine: `issue-1119-class-mcpdomain` (1 commit, not pushed)
- IoT: `issue-105-class-mcpdomain` (1 commit, not pushed)
- Work: `issue-401-class-mcpdomain-panache` (1 commit, not pushed)
- Connectors: `issue-100-class-mcpdomain` (1 commit, not pushed)
- Clinical: `issue-171-mcpdomain-spi` (9 commits, not pushed)
- Neocortex (canonical): `panache-mcpdomain-port` (1 commit, not pushed)
- Ops: `panache-to-jpa` (1 commit, not pushed)
- Life: `panache-to-jpa` (1 commit, not pushed)

## Active Issue
`casehubio/clinical#171` — migrate clinical to @McpDomain SPI.
Queue position 4/24 (expanded with class-based + Panache batches).

## Session Summary

### Clinical #171 — @McpDomain SPI Migration (Completed)

Finished migration started in previous session. Created 2 more SPI interfaces
(EscalationPlanResource, TrialDashboardResource), bringing total to 15 @McpDomain
SPIs with 15 APT-generated REST resources. Deleted 14 hand-written resources total.
NarrativeResource stays hand-written (blocks dep boundary). DemoActionResource stays
by design.

CDI test failures investigated — root cause: engine #1049 moved CDI beans to plain
POJOs in runtime-core. Created `ClinicalTestSpiDefaults.java` with 15 @DefaultBean
producers as workaround. Remaining CDI failures are systemic (engine#1119).

### Engine #1119 — @DefaultBean Producers (Filed + Verified)

Filed casehubio/engine#1119. User fixed it in a parallel session. Verified engine
builds (excluding Spring modules which have a separate generator issue). Reinstalled
engine SNAPSHOT to slot .m2.

### Platform #341 — @McpDomain on Class (Reviewed)

User implemented in canonical platform. Reviewed code: clean implementation,
`DomainScanResult` gains `isInterface` flag, scanner handles both interface and
class sources identically. Suggested collision warning and @ContextParam test coverage.
User applied feedback.

### Ecosystem-Wide @McpDomain Class-Based Conversion (20 interfaces)

Converted interface+impl splits to class-based @McpDomain across 5 repos:
- Ledger: 4 interfaces → class-based, 3 Panache → JPA (862 tests pass)
- Engine: 5 interfaces → class-based (api + rest compile clean)
- IoT: 5 interfaces → class-based (compiles clean)
- Work: 5 interfaces → class-based + 21 entity Panache removals (partial)
- Connectors: 1 interface → class-based (compiles clean)

### Panache-to-JPA Porting (Partial)

Ported simple Panache entity removals across repos:
- Neocortex: 2 files (MemoryEntry, JpaMemoryStore)
- Ops: 5 entities + 8 callers
- Life: 3/4 entities (LifeCommitmentRecord deferred — 60+ callers)
- Eidos: false positive (stale worktree copies only)

Filed follow-up issues for remaining Panache work:
- casehubio/work#402 — stores/repos/MongoDB (~45 files)
- casehubio/life#119 — LifeCommitmentRecord (60+ callers)
- casehubio/clinical#172 — 18 entity files

### Issues Filed

| Issue | Repo | What |
|-------|------|------|
| #1119 | engine | @DefaultBean producers for RuntimeBeans SPIs |
| #341 | platform | @McpDomain on class (landed same session) |
| #402 | work | Complete Panache-to-JPA (~45 files) |
| #119 | life | LifeCommitmentRecord Panache (60+ callers) |
| #172 | clinical | Panache-to-JPA (18 files) |

## What's Next

1. **work#402** — complete Panache-to-JPA in work stores/repos/MongoDB (~45 files)
2. **life#119** — LifeCommitmentRecord Panache removal (60+ callers)
3. **clinical#172** — port 18 Panache entity files to JPA
4. **aml#130** — migrate aml to @McpDomain SPI (use class-based from the start)

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Push slot clones to `local` remote first, then push from canonical local repos to GitHub.
4. The JDK 26 surefire fix is in `webapp/pom.xml` — apply to other casehub webapps if they hit the same hang.
5. `casehub-platform-graphql-generator` is the APT processor — version managed by slot .m2 SNAPSHOT.
6. Engine Spring modules don't compile — skip with `-pl '!runtime-spring,...'` when installing.
