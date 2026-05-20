# Design Journal — issue-93-agent-signature-verification-service

### 2026-05-20 · §Key Design Decisions

`LedgerVerificationService` held two unrelated concerns: Merkle tree operations and agent signature verification. The extraction separates them cleanly. `LedgerVerificationService` is now Merkle-only; `AgentSignatureVerificationService` (blocking) and `ReactiveAgentSignatureVerificationService` (reactive) own the signature pipeline. The `verifyCryptographic` method was duplicated verbatim between the old blocking and reactive beans; extraction to `AgentCryptographicVerifier` (package-private `final` static utility, no CDI) eliminates that duplication and makes the single source of truth explicit. This class mirrors the `LedgerMerkleTree` pattern already established in the codebase.

### 2026-05-20 · §Architecture

The reactive service tier separation established in #92 is now structurally complete for signature verification. The blocking/reactive pairing is now symmetric: `AgentSignatureVerificationService` ↔ `ReactiveAgentSignatureVerificationService`, `KeyRotationService` ↔ `ReactiveKeyRotationService`. `LedgerVerificationService` has no reactive counterpart because Merkle verification is always blocking (it requires sequential consistency over the full subject history). `BlockingTierPurityTest` enforces this via reflection — now covers all three blocking-tier service beans. `LedgerProcessor.excludeReactiveBeans` gates `ReactiveAgentSignatureVerificationService` (renamed from `ReactiveLedgerVerificationService`) and `ReactiveKeyRotationService` via `ExcludedTypeBuildItem` when `casehub.ledger.reactive.enabled=false`.
