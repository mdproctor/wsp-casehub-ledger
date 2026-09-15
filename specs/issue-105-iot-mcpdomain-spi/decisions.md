## D1: Module placement for SPI interfaces and view records

**Choice:** `api/` module — consistent with engine/work/ledger pattern
**Alternatives:**
- `webapp-api/` — respects iot's existing layering but deviates from cross-repo pattern, requires Jandex/compiler setup
- New `iot-spi/` module — clean separation but over-engineering, adds Maven module overhead
**Rationale:** Jandex gotcha (GE-20260914-f53be7) confirms SPI must be in a pre-compiled dependency JAR. `api/` already has Jandex configured. View records are additive (new `io.casehub.iot.api.view` package), don't change existing device provider SPIs.
**Trade-offs:** Webapp-specific view records (case summaries, resolution queue entries) live alongside device provider SPIs in the same module, blurring the boundary. Manageable because they're in a separate package and don't affect the existing public API.
**Sources:** GE-20260914-f53be7, engine api/ module, work api/ module
**Exploration:** quick
**Status:** captured

## D2: Domain groupings — 4 SPI interfaces

**Choice:** 4 interfaces organized by caller intent:
1. `IoTDeviceApi` (`iot/devices`) — 5 methods (list, get, command, history, stream)
2. `IoTSituationApi` (`iot/situations`) — 10 methods (definitions CRUD, active, suggestions, dismissal, suppressions)
3. `IoTCaseApi` (`iot/cases`) — 6 methods (cases + resolution queue merged)
4. `IoTOperationsApi` (`iot/ops`) — 7 methods (providers, bridge, health)
**Alternatives:**
- 5 interfaces (separate resolution queue) — cleaner separation but resolution queue is tightly coupled to cases
- 6 interfaces (add workitems + KPI) — workitems are placeholder, KPI doesn't fit a single domain
**Rationale:** Cases and resolution queue share the same domain context (a resolution queue entry IS a case). WorkItemResource is excluded (almost entirely placeholder, real work-item API is in casehub-work). KPI endpoints excluded (thin aggregations, path structure doesn't fit one domain).
**Trade-offs:** 6 hand-written endpoints remain (2 KPI, 4 WorkItem placeholder). These stay as `KpiResource` and `WorkItemResource` in `webapp`.
**Sources:** DeviceResource, CaseResource, SituationResource, ProviderResource, BridgeResource, HealthResource, ResolutionQueueResource, DeviceSseResource
**Exploration:** quick
**Status:** captured

## D3: View record strategy for existing webapp-api DTOs

**Choice:** New view records in `api/view/`, existing webapp-api DTOs remain for internal use
**Alternatives:**
- Move existing webapp-api DTOs to `api/view/` — avoids duplication but forces import changes on any existing consumers, some DTOs have nested inner records needing flattening
- Reuse webapp-api DTOs from SPI — creates dependency from `api/` on `webapp-api/` (inverts natural dependency direction)
**Rationale:** View records ARE the curation step. They curate fields for the generated API surface and become the single source of truth for API shape. Some fields may differ from what hand-written DTOs expose. Webapp-api DTOs used internally by service classes (e.g. QueueEntrySummary) can stay until no longer needed.
**Trade-offs:** Temporary type duplication — view records in `api/view/` coexist with webapp-api DTOs until the old DTOs are cleaned up. Impl beans do the mapping.
**Depends on:** D1 (module placement determines where view records live)
**Sources:** engine api/view/ records, work api/view/ records, webapp-api DeviceResponse/CommandRequest/etc.
**Exploration:** quick
**Status:** captured

## D4: SSE stream handling

**Choice:** Include in IoTDeviceApi as `@PlatformStream` method — `deviceStateStream(tenancyId)` returns `Multi<DeviceStateEventView>`. Impl bean bridges the existing BroadcastProcessor pattern.
**Alternatives:**
- Keep DeviceSseResource hand-written — simpler but leaves a gap in the generated API surface
**Rationale:** One method, generator handles SSE framing. Keeps the device API complete — consumers get devices, commands, history, AND live state from one interface.
**Trade-offs:** None significant.
**Sources:** DeviceSseResource, GE-20260804-8b0fd6 (BroadcastProcessor pattern)
**Exploration:** quick
**Status:** captured

## D5: Placeholder/unimplemented endpoint strategy

**Choice:** Default methods + `NotImplementedException` + exception mapper. SPI uses Java `default` methods for unimplemented operations that throw `NotImplementedException`. A JAX-RS exception mapper converts to HTTP 501.
**Alternatives:**
- `@Unimplemented` annotation on SPI methods — generator recognizes and produces 501 directly. But mixes lifecycle state into the contract (layer violation), requires generator changes, and needs two changes to "turn on" a method (remove annotation + add impl override)
- Impl bean throws manually — pure, but requires boilerplate per unimplemented method
**Rationale:** Java already has the right mechanism: abstract = must implement, default = can defer. This uses standard Java patterns (default methods, exceptions, exception mappers) with zero generator changes. The SPI remains a pure contract. Only one change to "turn on" a method (add override in impl bean). Works across all three surfaces (REST, GraphQL, MCP) via standard error handling.
**Trade-offs:** The generated OpenAPI spec can't distinguish unimplemented from implemented endpoints. Acceptable because these are internal APIs and 501 at runtime is sufficient.
**Depends on:** D2 (domain groupings determine which methods are abstract vs default)
**Implementation scope:**
1. `NotImplementedException` → `platform-api` (casehubio/platform#300, in scope for slot)
2. `NotImplementedExceptionMapper` → platform runtime support (one registration, all domains)
3. IoT SPI uses default methods for `listCases`, `getCase`, `listActive`
**Sources:** First-principles analysis of contract vs lifecycle separation, Java default method semantics, JAX-RS exception mapper pattern
**Exploration:** deep-analysis
**Status:** captured
