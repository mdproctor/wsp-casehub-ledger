# Handover — Slot 194

## Branch
All repos on main. Feature branch `issue-130-mcpdomain-spi` exists in aml (empty, no changes).

## Active Issue
`casehubio/aml#130` — migrate aml to @McpDomain SPI.
Queue position 4/24. Sub-tasks under clinical#171: work#402 blocked, life#119 done, clinical#172 done.

## Session Summary

### life#119 — Panache-to-JPA Port (Completed)

Ported all 4 life entities from Panache active-record to plain JPA EntityManager:
- **LifeCommitmentRecord**: removed `extends PanacheEntityBase`, 5 `@NamedQuery`, removed 3 static finders
- **LifeCaseTracker**: 3 `@NamedQuery`, removed 3 static finders
- **ExternalActor**: 2 `@NamedQuery` (findNotErased, findAll)
- **LifeTaskContext**: 2 `@NamedQuery` (findByExternalActorId, countByExternalActorId)
- ~20 production files: injected EntityManager, replaced Panache with em.find/em.persist/named queries
- ~27 test files: @Inject EntityManager for @QuarkusTest, @Mock EntityManager for unit tests
- WorkItemEntity/WorkItemTemplate calls left unchanged (casehub-work scope, still Panache)
- Total: 57 files changed, 422 insertions, 215 deletions
- Squashed to single commit `5d56332`, merged ff to main, pushed to local
- GitHub issue closed

### Local Remote Push (7/8 repos)

Pushed main to local remotes for: ledger, engine, iot, connectors, ops, life, clinical.
Work repo skipped — active session in canonical clone (unstaged changes to casehub-platform-testing dep).

## What's Next

1. **aml#130** — migrate aml to @McpDomain SPI (37 endpoints, 12 resources → 7 @McpDomain classes)
   - Survey complete, grouping planned (see below)
   - Branch `issue-130-mcpdomain-spi` exists (empty)
   - WebSocket endpoint (AmlPushEndpoint) stays as-is
2. **work#402** — complete Panache-to-JPA in work stores/repos/MongoDB (~45 files) — blocked by active session on canonical work repo
3. **Remaining queue** — 19 items at position 4/24

### aml#130 Grouping Plan

| @McpDomain class | value | Endpoints | Source resources |
|---|---|---|---|
| `AmlInvestigationApi` | `aml/investigations` | 7 | AmlInvestigationResource, AmlInvestigationQueryResource |
| `AmlEngineApi` | `aml/engine` | 8 | Layer5, Layer6, Layer9 |
| `AmlWorkerApi` | `aml/workers` | 2 | AmlWorkerTaskResource |
| `AmlAuditApi` | `aml/audit` | 3 | AuditTrail, Provenance |
| `AmlComplianceApi` | `aml/compliance` | 4 | Layer7 (evidence + 3 erasure) |
| `AmlMetricsApi` | `aml/metrics` | 7 | Metrics, SarQuality, CBR bootstrap |
| `AmlSimulationApi` | `aml/simulation` | 6 | Simulation (dev-only, @IfBuildProperty) |

Pattern: class-based @McpDomain (not interface), @ApplicationScoped, @PlatformQuery/@PlatformMutation methods.
Reference impl: ledger's DefaultLedgerEntryApi.java.

### aml#130 Implementation Notes
- AmlInvestigationQueryResource uses Panache Page — convert to JPA setFirstResult/setMaxResults
- AmlWorkerTaskResource has inline logic — extract to service
- AmlLayer6Resource GET has inline logic — extract to service
- AmlCbrResource has inline EntityManager JPQL — keep as-is or extract
- Slot 181 has active AML work (issue-469-dual-framework-core-extraction) — different scope but same repo, confirmed safe to proceed

### Pre-existing Issues (other repos, not blocking)
- Life: upstream API compile errors (humanTask(), WorkItem→WorkItemEntity) — not from Panache port
- Clinical: `@QuarkusTest` integration tests broken by engine#1119 CDI ambiguity

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Push slot clones to `local` remote first, then push from canonical local repos to GitHub.
4. The JDK 26 surefire fix is in `webapp/pom.xml` — apply to other casehub webapps if they hit the same hang.
5. `casehub-platform-graphql-generator` is the APT processor — version managed by slot .m2 SNAPSHOT.
6. Engine Spring modules don't compile — skip with `-pl '!runtime-spring,...'` when installing.
