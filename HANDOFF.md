# CaseHub Ledger — Session Handover
**Date:** 2026-06-01

## Current State

546 runtime tests + 42 persistence-memory tests, BUILD SUCCESS. Both repos on `main`.
Identity extraction from ledger to casehub-platform-identity **fully complete** — Phase 1 + Phase 2 done, work-end closed, parent#135 closed, DESIGN.md updated, blog published.

## What Landed This Session

- Platform ownership rule formalised (PP-20260531-dd7062 in casehubio/parent)
- casehub-platform-identity created (platform#52) — all identity SPIs, model, events, implementations
- Ledger migrated to consumer (ledger#112) — config prefix casehub.ledger.agent-identity.* → casehub.identity.*
- AgentIdentityVerificationService + ReactiveAgentIdentityVerificationService moved to platform with primitive signatures; ledger thin adapters (ledger#113, platform#53)
- IdentityCacheInvalidator bridge pattern (PP-20260601-f7c2b2)
- Garden: GE-20260601-b9a489 (Quarkus extension descriptor validation reads installed .m2, not source)
- DESIGN.md: platform ownership KDDs added, identity tracker row updated

## Immediate Next Step

Pick next issue. `/work` to start:
- `#111` — minor review findings from #107 (S · Low, fastest win)
- `#108` — JwtVCValidator W3C VC JWT validation (M · Med) — belongs in casehub-platform-identity

## What's Left

- `#111` — minor review findings (requireHttps field, authToken @PostConstruct warn, enricher unit test) · S · Low
- `#110` — ScimDIDResolver — likely close as won't-fix now that SCIM lives in platform · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #111 | Minor review findings from #107 | S | Low | Fastest to close |
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | Med | Land in casehub-platform-identity |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | Med | Deferred from #85 |
| #102 | Cloud KMS adapters (AWS KMS, GCP, Azure) | L | Med | Deferred from #85 |
| #96 | Reactive tier code-gen (Vert.x codegen style) | XL | High | Largest remaining gap |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-06-01-mdp01-the-infrastructure-that-found-its-address.md` |
| casehub-identity deep-dive | `casehubio/parent docs/repos/casehub-identity.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
