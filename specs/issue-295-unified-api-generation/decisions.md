# Decisions — Unified API Generation Migration (ledger#207)

## D1: Annotation model — platform-specific annotations only

**Choice:** Use `@PlatformQuery`, `@PlatformMutation`, `@PathParam` (from `io.casehub.platform.api.mcp`), and `@RestMethod` — never JAX-RS annotations on SPI interfaces.
**Alternatives:**
- JAX-RS annotations directly on SPI (`@GET`, `@Path`, `@PathParam`) — causes Quarkus REST server resource registration conflict (GE-20260612-4f9a47)
- Hybrid (platform query/mutation + JAX-RS `@PathParam`) — mixes two annotation vocabularies for no gain
**Rationale:** JAX-RS `@Path` on an interface causes Quarkus REST to register the interface itself as a server resource, conflicting with the generated resource class. Platform annotations are invisible to Quarkus REST's resource scanner.
**Trade-offs:** SPI authors must learn platform annotations. Non-standard, but the generator maps them 1:1 to JAX-RS in generated code.
**Sources:** GE-20260612-4f9a47, platform `io.casehub.platform.api.mcp` package, `GraphQLResolverProcessor.java`
**Exploration:** quick — constraint, not a choice
**Status:** captured

## D2: REST path style — generated flat kebab-case

**Choice:** Accept generator-derived paths from kebab-cased method names (e.g. `getEntry` → `/get-entry/{id}`).
**Alternatives:**
- Add `@RestPath` annotation to platform for explicit path control — more work, not needed for pre-release
- Extend `@RestMethod` with `path` attribute — couples path semantics to verb override annotation
**Rationale:** Pre-release with no consumers. Flat paths follow the platform generator convention. No platform changes needed.
**Trade-offs:** Current nested paths (`/entries/{id}/attestations`) become flat (`/list-attestations/{entryId}`). Breaking change, acceptable pre-release.
**Sources:** `GraphQLResolverProcessor.generateRestMethod()`, issue #207
**Exploration:** quick
**Status:** captured

## D3: Domain structure — four SPI interfaces by sub-domain

**Choice:** Four interfaces with hierarchical `@McpDomain`: `LedgerEntryApi` (`ledger/entries`), `LedgerAttestationApi` (`ledger/attestations`), `LedgerVerificationApi` (`ledger/verification`), `LedgerTrustApi` (`ledger/trust`).
**Alternatives:**
- Single `LedgerApi` interface with `@McpDomain("ledger")` — 11 methods is too coarse for MCP progressive discovery
- Two interfaces (queries + mutations) — doesn't map to domain concepts, generator handles mixed interfaces fine
**Rationale:** Matches current REST resource grouping. Hierarchical domains enable MCP progressive discovery at the sub-domain level.
**Trade-offs:** Four interfaces = four generated REST resources + four generated GraphQL resolver classes. More files, but each is focused and small.
**Sources:** Current REST resources (`LedgerEntryResource`, `AttestationResource`, `MerkleVerificationResource`, `TrustScoreResource`), platform `CallbackApi`/`NotificationPreferenceApi` patterns
**Exploration:** quick
**Status:** captured

## D4: SPI interface location — `api/` module

**Choice:** SPI interfaces and return type records live in `api/src/main/java/io/casehub/ledger/api/`.
**Alternatives:**
- New `spi/` module — adds a module for no reason; `api/` already holds all SPIs
- In `runtime/` — would force generated code in `rest/`/`graphql/` to depend on runtime, violating GE-20260816-d18a02
**Rationale:** `api/` is the public contract module. No JPA deps, no framework deps. Generated code in `rest/`/`graphql/` depends only on `api/`.
**Trade-offs:** None — this is the established pattern.
**Sources:** GE-20260816-d18a02, existing `api/spi/` package structure
**Exploration:** quick
**Status:** captured

## D5: Service implementations — `runtime/` module

**Choice:** Four `@ApplicationScoped` CDI beans in `runtime/` implementing the SPI interfaces, composing existing services.
**Alternatives:**
- In `rest/` or `graphql/` — would require runtime deps in those modules (violates GE-20260816-d18a02)
- Default methods on the SPI interface — not CDI-injectable
**Rationale:** Runtime beans inject existing repositories and services (`LedgerEntryRepository`, `TrustScoreSource`, `LedgerVerificationService`). CDI resolves the SPI interface to the runtime implementation at runtime. Generated code in `rest/`/`graphql/` needs only `api/` at compile time.
**Trade-offs:** Service implementations must be provided by the consumer (by depending on `casehub-ledger` runtime). This is already the pattern.
**Sources:** Platform `CallbackService`, `NotificationPreferenceService` pattern
**Exploration:** quick
**Status:** captured

## D6: APT configuration — separate in `rest/` and `graphql/`

**Choice:** Both modules add `graphql-generator` as `annotationProcessorPath`. `rest/` uses `-AgenerateGraphQL=false`, `graphql/` uses `-AgenerateRest=false`. Both use `-AdomainFilter=ledger/entries,ledger/attestations,ledger/verification,ledger/trust`.
**Alternatives:**
- Merge `rest/` and `graphql/` into one module — breaks opt-in model; consumers should choose API surface independently
- APT in `api/` — would add JAX-RS + SmallRye deps to the lightweight API module
**Rationale:** Preserves current opt-in model. Consumers add `casehub-ledger-rest` for REST, `casehub-ledger-graphql` for GraphQL, or both. Domain filtering (GE-20260914-3854b8) prevents generating endpoints for other modules' domains.
**Trade-offs:** APT config duplicated across two pom.xml files. Manageable — same domainFilter, different generate flags.
**Sources:** GE-20260914-3854b8, platform `callback/pom.xml` and `notifications/pom.xml` patterns
**Exploration:** quick
**Status:** captured

---

# Engine Migration Decisions (casehubio/engine#1095)

## D7: Engine domain structure — five SPI interfaces by function

**Choice:** Five interfaces: `engine/cases` (list, get, start, context, plan-items, goals), `engine/control` (suspend, resume, cancel, signal), `engine/definitions` (list, get-by-key), `engine/events` (event log), `engine/plan` (model, definitions, decomposition, dag, dag-result, state).
**Alternatives:**
- 3 interfaces (cases+control+signal+events, definitions, plan) — 14-method `engine/cases` is too coarse for MCP discovery
- 8 interfaces (1:1 with REST resources) — single-method interfaces (`engine/signals`, `engine/events`) generate 3 boilerplate classes each for no MCP benefit; signal and control are the same domain action ("act on a running case")
**Rationale:** Groups by what the caller is thinking about. Signal and control are both case-scoped state mutations — splitting them is a REST path artifact, not a domain boundary. Single-method domains are MCP-wasteful.
**Trade-offs:** `engine/events` is still single-method but conceptually distinct (audit trail, not case state).
**Sources:** Ledger D3 pattern, engine REST resource structure, MCP progressive discovery model
**Exploration:** quick
**Status:** captured

## D8: SPI implementation location — engine rest/ module

**Choice:** SPI implementation beans live in `rest/service/` alongside the existing `CaseService`.
**Alternatives:**
- New `engine-api-impl/` module — clean layering but adds a module for no gain; CaseService already has all deps
- Engine `runtime/` — conflates REST-shaped service methods with engine core
**Rationale:** CaseService already injects `CaseHubRuntime`, `CaseInstanceRepository`, `CaseDefinitionRegistry`, `ExpressionEngineRegistry`, `AccessControlProvider`. SPI impls delegate to CaseService + existing engine services. No new modules needed.
**Trade-offs:** SPI impls live in `rest/`, not a framework-neutral module. Acceptable because the engine `rest/` module is already the API surface layer.
**Sources:** Engine CaseService, ledger D5 pattern
**Exploration:** quick
**Status:** captured

## D9: Streaming endpoints — all migrated to @PlatformStream

**Choice:** All 4 streaming endpoints (2 SSE REST + 2 GraphQL subscriptions) migrate to `@PlatformStream` on SPI interfaces. Generator produces SSE + @Subscription from the same method.
**Alternatives:**
- Keep hand-written — contradicts the strategic goal of unified generation after investing in @PlatformStream support (platform #300)
**Rationale:** The generator was specifically enhanced with @PlatformStream for this use case. The SPI method delegates to the broadcaster (`broadcaster.stream(caseId) → Multi<T>`), which is the same thin-delegation pattern as every other SPI method. If the generator needs improvements for edge cases, fix the generator.
**Trade-offs:** None significant — broadcaster infrastructure stays, only the resource/resolver class is replaced.
**Sources:** Platform #300 (@PlatformStream support), GE-20260804-8b0fd6 (BroadcastProcessor pattern)
**Exploration:** quick
**Status:** captured
