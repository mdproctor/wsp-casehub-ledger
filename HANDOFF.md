# Handover — Slot 194

## Branch
All repos on main. Feature branches still exist (not deleted, not pushed).

## Active Issue
`casehubio/clinical#171` — migrate clinical to @McpDomain SPI.
Queue position 4/24. Sub-tasks remain (work#402, life#119).

## Session Summary

### Clinical #172 — Panache-to-JPA Port (Completed)

Ported all 18 Panache files in clinical to plain JPA EntityManager:
- 15 entity files: removed `extends PanacheEntityBase`, added 24 `@NamedQuery`
  annotations, deleted all static query methods
- 3 CBR files: replaced `PanacheEntityResolver` with `JpaEntityResolver`
- Created `TenantEntityLookup` utility for the common `findByIdForTenant`
  pattern (32 call sites across services, resources, demo code)
- Updated ~60 caller files (services, APIs, CBR, demo, scenario)
- Updated ~50 test files (constructor signatures, Panache calls)
- Production code compiles clean. Unit tests pass.
- `@QuarkusTest` integration tests have pre-existing CDI errors from engine#1119.
  Total: 134 files changed, 1428 insertions, 786 deletions.

### Ecosystem-Wide Branch Merge (8 repos)

Merged all feature branches to main across the slot:
- Ledger: `issue-208-class-mcpdomain-panache` → main (ff)
- Engine: `issue-1119-class-mcpdomain` → main (rebase + ff, 7 canonical catches up)
- Work: `issue-401-class-mcpdomain-panache` → main (rebase + ff, 2 canonical catches up)
- IoT: `issue-105-class-mcpdomain` → main (ff)
- Connectors: `issue-100-class-mcpdomain` → main (ff)
- Ops: `panache-to-jpa` → main (ff)
- Life: `panache-to-jpa` → main (ff)
- Clinical: `issue-171-mcpdomain-spi` → main (squash 12 → 1)

Nothing pushed to `local` remotes yet.

## What's Next

1. **Push to local remotes** — all 8 repos have unpushed main commits
2. **work#402** — complete Panache-to-JPA in work stores/repos/MongoDB (~45 files)
3. **life#119** — LifeCommitmentRecord Panache removal (60+ callers)
4. **aml#130** — migrate aml to @McpDomain SPI (use class-based from the start)
5. **Remaining queue** — 20 items at position 4/24

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Push slot clones to `local` remote first, then push from canonical local repos to GitHub.
4. The JDK 26 surefire fix is in `webapp/pom.xml` — apply to other casehub webapps if they hit the same hang.
5. `casehub-platform-graphql-generator` is the APT processor — version managed by slot .m2 SNAPSHOT.
6. Engine Spring modules don't compile — skip with `-pl '!runtime-spring,...'` when installing.
7. Clinical `@QuarkusTest` integration tests broken by engine#1119 CDI ambiguity — pre-existing, not from Panache port.
