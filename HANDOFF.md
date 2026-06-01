# CaseHub Ledger — Session Handover
**Date:** 2026-06-01

## Current State

546 runtime tests + 42 persistence-memory tests, BUILD SUCCESS. Both repos on `main`.
Identity extraction from ledger to casehub-platform-identity **complete** (Phase 1 + Phase 2).
All ledger identity infrastructure is platform-owned; ledger holds thin adapters and domain-specific enrichers.

## What Landed This Session

**Phase 1 (platform#52, ledger#112):** SPIs, model types, CDI events, implementations (NoOp*, KeyDIDResolver, WebDIDResolver, ConfiguredActorDIDProvider, ScimActorDIDProvider, AbstractCachingIdentityProvider) moved to platform.

**Phase 2 (platform#53, ledger#113):** AgentIdentityVerificationService + ReactiveAgentIdentityVerificationService moved to platform with primitive signatures. Ledger retains thin adapters extracting fields from LedgerEntry.

**Also this session:** New platform ownership rule (PP-20260531-dd7062) added to casehubio/parent.

## Immediate Next Step

Pick the next issue. Best picks:
- `#111` — minor review findings from #107 (S · Low, fastest win)
- `#108` — JwtVCValidator (M · Med) — should land in casehub-platform-identity

## What's Left

- `#111` — minor review findings from #107 (test constructor requireHttps field, authToken @PostConstruct warn, enricher unit test) · S · Low
- `#110` — ScimDIDResolver — consider closing as won't-fix (SCIM lookup now fully in platform) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #111 | Minor review findings from #107 | S | Low | Fastest to close |
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | Med | Should land in casehub-platform-identity |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | Med | Deferred from #85 |
| #102 | Cloud KMS adapters (AWS KMS, GCP, Azure) | L | Med | Deferred from #85 |
| #96 | Reactive tier code-gen (Vert.x codegen style) | XL | High | Largest remaining gap |

## References

| What | Path |
|------|------|
| Previous handover | `git show HEAD~1:HANDOFF.md` |
| Tracking issue (closed) | `casehubio/parent#135` |
| casehub-identity deep-dive | `casehubio/parent docs/repos/casehub-identity.md` |
