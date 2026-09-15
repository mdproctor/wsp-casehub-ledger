## D1: Module placement for SPI interfaces and view records

**Choice:** `webapp-api/` module — webapp-specific SPIs alongside existing webapp types; `api/` stays foundation-only
**Alternatives:**
- `api/` module — consistent with platform-api pattern, but blurs foundation/webapp boundary (original choice, revised in R1)
- New `iot-spi/` module — clean separation but over-engineering, adds Maven module overhead
**Rationale:** The SPI interfaces (IoTDeviceApi, IoTCaseApi, etc.) define the webapp's generated REST/GraphQL/MCP surface — a webapp concern, not a foundation concern. ARC42STORIES boundary rules are clear: "casehub-iot owns: device type vocabulary, provider SPI, event model" — the webapp API shape is not in that list. webapp-api already has Jandex configured (jandex-maven-plugin in pom.xml) and depends on platform-api transitively (via casehub-iot-api), so @McpDomain is available. The cross-repo pattern confirms this: engine's @McpDomain resolvers live in engine runtime modules, not engine-api; platform-level APIs (DeliveryChannelApi, AclApi) live in platform-api because they ARE platform concerns. Analogously, webapp APIs belong in webapp-api because they ARE webapp concerns. api/ remains focused on device types, provider SPI, and event model — the foundation layer consumed by casehub-life and future apps.
**Trade-offs:** SPI interfaces are in the webapp tier, not the foundation tier. If a future consumer needs to depend on the SPI (e.g., an ops module wanting the same generated surface), it must depend on webapp-api rather than api/. Currently no such consumer exists.
**Sources:** GE-20260914-f53be7, webapp-api/pom.xml (Jandex already configured), ARC42STORIES.MD §3 boundary rules, platform @McpDomain usage analysis (DeliveryChannelApi in platform-api, CaseQueryResolver in engine runtime)
**Exploration:** quick
**Status:** revised — moved from api/ to webapp-api/ per R1-02 boundary analysis. webapp-api already has Jandex; api/ stays foundation-only.

## D2: Domain groupings — 5 SPI interfaces

**Choice:** 5 interfaces organized by caller intent:
1. `IoTDeviceApi` (`iot/devices`) — 4 methods (list, get, command, history)
2. `IoTSituationApi` (`iot/situations`) — 7 methods (definitions CRUD, active, suggestions, dismissal)
3. `IoTSuppressionApi` (`iot/situations/suppressions`) — 3 methods (suppression history, override, stats)
4. `IoTCaseApi` (`iot/cases`) — 6 methods (cases + resolution queue merged)
5. `IoTOperationsApi` (`iot/ops`) — 7 methods (providers, bridge, health)
**Alternatives:**
- 4 interfaces (original — suppression merged into IoTSituationApi at 10 methods) — spans too many sub-concerns, MCP tool surface too broad for a single domain
- 6 interfaces (separate resolution queue + separate suppression) — resolution queue separation not justified by domain analysis
**Rationale:** IoTSituationApi at 10 methods spanning definitions CRUD, operational awareness, and false-positive management is too broad. Suppression management (history, override, stats) is a distinct operational concern from situation detection. The hierarchical @McpDomain naming (`iot/situations/suppressions`) explicitly supports sub-domains — an LLM agent querying active situations shouldn't see suppression management tools in its tool set. Cases and resolution queue remain merged: CaseResource and ResolutionQueueResource share 4 injected services (CaseInstanceCache, CaseDefinitionRegistry, IoTCbrRetrievalService, CurrentPrincipal), confirming tight domain coupling.
**Trade-offs:** 5 interfaces instead of 4 adds one more SPI. The suppression sub-domain is small (3 methods) but cohesive.
**Sources:** SituationResource.java (10 methods spanning 4 concerns), ResolutionQueueResource.java (shares 4 services with CaseResource)
**Exploration:** quick
**Status:** revised — split IoTSuppressionApi from IoTSituationApi per R1-06; removed "stream" from IoTDeviceApi per R2-02 (D4 keeps SSE hand-written).

## D3: View record strategy for existing webapp-api DTOs

**Choice:** Evolve existing webapp-api DTOs into the view records in-place. SPI methods return these types directly. One set of types, one source of truth.
**Alternatives:**
- New view records alongside existing DTOs (original choice) — creates permanent type duplication with no forcing function for cleanup
- Reuse webapp-api DTOs from SPI without modification — may expose internal fields not appropriate for generated API surface
**Rationale:** If view records ARE the single source of truth for API shape, then creating a second set of types alongside existing DTOs is a backward-compatibility shim. The design philosophy is explicit: "bad design unless explicitly asked to preserve backward compatibility." Import changes from flattening nested inner records are mechanical. With D1 revised (SPIs in webapp-api/), the view records live in the same module as the existing DTOs — making in-place evolution natural. Some DTOs may need field curation (adding/removing fields for the generated surface), but the result is one canonical type per API shape, not two.
**Trade-offs:** Requires import changes on existing consumers of modified DTOs. Nested inner records need flattening. Both are mechanical migrations. The alternative (permanent duplication) is worse.
**Depends on:** D1 (module placement determines where view records live)
**Sources:** engine api/view/ records, work api/view/ records, webapp-api DeviceResponse/CommandRequest/etc.
**Exploration:** quick
**Status:** revised — evolve existing DTOs in-place per R1-09 design philosophy analysis. Eliminates type duplication.

## D4: SSE stream handling

**Choice:** Keep DeviceSseResource hand-written — SSE endpoints stay outside the generated surface, as issue #105 explicitly directs.
**Alternatives:**
- Include in IoTDeviceApi as `@PlatformStream` method (original choice) — directly contradicts issue #105's pre-migration guidance, and platform#305 (@PlatformStream → SSE generation) is still open
**Rationale:** Issue #105's body explicitly states under "Pre-migration cleanup": "SSE/streaming endpoints stay hand-written (keep `Multi<T>` endpoints as-is)." The existing DeviceSseResource has complex application logic (initial snapshot + merged broadcast updates, tenancy filtering, JSON serialization with operation types) that doesn't fit a generated endpoint. Platform #305 (`@PlatformStream → SSE + GraphQL subscriptions`) is still open — the generator doesn't fully support SSE generation yet. When #305 lands, this decision can be revisited.
**Trade-offs:** The device API surface is split: generated REST/GraphQL/MCP for request-response methods, hand-written SSE for streaming. Consumers need to know both. This is the documented intent of issue #105.
**Sources:** Issue #105 body ("SSE/streaming endpoints stay hand-written"), DeviceSseResource.java (snapshot + broadcast pattern), platform #300/#305 (open — SSE generation not complete)
**Exploration:** quick
**Status:** revised — SSE stays hand-written per issue #105's explicit guidance. Original decision contradicted the authorizing issue.

## D5: Placeholder/unimplemented endpoint strategy

**Choice:** Default methods + `NotImplementedException` + exception mapper. SPI uses Java `default` methods for unimplemented operations that throw `NotImplementedException`. A JAX-RS exception mapper converts to HTTP 501.
**Alternatives:**
- `@Unimplemented` annotation on SPI methods — generator recognizes and produces 501 directly. But mixes lifecycle state into the contract (layer violation), requires generator changes, and needs two changes to "turn on" a method (remove annotation + add impl override)
- Impl bean throws manually — pure, but requires boilerplate per unimplemented method
**Rationale:** Java already has the right mechanism: abstract = must implement, default = can defer. This uses standard Java patterns (default methods, exceptions, exception mappers) with zero generator changes. The SPI remains a pure contract. Only one change to "turn on" a method (add override in impl bean). Works across all three surfaces (REST, GraphQL, MCP) via standard error handling.
**Trade-offs:** The generated OpenAPI spec can't distinguish unimplemented from implemented endpoints. Acceptable because these are internal APIs and 501 at runtime is sufficient. NotImplementedException does not yet exist in platform-api — if platform#300 doesn't deliver it, create it locally in webapp-api as a temporary location.
**Depends on:** D2 (domain groupings determine which methods are abstract vs default)
**Implementation scope:**
1. `NotImplementedException` → `platform-api` (casehubio/platform#300, in scope for slot). **Fallback:** create in webapp-api if platform doesn't deliver.
2. `NotImplementedExceptionMapper` → platform runtime support (one registration, all domains)
3. IoT SPI uses default methods for `listCases`, `getCase`, `listActive`
**Sources:** First-principles analysis of contract vs lifecycle separation, Java default method semantics, JAX-RS exception mapper pattern, CaseResource.java (list() returns List.of() with TODO, get() throws NotFoundException with TODO — confirmed placeholder), SituationResource.java (listActive() returns List.of() with TODO — confirmed placeholder)
**Exploration:** deep-analysis
**Status:** captured — unchanged, with explicit fallback for NotImplementedException dependency.

## D6: Annotation model — established platform annotations

**Choice:** Use the established platform annotation model: `@PlatformQuery("description")` / `@PlatformMutation("description")` as multi-transport semantic markers. `@RestMethod(HttpMethod.PUT/DELETE)` for non-default HTTP verbs. `@RestPath("/{id}")` for REST path overrides. `@PathParam`, `@ContextParam`, `@PaginatedResponse`, `@RestStatus` as parameter/response modifiers. Same model as engine and work SPIs.
**Alternatives:**
- JAX-RS annotations (`@GET`, `@POST`, `@Path`) directly on SPI interfaces — blocked by Quarkus scanning: Quarkus discovers and processes JAX-RS annotations on interfaces and their implementing CDI beans, causing path conflicts with the generated resource classes. This is the specific known limitation that prompted creating the platform annotation vocabulary.
- Issue #105's annotation model (JAX-RS + `@Description`) — aspirational but not feasible without a Quarkus extension to suppress JAX-RS scanning on `@McpDomain` interfaces. Not in scope.
**Rationale:** Platform annotations exist for a concrete technical reason: Quarkus always finds and processes JAX-RS annotations. `@PlatformQuery`/`@PlatformMutation` are invisible to the runtime scanner — only the generator reads them. This separation is documented in platform#295 decisions D1 (multi-transport semantic markers), D3 (zero JAX-RS dependency in platform-api), and D8 (platform HttpMethod enum). The established model works, is tested across 3 prior migrations, and the Spring generators in slot 192 also use it via `McpDomainJandexScanner`.
**Trade-offs:** Two annotations for non-default HTTP verbs (`@PlatformMutation + @RestMethod(PUT)`). This is the minority case — most mutations are POST, most queries are GET.
**Sources:** platform#295 decisions D1/D3/D8, Quarkus JAX-RS scanning limitation (known issue), GE-20260612-4f9a47 (class-level @Path conflict), engine WorkItemApi.java (established pattern), `McpDomainJandexScanner.java` (shared scanner in generator-common)
**Exploration:** deep-analysis (attempted JAX-RS redesign, discovered blocking constraint, reverted to established model)
**Status:** revised — reverted from JAX-RS proposal to established platform annotations after confirming Quarkus scanning limitation

## D7: MCP surface migration strategy

**Choice:** Coexistence then replacement. The existing curated MCP module (`casehub-iot-mcp` with `IoTDeviceMcpTool`) remains the production MCP surface during migration. The generated MCP surface from SPI interfaces is deployed alongside for verification. Once verified, the curated module is deprecated and eventually removed.
**Alternatives:**
- Immediate replacement — generated surface replaces curated surface in one step. Risk: generated tools may not match curated tool quality (parameter descriptions, error messages)
- Permanent coexistence — both surfaces exist indefinitely. Creates confusion for LLM agents about which tools to use
**Rationale:** The curated MCP tools (`iot_get_devices`, `iot_get_state`, `iot_send_command`, `iot_get_history`) have hand-tuned descriptions, parameter validation, and error handling. The generated surface may not initially match this quality. Coexistence during migration allows comparing generated vs curated tool behavior before committing to the generated surface. The migration is complete when: (1) generated MCP tools cover all curated tool functionality, (2) parameter descriptions and error messages meet quality bar, (3) curated MCP module is deprecated with a removal timeline.
**Trade-offs:** During migration, LLM agents may see overlapping tools from both surfaces. This is manageable via MCP domain naming — the curated tools use flat names (`iot_get_devices`) while generated tools use domain-scoped names (`iot/devices/list`).
**Sources:** IoTDeviceMcpTool.java (curated MCP surface with 4 tools), issue #105 scope
**Exploration:** surfaced by reviewer (R1-22)
**Status:** captured

## D8: REST base path — basePath attribute on @McpDomain

**Choice:** New `basePath` attribute on `@McpDomain` decouples MCP domain name from REST class-level `@Path`. IoT SPIs use `@McpDomain(value = "iot/devices", basePath = "/api/devices")` — MCP namespace is `iot/devices`, REST path is `/api/devices`.
**Alternatives:**
- Short domain names (`devices` instead of `iot/devices`) — sacrifices MCP namespace grouping
- Accept new paths (`/api/iot/devices/...`) and update frontend — clean break but unnecessary churn
- No change — live with mismatched paths
**Rationale:** The generator conflated MCP domain name and REST base path (line 752: `@Path("/api/" + domain)`). These serve different purposes — MCP namespace groups tools for LLM agents across domains, REST path is the URL structure for HTTP clients. Decoupling is the right architectural fix. Small change: one attribute on `@McpDomain`, one line in generator, one line in shared scanner.
**Trade-offs:** None — backward compatible, empty basePath derives from value as before.
**Implementation:** Done. platform#333, committed to `issue-311-context-param` branch, installed to slot .m2.
**Cross-repo benefit:** engine#1095 and work#400 can retroactively add basePath to match their original REST paths before merging.
**Sources:** GraphQLResolverProcessor.java line 752, platform#295 path convention, first-principles concern separation
**Exploration:** deep-analysis
**Status:** captured — implemented
