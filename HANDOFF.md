# CaseHub Ledger — Session Handover
**Date:** 2026-05-29

## Current State

463 runtime tests + 4 WireMock example tests, BUILD SUCCESS. Both repos on `main`.
#85 closed and merged to `casehubio/ledger:main`. 34 blog entries published.

## What Landed This Session

**#85 — AgentSigner SPI (external key distribution):**
- `AgentKeyProvider` → `AgentSigner.sign(actorId, data) → Optional<AgentSignature>` — signing responsibility into the SPI, enabling Vault Transit / Cloud KMS remote signing
- `AbstractCachingAgentSigner<C>` — generic caching base for external providers (`putIfAbsent` not `computeIfAbsent` — deliberate: bucket locking)
- `ConfiguredAgentSigner` — `@DefaultBean`, eager PEM loading, `failedActors` sentinel
- `LedgerPemUtil.parsePublicKey(String)` — public method for PEM-from-string parsing
- `examples/vault-transit-signing/` — Vault Transit remote signing with WireMock tests
- ADR 0014 — extends ADR 0011, records SPI rename rationale
- Deferred: #101 (Vault auth), #102 (Cloud KMS), #103 (rotation invalidation), #104 (InMemoryAgentSigner)

## Immediate Next Step

Run `work-start` from `main`. Best pick: `casehubio/parent#62` (XS · Low — doc update) or `#81` (L · High — agent DID/VC, now unblocked by #85).

## What's Left

- `casehubio/parent#62` — ledger deep-dive SPIs table needs `LedgerMerkleFrontierRepository` row and `persistence-memory/` module entry · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #81 | Agent DID/VC cryptographic identity binding | L | High | Unblocked by #85 |
| #96 | Code-gen for reactive tier (Vert.x codegen style) | XL | High | Not yet: pair count too small |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | Med | Deferred from #85 |
| #102 | Cloud KMS adapters (AWS KMS, GCP, Azure) | L | Med | Deferred from #85 |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-29-mdp01-signing-doesnt-belong-in-the-enricher.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
