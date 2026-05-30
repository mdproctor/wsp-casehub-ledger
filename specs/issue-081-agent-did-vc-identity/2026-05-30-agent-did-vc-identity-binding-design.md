# Agent DID/VC Cryptographic Identity Binding — Design Spec

**Issue:** casehubio/ledger#81  
**Branch:** issue-081-agent-did-vc-identity  
**Date:** 2026-05-30  
**Status:** Approved

---

## Problem

`actorId` on `LedgerEntry` is a convention string (`claude:reviewer@v1`) — not
cryptographically bound to any verified identity. Any caller can claim any `actorId`.
Bilateral entry signing (#85) proves *someone* signed with a specific Ed25519 key, but
it does not prove that the holder of that key is who `actorId` claims.

DID (Decentralized Identifier) binding closes this gap: the agent's signing key is
publicly attested in a DID document, and the DID document asserts via `alsoKnownAs`
that the DID maps to a specific `actorId` string. This enables third-party verification
without trusting the ledger's key store.

---

## Identity Model

Two fields; two distinct concerns.

| Field | Purpose | Format | Trust role |
|---|---|---|---|
| `actorId` | Trust accumulation key | `claude:reviewer@v1` (ADR 0004, unchanged) | Trust scores accumulate here |
| `actorDid` | Cryptographic identity | `did:web:casehub.io:agents:claude:reviewer:v1` | Globally unique; key-verifiable |

`actorId` as trust key is unchanged and correct (ADR 0004). The convention string
encodes meaningful versioning semantics. DID is the cryptographic binding layered on
top — not a replacement.

The DID document at the resolved URI **must** contain:
1. `alsoKnownAs` claim including the `actorId` string — closes the divergence attack
2. A `verificationMethod` whose `publicKeyBytes` match `agentPublicKey` on the entry

The long-term direction (Approach B — DID as `actorId`) is tracked in
casehubio/parent#111 and requires a cross-repo migration once this infrastructure is
established.

---

## Three-SPI Strategy Model

Each concern is independently swappable via `@DefaultBean` / `@Alternative`.

### SPI 1 — `ActorDIDProvider` (api module)

Maps `actorId` → DID URI at write time.

```java
public interface ActorDIDProvider {
    Optional<String> didFor(String actorId);
}
```

| Implementation | CDI | Mechanism |
|---|---|---|
| `NoOpActorDIDProvider` | `@DefaultBean` | Always empty — zero behaviour change for existing consumers |
| `ConfiguredActorDIDProvider` | `@Alternative` | `casehub.ledger.agent-identity.dids."claude:reviewer@v1"=did:web:...` — quoted key escapes colon (GE-20260529-8eb96e) |
| `ScimActorDIDProvider` | `@Alternative` | `GET /scim/v2/Agents?filter=externalId eq "{actorId}"` (deferred: #107) |
| Consumer-provided | `@Alternative` | Any source: Vault, DB, LDAP |

### SPI 2 — `DIDResolver` (api module)

Resolves DID URI → `DIDDocument`.

```java
public interface DIDResolver {
    Optional<DIDDocument> resolve(String did);
}

public record DIDDocument(String id, List<VerificationMethod> verificationMethods, List<String> alsoKnownAs) {}
public record VerificationMethod(String id, String type, byte[] publicKeyBytes) {}
```

| Implementation | CDI | Mechanism |
|---|---|---|
| `NoOpDIDResolver` | `@DefaultBean` | Always empty |
| `WebDIDResolver` | `@Alternative` | `did:web` → HTTPS GET `/.well-known/did.json` or `/path/did.json` |
| `KeyDIDResolver` | `@Alternative` | `did:key` — key bytes decoded from DID itself; no HTTP; primary test/dev resolver |
| Consumer-provided | `@Alternative` | Any DID method |

### SPI 3 — `AgentCredentialValidator` (api module)

Validates the binding claim (the VC layer).

```java
public interface AgentCredentialValidator {
    Optional<CredentialValidationResult> validate(String actorId, String did);
}

public enum CredentialValidationResult {
    VALID, EXPIRED, INVALID_SIGNATURE, ISSUER_UNKNOWN, NOT_FOUND
}
```

| Implementation | CDI | Mechanism |
|---|---|---|
| `NoOpCredentialValidator` | `@DefaultBean` | Always empty — DID document key check is sufficient |
| `JwtVCValidator` | `@Alternative` | W3C VC in JWT format; resolves issuer DID to verify signature (deferred: #108) |
| Consumer-provided | `@Alternative` | X.509/SPIFFE, enterprise PKI |

### Caching base — `AbstractCachingIdentityProvider<C>` (runtime)

Shared abstract base for all three SPIs that involve network/external calls.
Pattern mirrors `AbstractCachingAgentSigner<C>` exactly:

- `ConcurrentHashMap<String, Optional<C>>` — per-actorId or per-DID
- `putIfAbsent` not `computeIfAbsent` (avoids bucket locking under contention)
- `loadContext(key)` template method — empty = not configured (cached); throw = transient (not cached)
- `invalidate(key)` / `invalidateAll()` — CDI `@ObservesAsync KeyRotationEntry` fires `invalidateAll()` by default; implementations may narrow to `invalidate(actorId)`
- Configurable TTL (default 5 minutes for DID documents; indefinite for VC validation results)

---

## Write Path — Enrichment Pipeline

Two new `LedgerEntryEnricher` implementations added in order.

### Enricher ordering — `LedgerEnricherPipeline` fix required

`LedgerEnricherPipeline.enrich()` currently iterates enrichers in unspecified CDI
order. `ActorIdentityValidationEnricher` depends on two prior enrichers having run:
`ActorDIDEnricher` (sets `actorDid`) and `AgentSignatureEnricher` (sets
`agentPublicKey`). `LedgerEnricherPipeline` must be updated to sort enrichers by
`@Priority` before iterating.

```java
// LedgerEnricherPipeline.enrich() — updated
enrichers.stream()
    .sorted(Comparator.comparingInt(e -> {
        Priority p = AnnotationUtils.findAnnotation(e.getClass(), Priority.class);
        return p != null ? p.value() : 100;
    }))
    .forEach(e -> { try { e.enrich(entry); } catch (Exception ex) { log.warn(...); } });
```

Assigned priorities (existing enrichers gain `@Priority` with no behaviour change at
their current call sites):

| Enricher | `@Priority` |
|---|---|
| `TraceIdEnricher` | 10 |
| `AgentSignatureEnricher` | 20 |
| `ProvenanceCaptureEnricher` | 30 |
| `ActorDIDEnricher` | 40 |
| `ActorIdentityValidationEnricher` | 50 |

### `ActorDIDEnricher` (`@Priority(40)`)

Runs for every entry; zero cost when no DID configured.

```
ActorDIDProvider.didFor(actorId)
  → empty  → skip (actorDid stays null)
  → did    → LedgerEntry.actorDid = did
```

### `ActorIdentityValidationEnricher` (`@Priority(50)`)

Runs only when `actorDid` is non-null on the entry being persisted.

```
cache lookup (actorId → BindingState)
  → VALID (cached)   → skip
  → absent/stale     →
      DIDResolver.resolve(actorDid)
        → empty        → result = DID_UNRESOLVABLE
        → DIDDocument  →
            alsoKnownAs contains actorId?
              → no   → result = IDENTITY_MISMATCH
              → yes  →
                  agentPublicKey == null?
                    → yes → result = UNSIGNED
                             (DID + alsoKnownAs verified; no key to cross-check)
                    → no  →
                        agentPublicKey matches a verificationMethod.publicKeyBytes?
                          → no  → result = KEY_MISMATCH
                          → yes →
                              AgentCredentialValidator.validate(actorId, actorDid)
                                → empty/VALID → result = VALID
                                → other       → result = CREDENTIAL_INVALID
      write ActorIdentityBindingEntry
      fire AgentIdentityValidatedEvent (CDI, async)
      cache result
```

**Validation mode** — `casehub.ledger.agent-identity.validation-mode`:

| Value | On non-VALID result |
|---|---|
| `WARN` (default) | Entry persisted; CDI event fired; result recorded in `ActorIdentityBindingEntry` |
| `ENFORCE` | `LedgerIdentityViolationException` thrown; entry not persisted |

**Cache invalidation:** `@ObservesAsync KeyRotationEntry` CDI event calls `invalidate(actorId)`. On next write for that actor, full resolution runs again and a new `ActorIdentityBindingEntry` is written.

---

## Read Path — Verification Service

`AgentIdentityVerificationService` (`@ApplicationScoped`) — symmetric read-side
counterpart to `AgentSignatureVerificationService`.

```java
AgentIdentityVerificationResult verifyIdentityBinding(LedgerEntry entry);
```

```
entry.actorDid == null         → UNVERIFIABLE
entry.agentPublicKey == null   → UNSIGNED
otherwise:
  DIDResolver.resolve(entry.actorDid)
    → empty           → DID_UNRESOLVABLE
    → DIDDocument     →
        alsoKnownAs contains entry.actorId?
          → no  → IDENTITY_MISMATCH
          → yes →
              any verificationMethod.publicKeyBytes == entry.agentPublicKey?
                → yes → VALID
                → no  → KEY_MISMATCH
```

```java
public enum AgentIdentityVerificationResult {
    VALID,              // key matches DID document; alsoKnownAs confirmed
    UNVERIFIABLE,       // no actorDid on entry
    UNSIGNED,           // no agentPublicKey to cross-check
    DID_UNRESOLVABLE,   // resolver returned empty
    IDENTITY_MISMATCH,  // DID document alsoKnownAs does not include actorId
    KEY_MISMATCH        // key no longer in DID document (rotated since entry written)
}
```

`KEY_MISMATCH` mirrors `VerificationResult.SUSPECT` — the entry content is intact;
the key was valid when written. The `KeyRotationEntry` chain provides historical context.

The read path uses its own `AbstractCachingIdentityProvider<DIDDocument>` cache,
independent of the write path, since it may be called on entries with arbitrary
`actorDid` values from the past.

Reactive counterpart (`Uni<AgentIdentityVerificationResult>`) deferred to #109,
excluded via `LedgerProcessor` ExcludedTypeBuildItem when reactive.enabled=false.

---

## `ActorIdentityBindingEntry` Subclass and Schema

### Subclass

```java
@Entity
@DiscriminatorValue("IDENTITY_BINDING")
@Table(name = "actor_identity_binding")
public class ActorIdentityBindingEntry extends LedgerEntry {
    @Column(name = "bound_did", nullable = false)
    public String boundDid;

    @Enumerated(EnumType.STRING)
    @Column(name = "validation_result", nullable = false)
    public AgentIdentityValidationResult validationResult;

    @Column(name = "also_known_as_verified", nullable = false)
    public boolean alsoKnownAsVerified;

    @Column(name = "key_match_verified", nullable = false)
    public boolean keyMatchVerified;

    @Column(name = "verified_key_ref")
    public String verifiedKeyRef;

    @Enumerated(EnumType.STRING)
    @Column(name = "credential_result")
    public CredentialValidationResult credentialResult;

    @Column(name = "did_method", length = 32)
    public String didMethod;
}
```

`subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8))` — identical to
`KeyRotationEntry`. Both entry types form a unified actor lifecycle sequence: key
rotations and identity binding events are ordered together per actor.

### V1008 Migration

```sql
-- Column on base table
ALTER TABLE ledger_entry ADD COLUMN actor_did TEXT;

-- Join table for ActorIdentityBindingEntry
CREATE TABLE actor_identity_binding (
    id                     UUID PRIMARY KEY REFERENCES ledger_entry(id),
    bound_did              TEXT NOT NULL,
    validation_result      VARCHAR(32) NOT NULL,
    also_known_as_verified BOOLEAN NOT NULL DEFAULT FALSE,
    key_match_verified     BOOLEAN NOT NULL DEFAULT FALSE,
    verified_key_ref       TEXT,
    credential_result      VARCHAR(32),
    did_method             VARCHAR(32),
    CONSTRAINT chk_identity_result CHECK (
        validation_result IN ('VALID','DID_UNRESOLVABLE','IDENTITY_MISMATCH',
                              'KEY_MISMATCH','CREDENTIAL_INVALID','UNSIGNED')
    )
);
```

### Repository SPI

```java
public interface ActorIdentityBindingRepository {
    Optional<ActorIdentityBindingEntry> latestBindingFor(String actorId);
    List<ActorIdentityBindingEntry> bindingHistoryFor(String actorId);
}
```

---

## Module Placement

### `api` module

```
api/src/main/java/io/casehub/ledger/api/
├── model/
│   ├── AgentIdentityValidationResult.java
│   └── CredentialValidationResult.java
└── spi/
    ├── identity/
    │   ├── ActorDIDProvider.java
    │   ├── AgentCredentialValidator.java
    │   ├── DIDDocument.java
    │   └── VerificationMethod.java
    └── resolve/
        └── DIDResolver.java
```

### `runtime` module

```
runtime/src/main/java/io/casehub/ledger/runtime/
├── model/
│   └── ActorIdentityBindingEntry.java
├── repository/
│   ├── ActorIdentityBindingRepository.java
│   └── jpa/JpaActorIdentityBindingRepository.java
└── service/
    └── identity/
        ├── AbstractCachingIdentityProvider.java
        ├── ActorDIDEnricher.java
        ├── ActorIdentityValidationEnricher.java
        ├── AgentIdentityValidatedEvent.java
        ├── AgentIdentityVerificationService.java
        ├── ConfiguredActorDIDProvider.java
        ├── KeyDIDResolver.java
        ├── NoOpActorDIDProvider.java
        ├── NoOpCredentialValidator.java
        ├── NoOpDIDResolver.java
        └── WebDIDResolver.java
```

### `persistence-memory` module

```
InMemoryActorIdentityBindingRepository.java   (@Alternative @Priority(1))
```

### Config

`casehub.ledger.agent-identity.*` under `LedgerConfig`:

| Key | Type | Default | Description |
|---|---|---|---|
| `validation-mode` | enum | `WARN` | `WARN` or `ENFORCE` |
| `dids."<actorId>"` | String | — | DID URI per actorId (ConfiguredActorDIDProvider) |
| `did-resolver.web.timeout-ms` | int | 5000 | HTTP timeout for WebDIDResolver |
| `did-resolver.cache-ttl-minutes` | int | 5 | DID document cache TTL |

---

## Testing Strategy

### Unit tests (pure Java, no Quarkus)
- `DIDDocumentTest`, `VerificationMethodTest` — construction, null guards, defensive copy
- `ActorDIDEnricherTest` — all paths: no DID, DID populated, provider throws (non-fatal)
- `ActorIdentityValidationEnricherTest` — all result paths; cache hit; CDI event invalidation; WARN vs ENFORCE mode
- `AgentIdentityVerificationServiceTest` — all `AgentIdentityVerificationResult` paths
- `AbstractCachingIdentityProviderTest` — TTL, `invalidate`, `invalidateAll`, concurrent `putIfAbsent` semantics
- `WebDIDResolverTest` — WireMock: success, 404, malformed JSON, timeout; `did:web` path encoding
- `KeyDIDResolverTest` — valid `did:key:z6Mk...` roundtrip; bad multibase prefix; key matches VerificationMethod

### Integration tests (`@QuarkusTest`, H2)
- `ActorIdentityBindingEntryIT` — full write path: DID populated, binding entry written once, second write uses cache, KeyRotationEntry event triggers re-resolution and new binding entry
- `ActorIdentityBindingRepositoryIT` — `latestBindingFor`, `bindingHistoryFor` ordering
- `AgentIdentityVerificationServiceIT` — `did:key` + `ConfiguredAgentSigner` roundtrip: sign → bind → verify VALID; rotate key → KEY_MISMATCH
- `InMemoryActorIdentityBindingRepositoryIT` — zero datasource path

### Test helper pattern
All ITs use `KeyDIDResolver` + `ConfiguredActorDIDProvider` together:
generate Ed25519 key pair → encode as `did:key` → set `alsoKnownAs: ["claude:test-agent@v1"]`
in document → zero HTTP, zero external config, full end-to-end coverage.

---

## Deferred Issues

| Issue | Description |
|---|---|
| casehubio/ledger#107 | `ScimActorDIDProvider` — enterprise SCIM2 integration |
| casehubio/ledger#108 | `JwtVCValidator` — W3C VC JWT credential validation |
| casehubio/ledger#109 | `ReactiveAgentIdentityVerificationService` — reactive parity |
| casehubio/parent#111 | Cross-repo `actorId` → DID migration (Approach B, long-term) |

---

## Protocol

PP-20260530-bf919d — `scim2-agent-identity-lookup` (casehubio/parent)  
Governs: SCIM2 Agent endpoint lookup pattern for all casehub components.

---

## ADR Required

ADR 0015 — Agent DID/VC identity binding model. Must cover:
- Two-field model rationale (actorId trust key unchanged; actorDid cryptographic binding)
- `alsoKnownAs` verification requirement (closes divergence attack)
- `ActorIdentityBindingEntry` as subclass not supplement (canonical bytes, Merkle participation)
- `AbstractCachingIdentityProvider` caching model (TTL + CDI event invalidation)
- Validation mode: WARN default, ENFORCE opt-in
- Deferred: SCIM2 provider, JWT VC validator, reactive service, actorId→DID migration
