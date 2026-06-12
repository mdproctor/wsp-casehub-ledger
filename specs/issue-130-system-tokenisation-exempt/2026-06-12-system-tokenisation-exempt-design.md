# Exempt Non-Human Actors from Tokenisation

**Issue:** casehubio/ledger#130
**Date:** 2026-06-12
**Status:** Design — revision 2

## Problem

`JpaLedgerEntryRepository.save()` and `saveAttestation()` unconditionally tokenise `actorId` / `attestorId` via `ActorIdentityProvider.tokenise()`. The in-memory repository mirrors this behaviour. Under `InternalActorIdentityProvider`, the original string is replaced with a UUID token stored in the `ActorIdentity` table.

GDPR pseudonymisation targets natural persons. Non-human actors (`ActorType.SYSTEM`, `ActorType.AGENT`) are not natural persons. Tokenising them:

- Serves no privacy purpose
- Breaks client-side identity comparison after read-back (stored value is a UUID token, not the original string)
- Wastes `ActorIdentity` table rows
- Forces consumers to use the attestor-scoped query path (`findAttestationsByAttestorIdAndCapabilityTag` with `tokeniseForQuery`) even when the entry-scoped query is more natural

## Design

### SPI signature change

The tokenisation decision is a policy concern — it belongs in `ActorIdentityProvider`, not scattered across repository call sites. casehub-work already has its own `JpaWorkItemLedgerEntryRepository.save()` (with no tokenisation call today). Putting the guard at call sites means every current and future repository implementation must independently discover the exemption rule. The SPI is the single owner of tokenisation policy.

```java
// Before
String tokenise(String rawActorId);

// After
String tokenise(String rawActorId, ActorType actorType);
```

`tokeniseForQuery`, `resolve`, and `erase` do not need `ActorType` — they operate on existing mappings or return raw values when no mapping exists. The type is only relevant at the write-path decision point.

### Implementation rule: tokenise HUMAN only (positive selection)

`InternalActorIdentityProvider.tokenise()` tokenises when `actorType == ActorType.HUMAN` or `actorType == null`. All other types return `rawActorId` unchanged.

Positive selection is safer than negative exclusion (`!= SYSTEM`): if a new `ActorType` is added in the future, it defaults to not-tokenised and must explicitly opt in. Negative exclusion silently tokenises new non-person types.

**Null handling:** `LedgerEntry.actor_type` is nullable in the schema (V1000 — `VARCHAR(20)` with no NOT NULL). `LedgerAttestation.attestor_type` is NOT NULL. When `actorType` is null, the safe default is to tokenise — could be human, better to pseudonymise unnecessarily than miss PII.

```java
// In InternalActorIdentityProvider:
public String tokenise(final String rawActorId, final ActorType actorType) {
    if (rawActorId == null) {
        return null;
    }
    if (actorType != null && actorType != ActorType.HUMAN) {
        return rawActorId;
    }
    // null or HUMAN → tokenise
    // ... existing token lookup/creation logic
}
```

`PassThroughActorIdentityProvider.tokenise()` adds the parameter but ignores it — it already returns unchanged regardless.

### Repository call-site changes

Call sites become simpler (no conditional):

```java
// Before (JpaLedgerEntryRepository.save):
if (entry.actorId != null) {
    entry.actorId = actorIdentityProvider.tokenise(entry.actorId);
}

// After:
entry.actorId = actorIdentityProvider.tokenise(entry.actorId, entry.actorType);

// Before (JpaLedgerEntryRepository.saveAttestation):
if (attestation.attestorId != null) {
    attestation.attestorId = actorIdentityProvider.tokenise(attestation.attestorId);
}

// After:
attestation.attestorId = actorIdentityProvider.tokenise(
        attestation.attestorId, attestation.attestorType);
```

Same pattern in `InMemoryLedgerEntryRepository`. The null-guard moves into the SPI (already there — `InternalActorIdentityProvider` returns null for null input).

### Affected implementations

| Class | Change |
|---|---|
| `ActorIdentityProvider` (SPI) | Add `ActorType` parameter to `tokenise()` |
| `InternalActorIdentityProvider` | Add HUMAN-only guard |
| `PassThroughActorIdentityProvider` | Add parameter (ignored) |
| `LedgerPrivacyProducer` | No change (constructs implementations, doesn't call `tokenise`) |
| `JpaLedgerEntryRepository` | Pass `actorType` to `tokenise()`, remove null-guard |
| `InMemoryLedgerEntryRepository` | Same |
| `JpaCrossTenantLedgerEntryRepository` | No change (only calls `tokeniseForQuery`, not `tokenise`) |
| Consumer `ActorIdentityProvider` impls | Add `ActorType` parameter to `tokenise()` (compile error until updated) |
| Test mocks / `@InjectMock` | Update to new signature |

## Tests

1. **SPI unit — HUMAN tokenised:** `InternalActorIdentityProvider.tokenise("alice@example.com", ActorType.HUMAN)` returns a UUID token
2. **SPI unit — SYSTEM not tokenised:** `InternalActorIdentityProvider.tokenise("system:health-check", ActorType.SYSTEM)` returns `"system:health-check"`
3. **SPI unit — AGENT not tokenised:** `InternalActorIdentityProvider.tokenise("claude:tarkus-reviewer@v1", ActorType.AGENT)` returns the raw string
4. **SPI unit — null actorType tokenised:** `InternalActorIdentityProvider.tokenise("unknown@example.com", null)` returns a UUID token (safe default)
5. **JPA save — SYSTEM entry:** save with `actorType = SYSTEM`, verify `actorId` stored raw
6. **JPA saveAttestation — SYSTEM attestation:** save with `attestorType = SYSTEM`, verify `attestorId` stored raw
7. **In-memory save + saveAttestation:** same verification for both
8. **Regression — HUMAN:** existing tokenisation tests confirm HUMAN actors still tokenised
9. **findByActorId round-trip for SYSTEM:** save a SYSTEM entry with raw actorId, then `findByActorId` with the same raw string — query works without tokenisation lookup
10. **Erasure no-op for SYSTEM:** `LedgerErasureService.erase("system:health-check")` returns `mappingFound=false` (no mapping was ever created)
11. **DefaultOutcomeRecorder path:** with `casehub.ledger.outcome.default-attestor-type=SYSTEM`, verify the attestation created via `record()` stores the raw attestorId (traces through `OutcomeRecordSaveService` → `saveAttestation()`)
