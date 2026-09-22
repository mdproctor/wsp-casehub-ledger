# Handover — Slot 194

## Active Issue
`casehubio/work#405` — completed this session. Branch `issue-405-mcpdomain-rest` in work repo (3 commits, not yet merged).
Queue position 27/31 — advance needed to activate next issue.

## Context

The @McpDomain migration (platform#300 epic) is in its hardening phase. This session completed work#405 — the largest remaining coverage gap: 25 bare REST resources across 9 modules in casehub-work.

## What Was Done

### work#405 — @McpDomain for 25 bare REST resources

All 25 production REST resources in casehub-work now have @McpDomain coverage:

| Category | Count |
|----------|-------|
| @McpDomain interfaces/classes created | 20 |
| View/request records created | 53 |
| Implementation classes created | 20 |
| Hand-written REST resources deleted | 19 |
| @HandWrittenEndpoint annotated | 4 |
| Webhooks skipped (not domain APIs) | 2 |
| Modules with APT generator wired | 9 |

**Commit 1** (`31b7d34`): rest module — 9 resources converted (audit, vocabulary, spawn-groups, instances, bulk, spawn, schedules, templates, label-rules). Template PATCH retained as @HandWrittenEndpoint (JSON Merge Patch needs raw JsonNode).

**Commit 2** (`60a15a4`): SlaAdminResource and FederationEventResource annotated @HandWrittenEndpoint (admin infra and inbound webhook).

**Commit 3** (`0c654bf`): 7 remaining modules — queues(2), ai(3), federation(1), issue-tracker(1), reports(1), progress-rest(1), ledger(2). APT generator wired in all 7 module pom.xml files.

### Ecosystem scan — zero gaps

Scanned all 8 repos in slot 194. No bare REST resources remain anywhere:
- aml: migrated in aml#130
- clinical, life, soc, iot, chat-app, ledger: clean
- work: migrated this session

The AML note in the previous handoff about remaining hand-written resources was stale.

### Generator bugs discovered

Two APT generator issues encountered during this session (relevant to platform#379):
1. **`page` variable collision** — `@PaginatedResponse` generates `var page = ...` which collides if the API method has a parameter named `page`. Workaround: renamed to `pageIndex`.
2. **Hardcoded `totalCount()` accessor** — `@PaginatedResponse` calls `.totalCount()` on the return type. The field name must be exactly `totalCount`, not `total` or anything else.

## Queue (platform#300 children)

1. ~~**platform#378**~~ done — Replace 104 Object returns with typed records
2. ~~**ledger#210**~~ done — Wire APT generator for 5 existing @McpDomain classes
3. ~~**work#405**~~ done — @McpDomain for 25 bare REST resources
4. **platform#379** — Generator bugs: @PaginatedResponse shadowing, @Valid dep, @DefaultValue
5. **platform#380** — @HandWrittenEndpoint or delete old REST across 13 repos
6. **platform#381** — Consolidation: shared ApiResult, merge single-method classes

## Standing Rules

- **Type safety is non-negotiable** — no `Object` returns, no raw types, proper generics always
- Inject services directly — never delegate to REST resources via `.getEntity()`
- `List<T>`, `Map<K,V>`, `Optional<T>` — always parameterised
- `Map<String,Object>` → create a typed record when the structure is known

## Notes for Next Session

- The .plan still shows work#405 as active — run `work next` to advance to platform#379
- work#405 branch (`issue-405-mcpdomain-rest`) needs work-end to merge to main
- platform#379 is in casehubio/platform — different repo, different kind of work (generator internals)
- The two generator bugs found this session (page collision, totalCount hardcode) may be the same issues tracked in platform#379
- Some work modules (queues, ai) are commented out of the parent reactor (tracked by work#403) — deleting old REST resources may help re-enable them
