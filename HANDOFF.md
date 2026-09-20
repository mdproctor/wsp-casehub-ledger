# Handover — Slot 194

## Branch
SOC on branch `issue-55-mcpdomain-spi` (9 commits, all 474 tests pass, issue closed).
Connectors on branch `issue-100-mcpdomain-full-parity` (1 commit, compiles clean).
All other repos on main.

## Active Issue
`casehubio/life#118` — migrate life to @McpDomain SPI.
Queue position 7/24.

## Session Summary

### soc#55 — @McpDomain SPI Migration (COMPLETE, issue closed)

Full migration of 8 hand-written JAX-RS REST resources to 7 `@McpDomain` classes:
- 24 typed DTO records replacing `Map<String,Object>`
- `SocTrustService` extracted from inline logic
- APT generator wired with domain filter
- 474 tests pass, 0 failures
- CDI ambiguity resolved via `SocBeanOverrides`

### connectors#100 — @McpDomain Full Parity (IN PROGRESS)

Migrated 14 hand-written `@Tool` MCP operations to 3 new `@McpDomain` classes:
- `ConnectorMessagingApi` (connectors/messaging) — SMS, WhatsApp, email, Slack, Teams
- `ConnectorChatApi` (connectors/chat) — send_chat, list_chat_channels, list_channels
- `ConnectorCalendarApi` (connectors/calendar) — 6 calendar CRUD ops

All 18 ops now generate REST + GraphQL + MCP. APT generator wired. Compiles clean.
**Remaining:** delete old `mcp/` module `@Tool` classes, run tests, close issue.

### CDI Fixes (applied across slot repos)

- engine: `@Vetoed` on `ActorStateResource`/`ActorStateAggregator`, CDI ambiguity fixes
- eidos: removed `@DefaultBean` from `DefaultCapabilityHealth`
- neocortex: trust imports updated from `blocks.trust` to `engine.trust`
- SOC: `SocBeanOverrides` for local CDI disambiguation
- Test config: added connector defaults (twilio, whatsapp, slack)

### Upstream Issues

| # | Repo | Issue | Status |
|---|------|-------|--------|
| 1 | neocortex | #369 — TrustConsolidationPhase ClassNotFoundException | Fixed locally, upstream deleted the file |
| 2 | engine | CDI fixes | Pushed to origin |

### Slot Maintenance

- Rebased all 20 repos from canonical main
- Reinstalled platform, engine, work, ledger, qhorus, iot to .m2
- Fixed `InclusionProof` import drift in SOC (ledger package move)

## What's Next

1. Finish connectors#100: delete `mcp/` module, run tests, close issue
2. `work next` → life#118 (migrate life to @McpDomain SPI)
3. Continue through remaining queue (ops, devtown, fsitrading, claudony, etc.)

## Canonical Repo Sync Status

Several canonical repos are stalled on feature branches that were blocked by upstream CDI issues (now fixed):

| Canon repo | Stalled branch | May be unblocked by engine CDI fixes |
|---|---|---|
| aml | issue-10-operational-tooling-mcp | Yes |
| connectors | issue-94-bankfeed-email-spis | Possibly |
| devtown | issue-203-llm-reviewer-agents | Yes |
| life | issue-116-household-onboarding | Yes |
| quarkmind | issue-306-retrain-onnx-spatial | Possibly |
| soc | issue-51-rag-investigation-enrichment | Yes |

These canonical repos should be checked — the CDI fixes may unblock their stalled work.

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` — all slot repos installed, non-slot repos resolve from host .m2 fallback.
3. `casehub-platform-graphql-generator` is the APT processor.
