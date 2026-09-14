# Handover — Slot 194 / casehub-ledger

## Branch
`issue-295-unified-api-generation` on `casehubio/ledger`

## Active Issue
`casehubio/ledger#207` — unified API generation: migrate hand-written REST and
GraphQL endpoints to `@McpDomain` SPI interfaces with JAX-RS annotations.

Parent epic `casehubio/platform#295` is CLOSED (platform-side generator is done).
This is the ledger-specific child work.

## Standing Instructions

1. **Always rebase from origin/main** before starting work each session.
2. **Check work isn't already done or being done** in other slots before proceeding.
3. **AML is ongoing** — Slot 181 (`issue-469-dual-framework-core-extraction`) is
   actively working across many repos including `aml` and related workspaces.
   Avoid conflicts with that work stream.

## Critical: Slot-Local Maven Repository

This slot uses a **slot-local `.m2`** at `/Users/mdproctor/claude/casehub/slots/194/.m2`.
Any `mvn install` must target this directory, not `~/.m2/repository`. The slot's
`.mvn/slot-settings.xml` configures this with a `host-m2` fallback to `~/.m2/repository`.

When patching platform jars (e.g., `casehub-platform-graphql-generator`), copy the
patched jar to BOTH locations:
- `~/.m2/repository/io/casehub/<artifact>/.../` (host fallback)
- `/Users/mdproctor/claude/casehub/slots/194/.m2/io/casehub/<artifact>/.../` (slot local — this is what Maven actually uses)

## Ecosystem Context — Other Active Slots on Ledger

| Slot | Branch | Issue | Status |
|------|--------|-------|--------|
| 189 | `issue-206-extract-framework-neutral-core` | ledger#206 | scaffolded |
| 192 | `issue-474-spring-boot-generators` | platform#474 | active |
| 194 | `issue-295-unified-api-generation` | ledger#207 | active (this slot) |

## Session Progress

### Session 1 (2026-09-14)

**Completed:**
- Rebased from origin/main — already up to date
- Confirmed parent epic closed, ledger child #207 is open and unworked
- Updated .plan to track casehubio/ledger#207
- Brainstormed design: 6 decisions captured, all constraint-driven
- Wrote spec: `specs/issue-295-unified-api-generation/2026-09-14-unified-api-generation-design.md`
- Wrote plan: `plans/2026-09-14-unified-api-generation.md`
- **Task 1 DONE**: Created 10 view/request records + 4 SPI interfaces in `api/`
- **Task 2 DONE**: Created 4 `DefaultXxxApi` service beans in `runtime/service/api/` with unit tests
- **Task 3 IN PROGRESS**: APT wiring
  - Updated `rest/pom.xml` and `graphql/pom.xml` with APT config and dependency changes
  - APT generates correct class names (`GeneratedLedgerEntriesResource`, etc.)

**Discovered issues:**
- **Generator `/` in domain names** — `toPascalCase` didn't handle `/` separator.
  Fixed with `kebab.replace('/', '-')` in platform's `GraphQLResolverProcessor`.
  Currently a LOCAL PATCH in `~/.m2` and slot `.m2` — needs a proper platform commit.
  Protocol PP-20260914-7387db captured for this rule.
- **Pre-existing `LedgerMerkleFrontier` entity issue** — Hibernate says "no identifier".
  All `@QuarkusTest` tests in runtime, rest, graphql fail. Not caused by this branch —
  from the core extraction (commit `983a274`). Service implementation tests use plain
  JUnit with no-op repos as workaround.
- **Slot-local `.m2`** — caused hours of debugging. Generator patches installed to
  `~/.m2/repository` were invisible to the slot's Maven. Must install to slot's `.m2`.

**Audit result:** No repos have accidentally flattened hierarchical `@McpDomain` values.
All existing repos use flat domains. Ledger is the first to use hierarchical `/` domains.
Candidates for future hierarchy: notifications (digest, delivery-channels,
notification-preferences), preferences (preference-schemas), qhorus (compliance),
engine (cases).

## What's Next

| Item | Scale | Complexity |
|------|-------|------------|
| Delete old hand-written REST resources and DTOs from `rest/` | S | Low |
| Delete old hand-written GraphQL resolvers and DTOs from `graphql/` | S | Low |
| Update REST tests for new generated paths | S | Med |
| Update GraphQL tests for generated resolvers | S | Med |
| File platform issue for generator `/` fix (PP-20260914-7387db) | XS | Low |
| Investigate LedgerMerkleFrontier @Id issue (pre-existing) | S | Med |
