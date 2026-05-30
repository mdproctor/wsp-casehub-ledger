# CaseHub Ledger — Session Handover
**Date:** 2026-05-30

## Current State

523 runtime tests + 42 persistence-memory tests, BUILD SUCCESS. Both repos on `main`.
#81 closed and merged to `casehubio/ledger:main`. 35 blog entries published.

## What Landed This Session

**#81 — Agent DID/VC cryptographic identity binding:**
- Three-SPI strategy: `ActorDIDProvider`, `DIDResolver`, `AgentCredentialValidator` — all no-op by default
- `AbstractCachingIdentityProvider<C>` — TTL-capable caching base with atomic conditional eviction (`cache.remove(key, existing)` before miss path — not `putIfAbsent`)
- `ActorDIDEnricher` @Priority(40) + `ActorIdentityValidationEnricher` @Priority(50) — write-path validation pipeline
- `LedgerEnricherPipeline` now sorts by `InjectableBean.getPriority()` (Arc-specific, proxy-safe — not `getClass().getAnnotation()`)
- `ActorIdentityBindingEntry` — `LedgerEntry` subclass (V1008); `@ObservesAsync` + `REQUIRES_NEW` observer for persistence; `@EntityListeners` enforcement listener for ENFORCE mode
- `AgentIdentityVerificationService` — read-path DID↔key verification
- ADR 0015, protocol PP-20260530-d4d294 (enricher @Priority mandate), `alternative-extension-patterns.md` Pattern C
- 7 garden entries, SCIM2 protocol PP-20260530-bf919d captured in casehubio/parent
- Deferred: #107 ScimActorDIDProvider, #108 JwtVCValidator, #109 reactive parity, parent#111 actorId→DID migration

## Immediate Next Step

Run `work-start` from `main`. Best pick: `casehubio/parent#62` (XS · Low — doc update, long deferred) or #96 (XL · High — reactive code-gen, now the largest remaining gap).

## What's Left

- `casehubio/parent#62` — ledger deep-dive SPIs table needs `LedgerMerkleFrontierRepository` row and `persistence-memory/` module entry · XS · Low
- `casehubio/parent#107` — raggable SCIM2 agent identity doc + PLATFORM.md update · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #107 | ScimActorDIDProvider — enterprise SCIM2 integration | M | Med | Deferred from #81 |
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | Med | Deferred from #81; validUntil-bounded TTL gap |
| #96 | Code-gen for reactive tier (Vert.x codegen style) | XL | High | Largest remaining gap; pair count now sufficient |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | Med | Deferred from #85 |
| #102 | Cloud KMS adapters (AWS KMS, GCP, Azure) | L | Med | Deferred from #85 |
| #109 | ReactiveAgentIdentityVerificationService | S | Low | Deferred from #81 |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-30-mdp01-did-proves-who.md` |
| ADR 0015 | `docs/adr/0015-agent-did-vc-identity-binding.md` (project repo) |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
