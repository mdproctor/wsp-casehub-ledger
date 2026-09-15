## D1: SPI interface grouping

**Choice:** 5 SPI interfaces organized by caller intent
- `WorkItemApi` (work/items) — 10 methods: CRUD, query, inbox, labels, streaming
- `WorkItemLifecycleApi` (work/lifecycle) — 17 methods: all lifecycle transitions
- `WorkItemNoteApi` (work/notes) — 4 methods: note CRUD
- `WorkItemLinkApi` (work/links) — 3 methods: link CRUD
- `WorkItemRelationApi` (work/relations) — 6 methods: relations + children/parent
**Alternatives:**
- 7 fine-grained SPIs (labels as separate interface) — unnecessary granularity for 2 methods
- 3 coarse-grained SPIs (merge notes+links+relations) — too wide, unclear responsibility
**Rationale:** Matches engine#1095 pattern (5 focused SPIs). Each has a clear MCP domain. Labels (2 methods) fold naturally into the core WorkItemApi.
**Trade-offs:** WorkItemLifecycleApi has 17 methods — large but cohesive (all lifecycle state transitions).
**Sources:** engine spec (2026-09-15-engine-api-generation-design.md), WorkItemResource.java survey
**Exploration:** quick
**Status:** captured

## D2: Service layer strategy

**Choice:** SPI impl beans ARE the service layer — no separate extraction step
**Alternatives:**
- Extract a WorkItemService first, then have impl beans delegate to it — extra indirection for no benefit
- Keep logic in resources, have impl beans call resources — circular, defeats the purpose
**Rationale:** Work has no existing service layer. The inline logic in WorkItemResource is mostly query construction and entity mapping (not complex domain logic). Moving it directly to impl beans follows the engine pattern (DefaultEngineCaseApi contained all mapping logic). WorkItemOperations and WorkItemStore already provide the repository/domain layer.
**Trade-offs:** Impl beans are slightly thicker than pure delegates. The heaviest is WorkItemRelationApi (~40 lines for BFS cycle detection).
**Sources:** WorkItemResource.java (823 lines), DefaultEngineCaseApi.java pattern
**Exploration:** quick
**Status:** captured

## D3: View records strategy

**Choice:** New view records for response types; reuse api types for requests, enums, and summaries
**Alternatives:**
- Reuse all api records directly — leaks 20 internal fields (tenancyId, escalation config, operational metrics, provenance internals) into REST/GraphQL responses
- Create views for absolutely everything — unnecessary for request types and enums that don't have a curation gap
**Rationale:** WorkItem (57 fields) vs WorkItemResponse (37 fields) shows deliberate curation — 20 fields excluded, 3 reshaped. Exposing WorkItem directly would be a security and API design regression. But request types (WorkItemCreateRequest), enums (WorkItemStatus, WorkItemPriority), and summaries (WorkItemSummary) are already safe shapes.
**Trade-offs:** More mapping code than pure reuse, but less than engine (which created ALL new types). Estimated ~10-12 new records.
**Sources:** WorkItem.java (57 fields), WorkItemResponse.java (37 fields), WorkItemWithAuditResponse.java (38 fields)
**Exploration:** deep-analysis
**Status:** captured
