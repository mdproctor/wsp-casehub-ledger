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
