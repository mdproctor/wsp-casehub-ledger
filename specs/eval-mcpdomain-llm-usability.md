# Evaluation: @McpDomain Real-World LLM Usability

**Issue:** casehubio/platform#382
**Date:** 2026-09-23
**Scope:** Structural evaluation of discovery, hierarchy, and usability across the full CaseHub org (16 repos, 142 domains, 607 operations)

## Surface Area

| Repo | Domains | Queries | Mutations | Inner Records |
|------|---------|---------|-----------|---------------|
| platform | 16 | 46 | 47 | 1 |
| engine | 5 | 16 | 5 | 0 |
| work | 26 | 42 | 70 | 1 |
| iot | 8 | 22 | 14 | 8 |
| clinical | 16 | 38 | 19 | 0 |
| aml | 9 | 20 | 18 | 0 |
| soc | 7 | 18 | 2 | 0 |
| life | 5 | 18 | 8 | 0 |
| ops | 8 | 14 | 14 | 5 |
| devtown | 7 | 21 | 5 | 13 |
| fsitrading | 14 | 38 | 15 | 9 |
| claudony | 4 | 13 | 13 | 4 |
| openclaw | 5 | 5 | 13 | 0 |
| chat-app | 2 | 2 | 5 | 0 |
| connectors | 6 | 19 | 16 | 0 |
| ledger | 4 | 9 | 2 | 0 |
| **Total** | **142** | **341** | **266** | **41** |

## Q1: Hierarchy Fitness

**Verdict: The three-tier hierarchy (domain -> operation -> params) is sound for single-app deployments. It breaks down at org scale.**

The hierarchy works well when a running app has 4-20 domains — the LLM calls `casehub_model()`, gets a manageable list, drills into the right domain, and dispatches. This is the typical single-app case.

At full org scale (142 domains), the level 1 list becomes a wall of text. An LLM must scan all 142 entries to find the right domain. Slash-separated naming (`fsi/compliance`, `work/items`) provides implicit grouping, but the LLM still receives a flat list.

**What works:**
- Two-step drill-down (list domains -> inspect domain) is the right pattern
- Operation summaries from `@PlatformQuery`/`@PlatformMutation` give the LLM enough context to pick the right operation
- Complex params (records) get one-level field expansion — good enough for most cases
- `casehub_activate` per-domain tool explosion is elegant for focused work

**What doesn't:**
- No app-level grouping. An LLM can't ask "what can the trading app do?"
- 142 domains in a flat list is 3-4x beyond the practical threshold (~30-40) for reliable LLM selection
- Domain names use inconsistent conventions: some use org/area (`fsi/compliance`), some use bare names (`ledger`)

**Recommendation:** Add an app-level tier. The hierarchy becomes: app (16) -> domain (~9 per app) -> operation. Level 0 returns 16 app summaries. Level 1 returns the domains for one app. Level 2 returns operations for one domain. Each tier stays under 30 items.

## Q2: Discovery Efficiency

**Verdict: Good for targeted lookups within a known domain. Poor for exploratory "what can the system do?" queries.**

Three discovery modes serve different use cases:

| Mode | Best for | Limitation |
|------|----------|------------|
| `casehub_model` | Browsing domains and operations | Flat 142-domain list at level 1 |
| `casehub_action` SIMPLE | Execution when LLM knows the operation | Description blob with 607 operations is noise |
| `casehub_action` RICH | Execution with schema validation | Client must support `allOf`/`if-then` JSON Schema |
| `casehub_activate` | Focused work within one domain | Requires knowing which domain to activate first |
| MCP Resources | Pre-loading context | Same flat list problem |

**Signal-to-noise analysis:** When an LLM needs to "create a work item," it must:
1. Call `casehub_model()` — receive 142 domains
2. Scan for work-related domains — find `work/items`, `work/queues`, `openclaw/workitems`, etc.
3. Disambiguate — which domain owns work item creation?
4. Call `casehub_model(domain="work/items")` — get operations
5. Pick the right operation and call `casehub_action`

Steps 1-3 are where efficiency drops. The LLM will frequently pick the wrong domain or need multiple drill-down calls.

**Recommendation:** Add a `casehub_search` tool — keyword search across operation names and summaries. "create work item" returns the 2-3 matching operations directly, skipping the domain browsing entirely. This is the highest-impact single improvement.

## Q3: Indexing

**Verdict: Yes, a lightweight index layer would dramatically improve discovery at scale.**

Current state: all discovery is browse-based. The LLM must navigate the hierarchy top-down.

Proposed index layers (in priority order):

### 3a. Semantic search (highest impact)

A `casehub_search(query: String)` tool that searches operation descriptions with basic text matching. At 607 operations, even substring search over summaries is transformative.

Implementation: in-memory at startup — the `DomainModelRegistry` already has all operation metadata. Add a search method that matches against operation names, summaries, param names, and return types. No external dependencies.

### 3b. App-level grouping

Add `app` field to the domain model. Populate from the Maven module (or a `@McpDomain(app="fsitrading")` attribute). The domain list becomes filterable: `casehub_model(app="fsitrading")` returns only fsitrading's 14 domains.

### 3c. Capability tags

Cross-cutting categories: `compliance`, `messaging`, `trading`, `identity`, `audit`. Applied via annotation attribute: `@McpDomain(value="fsi/compliance", tags={"compliance", "audit"})`. Enables: `casehub_search(tag="compliance")` returns domains from fsitrading, clinical, and soc.

### 3d. Operation manifests (lowest priority)

Static YAML manifests per app listing domain-to-operation mappings. Useful for offline/CI analysis but lower priority than runtime search.

## Q4: Cross-Repo Automation

**Verdict: The infrastructure is production-ready for single-app automation. Org-wide automation needs a federated registry.**

**What works today:**
- Any consumer app with `casehub-platform-mcp` on classpath gets full MCP tooling at startup
- The manifest system (`agent-config.yaml`) wires LLM providers declaratively
- Vertex integration means zero API key management for GCP-native deployments
- `casehub_activate` creates typed per-operation tools — the best UX for focused automation

**What's missing for org-wide automation:**

1. **No unified registry** — each running app registers only its own domains. An LLM connected to fsitrading can't discover or call ledger operations.

2. **No federated discovery** — to see the full org's capabilities, you'd need one app that depends on all other apps (claudony comes closest as the composer).

3. **No cross-app dispatch** — `casehub_action` dispatches to local CDI beans only. Cross-app calls need HTTP forwarding.

**Recommendation:** For the near term, claudony (the composition layer) should be the unified MCP entry point — it already depends on most other modules. For the long term, consider a lightweight MCP gateway that aggregates domain registries from multiple running apps.

## Q5: Practical Gaps

### Gap 1: Operation summary quality

Some `@PlatformQuery`/`@PlatformMutation` annotations have generic or empty summaries. "List items" tells the LLM nothing about what kind of items. Build-time validation should enforce non-empty, descriptive summaries.

**Proposed fix:** Add Jandex validation in the deployment module — fail the build if a `@PlatformQuery` or `@PlatformMutation` has an empty value attribute.

### Gap 2: Return type erasure

`casehub_model` returns simple type names: `"returns": "List"` instead of `"returns": "List<LedgerEntry>"`. The LLM can't tell what it gets back without calling the operation and inspecting the response. This matters when multiple operations return `List` — which list?

**Proposed fix:** Resolve generic type parameters in `GraphQLModelScanner`. The method signature has the full `ParameterizedType`; emit `"List<LedgerEntry>"` or at minimum `"LedgerEntry[]"`.

### Gap 3: No operation examples

The LLM knows parameter names and types but not what values to pass. For UUIDs, enums, and structured records, example values would reduce trial-and-error.

**Proposed fix:** Optional `@Example` annotation on parameters. Include in the Level 2 discovery output. Low priority — the error messages from `ReflectiveOperationDispatcher` already include expected types and fields.

### Gap 4: Remaining inner records

41 inner records across 6 repos. These work at runtime but complicate generated REST resources and type introspection. The APT generator handles them, but extracted records are cleaner for both human and LLM consumers.

**Proposed fix:** Continue extraction as part of ongoing consolidation. Not blocking.

### Gap 5: Domain naming inconsistency

Some domains use `org/area` convention (`fsi/compliance`, `work/items`), others use bare names (`ledger`, `quarkmind`). The slash convention implies hierarchy but isn't enforced.

**Proposed fix:** Establish a naming convention: `{app}/{area}` for multi-domain apps, bare names acceptable for single-domain apps. Document in contributor guide.

## Summary of Recommendations

| # | Recommendation | Impact | Effort | Issue |
|---|---------------|--------|--------|-------|
| 1 | `casehub_search` tool — keyword search across operations | High | S | TBD |
| 2 | App-level grouping in `casehub_model` | High | M | TBD |
| 3 | Build-time summary validation | Medium | XS | TBD |
| 4 | Return type generics in discovery output | Medium | S | TBD |
| 5 | Domain naming convention | Low | XS | TBD |
| 6 | Capability tags | Low | M | TBD |
| 7 | Operation examples | Low | S | TBD |
