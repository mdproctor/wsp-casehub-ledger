# CaseHub Ledger — Session Handover
**Date:** 2026-05-31

## Current State

546 runtime tests + 42 persistence-memory tests, BUILD SUCCESS. Both repos on `main`.
#81 (DID/VC binding), #103 (AgentKeyRotatedEvent), #107 (ScimActorDIDProvider), #109 (ReactiveAgentIdentityVerificationService) all closed and merged to `casehubio/ledger:main` as 4 clean squash commits.

## What Landed This Session

**#103 + #107 + #109 — CDI event-driven identity cache invalidation + SCIM2 enterprise DID provider + reactive bridge:**
- `AgentKeyRotatedEvent` CDI event — replaces direct `identityEnricher.invalidate()` call in `KeyRotationService`; reactive path fires via `fireAsync()`
- `ScimActorDIDProvider @Alternative` — SCIM2 HTTPS endpoint, TTL cache (`Optional<String>` config after SmallRye SRCFG00040), `@Observes AgentKeyRotatedEvent` invalidation, WireMock CDI IT
- `ReactiveAgentIdentityVerificationService @DefaultBean @Unremovable` — pure bridge, no Hibernate Reactive dep, always active
- Protocols: PP-20260531-2c9f00 (`@Unremovable` rule), PP-20260531-15237d (identity cache rotation invalidation)
- Garden: 4 entries (ARC @Observes abstract class, SmallRye @WithDefault("") SRCFG00040, CDI sync/async channels, cache-eviction stub-swap technique)
- casehubio/parent#107 closed: `docs/integration/scim2-agent-identity.md` + PLATFORM.md updated
- Follow-on: #110 (ScimDIDResolver @Alternative — deliberate design decision deferred), #111 (minor review findings)

## Immediate Next Step

Clean main. Start next issue. Best picks:
- `#111` — minor review findings (S · Low, fastest win)
- `#112` — Phase 1 identity extraction (blocks on parent#133 existing first)
- `#108` — JwtVCValidator (M · Med, but will move to casehub-identity in Phase 1 — do after #112)

## Cross-Module

**We're blocking:**
- `casehubio/platform#52` — needs ledger to confirm extraction list before creating casehub-identity module

**Blocked by:**
- `casehubio/platform#52` — must publish casehub-identity before ledger#112 can land

## What's Left

- `#111` — minor review findings from #107 · S · Low
- `#110` — ScimDIDResolver @Alternative — subsumed by identity extraction; likely becomes part of casehub-identity · M · Med

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #111 | Minor review findings from #107 (batch fix) | S | Low | Fastest to close |
| #112 | Migrate identity infrastructure to casehub-identity (Phase 1) | L | Med | Blocks on parent#133 |
| #113 | Refactor AgentIdentityVerificationService to primitive signature (Phase 2 prep) | M | Med | Depends on #112 |
| #108 | JwtVCValidator — W3C VC JWT credential validation | M | Med | Will land in casehub-identity, not ledger — coordinate with parent#133 |
| #101 | Vault AppRole/OIDC auth for VaultTransitAgentSigner | M | Med | Deferred from #85 |
| #102 | Cloud KMS adapters (AWS KMS, GCP, Azure) | L | Med | Deferred from #85 |
| #96 | Reactive tier code-gen (Vert.x codegen style) | XL | High | Largest remaining gap |

## References

| What | Path |
|------|------|
| Latest blog | `blog/2026-05-31-mdp01-cdi-event-scim-enterprise-identity.md` |
| Design spec | `docs/specs/2026-05-31-scim-actor-did-provider-design.md` (project repo) |
| SCIM2 integration doc | `casehubio/parent docs/integration/scim2-agent-identity.md` |
| Previous handover | `git show HEAD~1:HANDOFF.md` |
