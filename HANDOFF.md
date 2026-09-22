# Handover — Slot 194

## Active Issue
`casehubio/platform#380` — complete. Queue position 29/32.

## Context

The @McpDomain migration (platform#300 epic) continues. This session completed platform#380: triaging all remaining hand-written `@Path` REST resources across 17 repos. Three issues remain (#381 consolidation, #382 eval).

## What Was Done

### platform#380 — @HandWrittenEndpoint or delete old REST (complete)

Surveyed all 17 slot repos plus qhorus for hand-written `@Path` classes. Triaged each into one of three categories: delete (covered by @McpDomain-generated endpoint), annotate `@HandWrittenEndpoint` (genuinely hand-written — webhooks, SSE, auth, callbacks, game simulation), or annotate as gap (no @McpDomain SPI yet).

**Deletions (38 files across 7 repos):**
- chat-app: 2 (ChatResource, PresenceResource)
- claudony: 6 (CaseBrowser, ActionInbox, Casehub, Session, Mesh, Peer)
- life: 8 (Dashboard, ExternalActor, Analytics, Case, Commitment, OversightGate, Task, PendingActions)
- ops: 8 (Security, Deployment, Approval, Cluster, Scaling, Case, ServiceOperation, Application)
- devtown: 2 (PrReview, Governance)
- fsitrading: 10 (Layout, Position, Order, Kpi, MarketData, Strategy, Audit, TrustScore, Compliance, Incident)
- openclaw: 2 (ScenarioRest, ChannelContextWindow)

**@HandWrittenEndpoint annotations (genuine — ~50 files across 16 repos):**
- Webhooks/callbacks: work (Jira, GitHub), devtown (GitHub), connectors (WebhookRouter), platform (CallbackDispatch, EngagementCallback), workers (WorkerCallback)
- SSE streaming: iot (DeviceSse), life (LifeEventSse), ops (Reconciliation), openclaw (ScenarioSse)
- Auth flows: claudony (Auth)
- Protocol endpoints: qhorus (AgentCard, A2A, WebhookRegistry, ExternalAgentBinding, SlackBinding)
- External system connectors: soc (7 connectors + health checks)
- Engine infrastructure: engine (ActorState)
- Game simulation: quarkmind (14 resources)
- Delivery mechanisms: openclaw (3 delivery + plugin + example)
- Clinical: DemoAction (simulation), PatientCompliance (cross-cutting audit)
- Devtown (can't delete — SPI injects REST resource directly): CodeReviewCompliance, GdprErasure, MemoryAdmin, GovernancePreferences

**Gap annotations (~24 files across 6 repos):**
Annotated with `@HandWrittenEndpoint("gap: no @McpDomain SPI — see platform#381")`:
- work: SlaAdmin, WorkItemTemplate, FederationEvent (already had annotation)
- iot: WorkItemResource, KpiResource
- clinical: NarrativeResource
- qhorus: ChannelResource, SpaceResource, CausalGraphResource, ComplianceScheduleResource (already had)
- devtown: IncidentFeedbackResource
- fsitrading: 10 resources (Preferences, Evaluation, GdprErasure, RoutingDecision, IncidentHistory, WorkItem, PostMortem, Narrative, Deliberation, SimilarIncident)

**Key finding:** devtown SPIs inject old REST resources directly via `.getEntity()` — 4 resources can't be deleted until refactored. This anti-pattern should be checked across other repos as part of platform#381.

## Queue (platform#300 children)

1. ~~**platform#380**~~ done — @HandWrittenEndpoint or delete old REST
2. **platform#381** — Consolidation: shared ApiResult, merge single-method classes
3. **platform#382** — Eval: @McpDomain real-world LLM usability

## Notes for Next Session

- platform#380 is ready to close on GitHub
- platform#381 should also address the devtown `.getEntity()` anti-pattern and the ~24 gap resources
- platform#382 is evaluation-only — hands-on testing, document findings
- ledger and aml were already clean (no old REST resources to triage)

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known
