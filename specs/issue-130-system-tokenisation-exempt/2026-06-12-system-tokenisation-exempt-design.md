# Exempt ActorType.SYSTEM from Tokenisation

**Issue:** casehubio/ledger#130
**Date:** 2026-06-12
**Status:** Design

## Problem

`JpaLedgerEntryRepository.save()` and `saveAttestation()` unconditionally tokenise `actorId` / `attestorId` via `ActorIdentityProvider.tokenise()`. The in-memory repository mirrors this behaviour. Under `InternalActorIdentityProvider`, the original string is replaced with a UUID token stored in the `ActorIdentity` table.

System actors (`ActorType.SYSTEM`) are not natural persons. Tokenising them:

- Serves no privacy purpose (GDPR pseudonymisation targets natural persons)
- Breaks client-side identity comparison after read-back (stored value is a UUID token, not the original string)
- Wastes `ActorIdentity` table rows
- Forces consumers to use the attestor-scoped query path (`findAttestationsByAttestorIdAndCapabilityTag` with `tokeniseForQuery`) even when the entry-scoped query is more natural

## Design

Add an `actorType != ActorType.SYSTEM` guard to the existing null-check at each tokenisation call site. Four sites total:

| Repository | Method | Current guard | New guard |
|---|---|---|---|
| `JpaLedgerEntryRepository` | `save()` | `actorId != null` | `actorId != null && actorType != SYSTEM` |
| `JpaLedgerEntryRepository` | `saveAttestation()` | `attestorId != null` | `attestorId != null && attestorType != SYSTEM` |
| `InMemoryLedgerEntryRepository` | `save()` | `actorId != null` | `actorId != null && actorType != SYSTEM` |
| `InMemoryLedgerEntryRepository` | `saveAttestation()` | `attestorId != null` | `attestorId != null && attestorType != SYSTEM` |

### What does NOT change

- `ActorIdentityProvider` SPI — no signature change
- Query paths — `tokeniseForQuery` behaviour unchanged
- `LedgerEntry` / `LedgerAttestation` models
- No database migration required

### Backward compatibility with existing data

Existing SYSTEM entries with tokenised `actorId`/`attestorId` values continue to work. `tokeniseForQuery` returns the token if a mapping exists, or the raw value if not. Queries handle both forms transparently.

## Tests

1. **JPA save — SYSTEM entry**: save with `actorType = SYSTEM`, verify `actorId` is stored as the raw string
2. **JPA saveAttestation — SYSTEM attestation**: save with `attestorType = SYSTEM`, verify `attestorId` is stored raw
3. **In-memory save — SYSTEM entry**: same verification
4. **In-memory saveAttestation — SYSTEM attestation**: same verification
5. **Regression — HUMAN/AGENT**: existing tokenisation tests confirm non-SYSTEM actors are still tokenised
