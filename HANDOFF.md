# Handover — Slot 194

## Branch
AML on branch `issue-130-mcpdomain-spi` (7 commits). All other repos on main.
Engine has 1 uncommitted-to-origin commit on main (TestWorkerProvisioner + TestCaseInstanceRepository).
Qhorus on branch `issue-42-ux-overhaul` with 3 fix commits (V55/V56 migration fixes).

## Active Issue
`casehubio/aml#130` — migrate aml to @McpDomain SPI.
Queue position 4/24.

## Session Summary

### aml#130 — @McpDomain Migration (Complete, Tests Partially Fixed)

Migrated all 37 AML REST endpoints from 12 hand-written JAX-RS resources to 9 `@McpDomain` API classes with APT-generated REST resources.

**@McpDomain classes created** (`io.casehub.aml.service`):
- `AmlInvestigationApi` (7 ops) — list, query, prior-context, flow, findings, gates, routing
- `AmlEngineApi` (6 ops) — Layer 5/6/9 start, get, outcome
- `AmlOversightControlApi` (2 ops) — suspend, resume (`@RolesAllowed`)
- `AmlWorkerApi` (2 ops) — list tasks, respond
- `AmlAuditApi` (3 ops) — audit trail, inclusion proof, provenance
- `AmlComplianceApi` (1 op) — compliance evidence (`@PermitAll` implied)
- `AmlErasureApi` (3 ops) — actor, entity, cross-tenant erasure (`@RolesAllowed`)
- `AmlMetricsApi` (7 ops) — throughput, trust, gates, history, interventions, SAR quality, CBR bootstrap
- `AmlSimulationApi` (6 ops) — seed, reset, investigate, CBR seed/clear (`@IfBuildProperty` gated)

**Also done:**
- Converted `InvestigationSummaryRepository` from Panache to plain JPA (EntityManager + @NamedQuery)
- Created local `PlanTrace` + `PlanCbrCase` in `aml.cbr` (relocated from neocortex-memory)
- Fixed imports for ledger-core package moves (InclusionProof, ContentSanitiser)
- Configured APT processor (maven-compiler-plugin + casehub-platform-graphql-generator)
- Split secured endpoints (erasure, oversight-control) into separate @McpDomain classes for RBAC isolation
- Created `TestCurrentPrincipal` (`@Alternative @Priority(200) @ApplicationScoped`) — provides default tenancy outside HTTP request scope
- Fixed `Path.root()` → `Path.of("casehubio", "aml")` for CBR store scope
- Added `@TestSecurity` to compliance and oversight tests

### Engine Fixes (1 commit on slot main, not pushed)
- `TestCaseInstanceRepository`: added no-args constructor + `@Inject` on injection constructor
- `TestWorkerProvisioner`: `@Alternative @Priority(1)` auto-completes provisioned workers via event bus

### Qhorus Fixes (3 commits on `issue-42-ux-overhaul`)
- V55: removed partial index WHERE clause (H2 incompatible), added `corrects_message_id` to `message_ledger_entry`
- V56: removed FK to `ledger_entry` (Flyway version ordering — V56 runs before V1000)

### Slot .m2 State
Nuked and rebuilt from slot repos + host .m2 fallback. All slot repos (platform, engine, work, ledger, qhorus) installed from their slot clones. Non-slot repos (blocks, connectors, pages, neocortex) resolve from host .m2.

### Test Results: 451 tests, 0 failures, ~5 errors

All previously-failing tests fixed except 5 remaining errors (4 unique test methods):

## What's Next — Fix Remaining 5 Test Errors

### 1. `AmlLayer7ResourceTest` (3 methods fail)

**Root cause:** The compliance evidence endpoint queries ledger entries via `LedgerEntryRepository.findBySubjectId()` which needs a JPA transaction context. The generated REST resource runs on virtual threads without automatic transaction wrapping.

**Fix approach:**
- Add `@Transactional` to `AmlComplianceEvidenceService.findEvidence()` method
- Or add `@jakarta.transaction.Transactional` to the `AmlComplianceApi.getComplianceEvidence()` method (check if APT propagates it to the generated resource)
- The `gdprDemoFlow_officerReview_erasure` test also calls erasure endpoints — may need `@TestSecurity` for the erasure calls within the test flow

**Files:** `app/src/main/java/io/casehub/aml/compliance/AmlComplianceEvidenceService.java`, `app/src/main/java/io/casehub/aml/service/AmlComplianceApi.java`

### 2. `CbrActivationIntegrationTest.learningMode_belowThreshold_advisorWritesActiveFalse`

**Root cause:** The test asserts `active=false` on the CBR advisory but gets `active=true`. The advisory activation threshold is resolved via `PreferenceProvider` which may return a different default now. Check `AmlCbrPolicyKeys.ACTIVATION_THRESHOLD` default value vs. the number of seeded cases.

**Files:** `app/src/test/java/io/casehub/aml/cbr/CbrActivationIntegrationTest.java`, `app/src/main/java/io/casehub/aml/cbr/AmlCbrPolicyKeys.java`

### 3. `SarNarrativeSeedingIntegrationTest.seededInvestigation_narrativeSeededTrue`

**Root cause:** Same pattern as CBR profile — the narrative seed flag is set by an async observer after investigation completion. The test may need an Awaitility wait for the narrative seed flag to propagate.

**Files:** `app/src/test/java/io/casehub/aml/cbr/SarNarrativeSeedingIntegrationTest.java`

### 4. `AmlCbrRetrieveTest` (1 method, 10s timeout)

**Root cause:** The test stores a past CBR case with `Path.of("casehubio", "aml")` scope but queries with `Path.root()` scope. The scopes need to match. Check the query scope in the test.

**Files:** `app/src/test/java/io/casehub/aml/cbr/AmlCbrRetrieveTest.java`

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` was nuked and rebuilt — all slot repos installed, non-slot repos resolve from host .m2 fallback.
3. Push slot clones to `local` remote first, then push from canonical local repos to GitHub.
4. Engine has unpushed commits — push to local remote when done.
5. `casehub-platform-graphql-generator` is the APT processor — version managed by slot .m2 SNAPSHOT.
6. The `@RolesAllowed` annotation propagates from @McpDomain methods to generated REST endpoints. `@PermitAll` does NOT propagate — split secured and unsecured endpoints into separate @McpDomain classes instead.
7. `TestCurrentPrincipal` (`@Priority(200)`) overrides `SecurityIdentityCurrentPrincipal` (`@Priority(100)`) in tests — provides `DEFAULT_TENANT_ID` tenancy outside request scope.
