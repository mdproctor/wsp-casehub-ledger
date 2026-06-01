# CaseHub Ledger — Session Handover
**Date:** 2026-06-01

## Current State

546 runtime tests + 42 persistence-memory tests, BUILD SUCCESS. Both repos on `main`.
Phase 1 identity extraction complete: casehub-platform-identity created in casehubio/platform (platform#52 closed), ledger migrated to consumer (ledger#112 closed). All doc updates landed across both repos and casehubio/parent.

## What Landed This Session

**Identity infrastructure extracted to casehub-platform-identity (platform#52, ledger#112):**
- New `casehub-platform-identity` module in casehubio/platform — all DID/VC SPIs, resolvers, validators, no-op defaults, SCIM provider, cache base, CDI events
- Ledger api: removed ActorDIDProvider, DIDResolver, AgentCredentialValidator, DIDDocument, VerificationMethod, IdentityVerificationResult, CredentialValidationResult, IdentityBindingStatus
- Ledger runtime: removed NoOp*, KeyDIDResolver, WebDIDResolver, ConfiguredActorDIDProvider, ScimActorDIDProvider, ScimAgentResource, AbstractCachingIdentityProvider, AgentIdentityValidated/ViolationEvent
- Added `IdentityCacheInvalidator` — bridges AgentKeyRotatedEvent → platform cache invalidation
- Config migration: `casehub.ledger.agent-identity.*` → `casehub.identity.*` (except validation-mode)
- Docs: platform CLAUDE.md, ledger CLAUDE.md, DESIGN.md, DESIGN-capabilities.md, parent PLATFORM.md, new casehub-identity.md deep-dive, updated casehub-ledger.md

## Immediate Next Step

Pick the next issue:
- `#111` — minor review findings from #107 (S · Low, fastest win)
- `#108` — JwtVCValidator W3C VC JWT validation (M · Med) — should now live in casehub-platform-identity, not ledger; coordinate with platform#53 scope

## Cross-Module

**Blocked by:**
- `casehubio/platform#53` — Phase 2: move AgentIdentityVerificationService after ledger#113 refactors its signature

## What's Left

- `#111` — minor review findings from #107 (test constructor requireHttps field, authToken @PostConstruct warn, enricher unit test) · S · Low
- `#113` — refactor AgentIdentityVerificationService to primitive signature (Phase 2 prep) · M · Med
- `#110` — ScimDIDResolver @Alternative — subsumed by identity extraction; likely closes as won't-fix or becomes platform#53 scope · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #111 | Minor review findings from #107 (batch fix) | S | Low | Fastest to close |
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | Med | Should land in casehub-platform-identity |
| #113 | Refactor AgentIdentityVerificationService to primitive signature | M | Med | Unblocks platform#53 |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | Med | Deferred from #85 |
| #102 | Cloud KMS adapters (AWS KMS, GCP, Azure) | L | Med | Deferred from #85 |
| #96 | Reactive tier code-gen (Vert.x codegen style) | XL | High | Largest remaining gap |

## References

| What | Path |
|------|------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| casehub-identity deep-dive | `casehubio/parent docs/repos/casehub-identity.md` |
| Tracking issue | `casehubio/parent#135` |
