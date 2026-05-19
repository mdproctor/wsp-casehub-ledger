# Design Journal — epic-reactive-key-service

### 2026-05-19 · §Architecture

`ReactiveKeyRotationRepository` was added as the reactive twin to `KeyRotationRepository`,
following the same SPI-only pattern established by `ReactiveLedgerEntryRepository`: no
production JPA implementation is bundled in `casehub-ledger`. The interface is
persistence-agnostic — consumers provide their own implementation using whatever reactive
stack they run (Hibernate Reactive, reactive MongoDB, etc.). The test suite resolves the
injection via `BlockingReactiveKeyRotationRepository`, a `@DefaultBean` shim that wraps
the H2/JDBC blocking impl with `Uni.createFrom().item()`. This keeps the reactive contract
testable without a Vert.x datasource. `KeyRotationService` gained three reactive methods
(`compromisedWindowsAsync`, `rotationHistoryAsync`, `recordRotationAsync`), completing
PP-20260517-15bf75 for the key rotation domain.

### 2026-05-19 · §Key Design Decisions

The blocking bridge in `verifyAgentSignatureAsync` (introduced in #83 as a placeholder)
was replaced with `compromisedEffectiveSinceAsync`, a private reactive helper that mirrors
`compromisedEffectiveSince` exactly — same `.min(Instant::compareTo)` (order-independent,
not `findFirst()` which would silently depend on query ordering) and same null-free filter
on `occurredAt` (which is `@Column(nullable = false)` with a `@PrePersist` fallback). The
`findFirst()` / `.min()` divergence was caught in code review; the fix ensures both paths
remain correct even if the named query's `ORDER BY effectiveSince ASC` changes.
`recordRotationAsync` carries no `@Transactional` annotation — reactive transaction
management is the caller's responsibility, consistent with the reactive SPI contract.
