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

## Ecosystem Context — Other Active Slots on Ledger

| Slot | Branch | Issue | Status |
|------|--------|-------|--------|
| 189 | `issue-206-extract-framework-neutral-core` | ledger#206 | scaffolded |
| 192 | `issue-474-spring-boot-generators` | platform#474 | active |
| 194 | `issue-295-unified-api-generation` | ledger#207 | active (this slot) |

## Session Progress

### Session 1 (2026-09-14)
- Rebased from origin/main — already up to date
- Confirmed parent epic closed, ledger child #207 is open and unworked
- Updated .plan to track casehubio/ledger#207
- Research in progress: examining current endpoints and platform generator

## What's Next

| Item | Scale | Complexity |
|------|-------|------------|
| Understand @McpDomain generator capabilities from platform | S | Low |
| Inventory current REST + GraphQL endpoints in ledger | S | Low |
| Design SPI interfaces for ledger domain | M | Med |
| Implement migration (replace hand-written with generated) | M | Med |
| Delete old hand-written endpoint classes | S | Low |
| Verify generated endpoints match existing API surface | S | Med |
