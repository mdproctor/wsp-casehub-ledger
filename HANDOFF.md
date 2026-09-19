# Handover — Slot 194

## Branch
AML on branch `issue-130-mcpdomain-spi` (11 commits, all tests pass, issue closed).
All other repos on main.

## Active Issue
`casehubio/soc#55` — migrate SOC to @McpDomain SPI.
Queue position 6/24.

## Session Summary

### aml#130 — Test Fixes (5 errors → 0)

Fixed all 5 test errors from the @McpDomain migration:

1. **JAX-RS path shadowing** — `AmlComplianceApi` had `basePath="/api"` with `RestPath="/investigations/{caseId}/compliance-evidence"`. The generated resource at `@Path("/api")` was shadowed by `GeneratedAmlInvestigationsResource` at `@Path("/api/investigations")` — JAX-RS dispatches to the more specific class-level match → 404. Fix: `basePath="/api/investigations"`, `RestPath="/{caseId}/compliance-evidence"`.

2. **`@RunOnVirtualThread` without `@Transactional`** — APT-generated endpoints use `@RunOnVirtualThread`. `AmlComplianceEvidenceService.findEvidence()` JPA queries returned empty without explicit transaction boundary. Fix: `@Transactional` on `findEvidence()` and `assembleEvidence()`.

3. **`@TestSecurity` missing** — `gdprDemoFlow_officerReview_erasure` called the erasure endpoint (`@RolesAllowed("aml-senior-compliance")`) without authentication. Fix: `@TestSecurity(user="compliance-officer", roles="aml-senior-compliance")`.

4. **CBR type mismatch** — `PlanCbrCase` (local record relocated from neocortex-memory) implements `CbrCase` but is NOT a `ResolvedCase`. Engine's `CbrRetrievalService` typeMap has `"plan" → ResolvedCase.class` — the store's `instanceof` filter rejected all `PlanCbrCase` entries. Fix: replaced `PlanCbrCase`/`PlanTrace` with `ResolvedCase`/`ResolutionStep` everywhere, deleted local classes.

5. **CBR store scope** — `AmlCaseProfileStoreObserver` stored at `Path.of("casehubio","aml")`, retrieval queries with `Path.root()`. `Path.root().isAncestorOf()` matches all paths so scope wasn't actually blocking, but aligned for consistency.

### clinical#171, #172 — Closed
Migration work was already on clinical main (`ce820c9`). Closed both issues.

### Queue Advanced
clinical#171 → aml#130 → soc#55 (current).

### Upstream Issues Filed
| # | Repo | Issue | Status |
|---|------|-------|--------|
| 1 | platform | #350 — APT path shadowing detection | Open |
| 2 | platform | #351 — @RunOnVirtualThread @Transactional | Open |
| 3 | engine | #1123 — CBR scope hardcoded Path.root() | Open |
| 4 | engine | #1124 — CBR CASE_LIFETIME timing | Open |
| 5 | neocortex | #368 — similarity normalization | Landed (fix in repo, but not the root cause — see #4 above) |

## Lessons for Next Migrations

1. **basePath conflicts** — when two `@McpDomain` classes share a path prefix (e.g., `/api` and `/api/investigations`), the more specific class shadows the less specific one. Set basePath to the longest common prefix shared with sibling resources.

2. **@Transactional** — any service called from a generated endpoint that uses JPA needs explicit `@Transactional`. The APT generator adds `@RunOnVirtualThread` which doesn't auto-wrap transactions.

3. **CBR case types** — if the app uses a local `CbrCase` subclass instead of `ResolvedCase`, it must be registered via `CbrCaseTypeRegistration` or use `ResolvedCase` directly.

4. **@RolesAllowed propagation** — `@RolesAllowed` propagates from `@McpDomain` methods to generated REST endpoints. Tests calling secured endpoints need `@TestSecurity`.

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` — all slot repos installed, non-slot repos resolve from host .m2 fallback.
3. `casehub-platform-graphql-generator` is the APT processor.
