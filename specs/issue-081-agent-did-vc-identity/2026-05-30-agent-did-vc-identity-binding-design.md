# Agent DID/VC Cryptographic Identity Binding — Design Spec

**Issue:** casehubio/ledger#81
**Branch:** issue-081-agent-did-vc-identity
**Date:** 2026-05-30
**Status:** Approved — v2 (post-review corrections)

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

### SPI placement rule

SPIs whose implementations take and return **simple value types** (String, records, enums)
and are expected to be implemented by consumer modules belong in the **api module**. SPIs
tightly coupled to runtime types (e.g. `AgentSigner` → `AgentSignature`) stay in the
**runtime module**. ADR 0015 documents this split. The three DID/VC SPIs below all
qualify for the api module.

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
| `ScimActorDIDProvider` | `@Alternative` | `GET /scim/v2/Agents?filter=externalId eq "{actorId}"` (deferred: #107, protocol PP-20260530-bf919d) |
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
| `WebDIDResolver` | `@Alternative` | `did:web` → HTTPS GET `/.well-known/did.json` or `/path/did.json`; see Security section |
| `KeyDIDResolver` | `@Alternative` | `did:key` — key bytes decoded from DID itself, no HTTP; standards-compliant only (no `alsoKnownAs` — use `TestDIDResolver` in tests) |
| `TestDIDResolver` | test-only | Map-backed stub: `Map<String, DIDDocument>` — set arbitrary documents including `alsoKnownAs`; primary test helper |
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

Shared abstract base for implementations with network/external call overhead.
Pattern mirrors `AbstractCachingAgentSigner<C>` with TTL extension:

```java
// Time-aware cache entry — enables TTL without external library
record CacheEntry<C>(Optional<C> value, Instant expiresAt) {
    boolean isExpired() { return Instant.now().isAfter(expiresAt); }
}
```

- `ConcurrentHashMap<String, CacheEntry<C>>` — per-key TTL-capable cache
- `loadContext(key)` template method: empty = not configured (cached); throw = transient (not cached)
- `invalidate(key)` / `invalidateAll()` hooks
- Configurable TTL; default varies by use (see below)

**TTL eviction algorithm** — `putIfAbsent` alone is insufficient for expired-entry replacement.
When a key is present but expired, `putIfAbsent` is a no-op (the key exists). The correct
eviction path uses atomic conditional remove:

```
// On each cache lookup for 'key':
CacheEntry<C> existing = cache.get(key);
if (existing != null && existing.isExpired()) {
    cache.remove(key, existing);   // atomic: only removes if value == existing (no lost updates)
    existing = null;               // fall through to miss path
}
if (existing == null) {
    Optional<C> loaded = loadContext(key);   // template method
    CacheEntry<C> fresh = new CacheEntry<>(loaded, Instant.now().plus(ttl));
    cache.putIfAbsent(key, fresh);           // safe: two threads racing → first wins, both correct
    existing = cache.get(key);               // read back the winner
}
return existing.value();
```

`AbstractCachingIdentityProviderTest` must include a clock-advancing TTL expiry test that
verifies re-resolution occurs after expiry and that the expired entry is not returned.

**DID document cache TTL:** configurable, default 5 minutes. Write-path and read-path
caches are independent instances with the same TTL.

**VC validation cache TTL:** configurable, default 1 hour (NOT indefinite — a cached
VALID result must not outlive the VC's own `validUntil` field). When `JwtVCValidator`
(#108) is implemented, the VALID cache TTL must be bounded by
`min(configuredTtl, vc.validUntil - now())`. `EXPIRED` results must not be cached.

---

## Write Path — Enrichment Pipeline

### `LedgerEnricherPipeline` ordering fix

`LedgerEnricherPipeline.enrich()` currently iterates enrichers in unspecified CDI order.
`ActorIdentityValidationEnricher` depends on `ActorDIDEnricher` (sets `actorDid`) and
`AgentSignatureEnricher` (sets `agentPublicKey`) having already run. The pipeline must
sort by `@Priority` before iterating.

The correct Quarkus/Arc approach uses `InjectableBean.getPriority()` via `h.getBean()`
— proxy-safe, reads from CDI bean metadata, not the proxy class (which returns null
from `getClass().getAnnotation()`):

```java
// LedgerEnricherPipeline.enrich() — updated
enrichers.handles()
    .sorted(Comparator.comparingInt(h ->
        (h.getBean() instanceof InjectableBean<?> ib) ? ib.getPriority() : Integer.MAX_VALUE))
    .map(Instance.Handle::get)
    .forEach(e -> {
        try { e.enrich(entry); }
        catch (Exception ex) { log.warnf("Enricher %s failed: %s",
            e.getClass().getSimpleName(), ex.getMessage()); }
    });
```

All CDI beans in Arc implement `InjectableBean`, so the `Integer.MAX_VALUE` fallback is dead
code in practice. It is retained as a guard rather than removed, with an explicit value that
correctly represents Arc's default priority (sorts last). **`@Priority` is mandatory on all
`LedgerEntryEnricher` implementations** — add this to the `LedgerEntryEnricher` Javadoc.
An enricher without `@Priority` sorts last (after all numbered enrichers) rather than at
some arbitrary position.

Assigned `@Priority` values (existing enrichers gain annotation with no behaviour change):

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

Runs only when `actorDid` is non-null. **Never throws** — the enricher contract
(`LedgerEntryEnricher` Javadoc: "Must not throw") is preserved.

```
cache lookup (actorId → CacheEntry<IdentityBindingStatus>)
  → hit, not expired   → set entry.pendingIdentityStatus (transient) and skip resolve
  → absent or expired  →
      DIDResolver.resolve(actorDid)
        → empty        → status = DID_UNRESOLVABLE
        → DIDDocument  →
            alsoKnownAs contains actorId?
              → no   → status = IDENTITY_MISMATCH
              → yes  →
                  agentPublicKey == null?
                    → yes → status = UNSIGNED
                             (DID + alsoKnownAs verified; no key to cross-check)
                    → no  →
                        agentPublicKey matches a verificationMethod.publicKeyBytes?
                          → no  → status = KEY_MISMATCH
                          → yes →
                              AgentCredentialValidator.validate(actorId, actorDid)
                                → empty/VALID → status = VALID
                                → EXPIRED     → status = CREDENTIAL_EXPIRED
                                → other       → status = CREDENTIAL_INVALID
      cache status
      set entry.pendingIdentityStatus = status (transient field — not persisted)
      fire AgentIdentityValidatedEvent (CDI async) if VALID
      fire AgentIdentityViolationEvent (CDI async) if non-VALID
```

### ENFORCE mode — `LedgerIdentityEnforcementListener`

**The enricher contract forbids throwing.** ENFORCE mode is implemented via a separate
JPA entity listener on `LedgerEntry`, not inside the enricher.

`LedgerEntry` gains one `@Transient` field (not persisted, zero schema change):

```java
@Transient
public IdentityBindingStatus pendingIdentityStatus;
```

`LedgerIdentityEnforcementListener` (registered in `@EntityListeners` on `LedgerEntry`
alongside `LedgerTraceListener`) reads this field in its own `@PrePersist`:

```java
@PrePersist
void enforceIdentity(Object entity) {
    if (!(entity instanceof LedgerEntry e)) return;
    if (e.pendingIdentityStatus == null) return;      // no binding configured
    if (config.validationMode() != ENFORCE) return;
    if (e.pendingIdentityStatus != VALID) {
        throw new LedgerIdentityViolationException(e.actorId, e.pendingIdentityStatus);
    }
}
```

This preserves the enricher's non-fatal contract and gives ENFORCE a well-defined,
testable, JPA-lifecycle-level enforcement point.

**ENFORCE is JPA-only.** `@EntityListeners` is a JPA lifecycle mechanism and does not fire
in `InMemoryLedgerEntryRepository`. The in-memory repository must not implement ENFORCE
mode — it is a test helper and should not block writes. Tests that need to verify ENFORCE
behaviour must use a `@QuarkusTest` with JPA active (not the in-memory path).

**No sequence gap on ENFORCE rollback.** In the JPA path, callers assign sequence numbers
via `SELECT MAX(sequenceNumber) + 1 FROM ledger_entry WHERE subjectId = ?` inside the same
`@Transactional` boundary as `save()`. When ENFORCE throws and the transaction rolls back,
that computation is also rolled back — no committed sequence number is incremented, so
`LedgerHealthJob`'s gap query (`COUNT != MAX - MIN + 1`) never observes a gap. ADR 0015
must document this reasoning so future maintainers do not introduce a non-transactional
sequence mechanism (which would break this invariant).

### Binding entry persistence — CDI async event observer

`ActorIdentityValidationEnricher` fires CDI async events but does **not** call
`entityManager.persist()` directly — doing so inside a `@PrePersist` callback is not
safe under the JPA spec (flush ordering is undefined). Persistence is the responsibility
of a separate observer running in a new transaction:

```java
@ApplicationScoped
public class ActorIdentityBindingObserver {

    @Inject EntityManager em;

    void onValidated(@ObservesAsync AgentIdentityValidatedEvent event) {
        persistBinding(event.actorId(), event.actorDid(), event.status(), ...);
    }

    void onViolation(@ObservesAsync AgentIdentityViolationEvent event) {
        persistBinding(event.actorId(), event.actorDid(), event.status(), ...);
    }

    @Transactional(REQUIRES_NEW)
    void persistBinding(String actorId, String actorDid, IdentityBindingStatus status, ...) {
        ActorIdentityBindingEntry entry = new ActorIdentityBindingEntry();
        entry.actorId = actorId;
        entry.subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(UTF_8));
        entry.entryType = LedgerEntryType.EVENT;
        entry.boundDid = actorDid;
        entry.validationResult = status;
        // ... populate remaining fields
        em.persist(entry);
    }
}
```

`@Transactional(REQUIRES_NEW)` ensures the binding entry commits independently of the
parent entry's transaction. Failure to write the binding entry is logged and swallowed
— the parent entry is not affected.

### Cache invalidation on key rotation

The write-path enricher cache is invalidated when a key rotation is recorded. Because
the `KeyRotationEntry` CDI event is deferred to #103, `KeyRotationService.recordRotation()`
will call `identityValidationEnricher.invalidateAll()` directly as part of #81. Issue
#103 will later replace this direct call with a CDI event-driven mechanism. **#103 is
an upstream dependency for proper cache invalidation; without it, write-path caches
retain stale results after a key rotation until the application restarts.** This
must be noted in ADR 0015.

---

## Read Path — Verification Service

`AgentIdentityVerificationService` (`@ApplicationScoped`) — symmetric read-side
counterpart to `AgentSignatureVerificationService`.

```java
IdentityVerificationResult verifyIdentityBinding(LedgerEntry entry);
```

```
entry.actorDid == null         → UNVERIFIABLE
entry.agentPublicKey == null   → UNSIGNED
otherwise:
  DIDResolver.resolve(entry.actorDid)   [uses read-path cache — TTL only, 5 min default]
    → empty           → DID_UNRESOLVABLE
    → DIDDocument     →
        alsoKnownAs contains entry.actorId?
          → no  → IDENTITY_MISMATCH
          → yes →
              any verificationMethod.publicKeyBytes == entry.agentPublicKey?
                → yes → VALID
                → no  → KEY_MISMATCH
```

**The read path performs cryptographic identity binding verification (DID ↔ public key)
only.** `AgentCredentialValidator` is NOT re-invoked on the read path. VC-level
validation results are recorded at write time in `ActorIdentityBindingEntry` and are not
re-evaluated on read. Consumers needing the full write-time VC validation result should
query `ActorIdentityBindingRepository.latestBindingFor(actorId)`.

```java
public enum IdentityVerificationResult {
    VALID,              // public key matches DID document; alsoKnownAs confirmed
    UNVERIFIABLE,       // no actorDid on entry — actor not bound to a DID
    UNSIGNED,           // no agentPublicKey — nothing to cross-check
    DID_UNRESOLVABLE,   // resolver returned empty
    IDENTITY_MISMATCH,  // DID document alsoKnownAs does not include actorId
    KEY_MISMATCH        // key no longer in DID document (rotated since entry written)
}
```

`KEY_MISMATCH` mirrors `VerificationResult.SUSPECT` — the entry content is intact; the
key was valid when written.

**Read-path cache:** own `AbstractCachingIdentityProvider<DIDDocument>` instance,
independent of the write-path cache. TTL-only invalidation (no CDI event hook — the
read path resolves historical `actorDid` values which may predate current actorId-based
invalidation). Default TTL: 5 minutes. `UNVERIFIABLE` and `UNSIGNED` results are not
cached (they depend on entry fields, not DID resolution).

Reactive counterpart (`Uni<IdentityVerificationResult>`) deferred to #109, excluded via
`LedgerProcessor` ExcludedTypeBuildItem when reactive.enabled=false.

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
    public IdentityBindingStatus validationResult;

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
`KeyRotationEntry`. `entryType = LedgerEntryType.EVENT` — set by
`ActorIdentityBindingObserver` before persist; must be explicit because it determines
canonical bytes for the Merkle leaf hash.

### Result enum — write path

```java
public enum IdentityBindingStatus {
    VALID, UNSIGNED, DID_UNRESOLVABLE, IDENTITY_MISMATCH,
    KEY_MISMATCH, CREDENTIAL_EXPIRED, CREDENTIAL_INVALID
}
```

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
    CONSTRAINT chk_identity_binding_result CHECK (
        validation_result IN ('VALID','UNSIGNED','DID_UNRESOLVABLE','IDENTITY_MISMATCH',
                              'KEY_MISMATCH','CREDENTIAL_EXPIRED','CREDENTIAL_INVALID')
    ),
    CONSTRAINT chk_identity_credential_result CHECK (
        credential_result IS NULL OR
        credential_result IN ('VALID','EXPIRED','INVALID_SIGNATURE','ISSUER_UNKNOWN','NOT_FOUND')
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

## CDI Events

Two events, analogous to `AgentSignatureSuspectEvent`:

```java
// Fired async on VALID result
public record AgentIdentityValidatedEvent(
    String actorId, String actorDid, IdentityBindingStatus status,
    boolean alsoKnownAsVerified, boolean keyMatchVerified, String verifiedKeyRef) {}

// Fired async on non-VALID result
public record AgentIdentityViolationEvent(
    String actorId, String actorDid, IdentityBindingStatus status) {}
```

Consumers use `@ObservesAsync` to react. `ActorIdentityBindingObserver` observes both
to persist the binding entry.

---

## WebDIDResolver Security

`WebDIDResolver` handles untrusted DID URIs. Required security decisions (not implementation details):

| Concern | Rule |
|---|---|
| SSRF | Resolved domain must not be RFC 1918 (`10.x`, `172.16-31.x`, `192.168.x`), loopback (`127.x`, `::1`), or link-local. Configurable allowlist via `casehub.ledger.agent-identity.did-resolver.web.allowed-domains` (empty = allow all non-RFC1918) |
| Redirects | Do not follow HTTP→HTTP redirects to internal addresses. Only follow HTTPS→HTTPS redirects; maximum 2 hops |
| Response size | Reject responses exceeding `casehub.ledger.agent-identity.did-resolver.web.max-response-bytes` (default: 1 MiB) |
| TLS | Non-optional. Use system trust store. No hostname verification bypass |

---

## Module Placement

### `api` module

```
api/src/main/java/io/casehub/ledger/api/
├── model/
│   ├── IdentityBindingStatus.java      (VALID | UNSIGNED | DID_UNRESOLVABLE | IDENTITY_MISMATCH
│   │                                    KEY_MISMATCH | CREDENTIAL_EXPIRED | CREDENTIAL_INVALID)
│   └── CredentialValidationResult.java (VALID | EXPIRED | INVALID_SIGNATURE | ISSUER_UNKNOWN | NOT_FOUND)
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
        ├── AbstractCachingIdentityProvider.java   (with CacheEntry<C> TTL wrapper)
        ├── ActorDIDEnricher.java                  (@Priority(40))
        ├── ActorIdentityValidationEnricher.java   (@Priority(50))
        ├── ActorIdentityBindingObserver.java      (async CDI observer; persists binding entry)
        ├── AgentIdentityValidatedEvent.java
        ├── AgentIdentityViolationEvent.java
        ├── AgentIdentityVerificationService.java
        ├── ConfiguredActorDIDProvider.java
        ├── KeyDIDResolver.java                    (standards-compliant; no alsoKnownAs)
        ├── LedgerIdentityEnforcementListener.java (@EntityListeners on LedgerEntry)
        ├── LedgerIdentityViolationException.java
        ├── NoOpActorDIDProvider.java
        ├── NoOpCredentialValidator.java
        ├── NoOpDIDResolver.java
        └── WebDIDResolver.java
```

**Also required:** add `LedgerIdentityEnforcementListener` to `@EntityListeners` on
`LedgerEntry` (alongside existing `LedgerTraceListener`). Add `@Transient
IdentityBindingStatus pendingIdentityStatus` field to `LedgerEntry`.

### `persistence-memory` module

```
InMemoryActorIdentityBindingRepository.java   (@Alternative @Priority(1))
```

### Test helpers (runtime test scope)

```
TestDIDResolver.java   — Map<String, DIDDocument>-backed stub; lets tests set
                         arbitrary documents including alsoKnownAs; primary test helper
                         for all ITs (not KeyDIDResolver — did:key has no alsoKnownAs)
```

### Config (`LedgerConfig`)

| Key | Type | Default | Description |
|---|---|---|---|
| `agent-identity.validation-mode` | enum | `WARN` | `WARN` or `ENFORCE` |
| `agent-identity.dids."<actorId>"` | String | — | DID URI per actorId |
| `agent-identity.did-resolver.web.timeout-ms` | int | 5000 | HTTP timeout for WebDIDResolver |
| `agent-identity.did-resolver.web.max-response-bytes` | int | 1048576 | Max DID doc size |
| `agent-identity.did-resolver.web.allowed-domains` | List | (none) | Optional SSRF allowlist |
| `agent-identity.did-resolver.cache-ttl-minutes` | int | 5 | DID document cache TTL |
| `agent-identity.credential-cache-ttl-minutes` | int | 60 | VC validation cache TTL |

---

## Testing Strategy

### Unit tests (pure Java)
- `ActorDIDEnricherTest` — no DID: actorDid stays null; DID configured: populated; provider throws: non-fatal
- `ActorIdentityValidationEnricherTest` — all `IdentityBindingStatus` paths; cache hit; cache TTL expiry; both CDI events fired with correct type; UNSIGNED path (actorDid set, agentPublicKey null); pendingIdentityStatus set correctly
- `LedgerIdentityEnforcementListenerTest` — WARN: no throw; ENFORCE + non-VALID: throws; ENFORCE + VALID: no throw; ENFORCE + null status: no throw (no binding configured)
- `AgentIdentityVerificationServiceTest` — all `IdentityVerificationResult` paths with `TestDIDResolver`
- `AbstractCachingIdentityProviderTest` — TTL expiry (advance clock); `invalidate(key)`, `invalidateAll()`; concurrent `putIfAbsent` semantics; EXPIRED not cached
- `WebDIDResolverTest` — WireMock: success, 404, malformed JSON, timeout; response size limit enforced; redirect policy; SSRF domain rejection; TLS required
- `KeyDIDResolverTest` — valid `did:key:z6Mk...` roundtrip; bad multibase prefix; no `alsoKnownAs` in resolved document (standards-compliant)
- `TestDIDResolverTest` — set arbitrary document with `alsoKnownAs`; override for same DID; miss returns empty

### Integration tests (`@QuarkusTest`, H2)
- `ActorIdentityBindingEntryIT` — full write path with `TestDIDResolver`: actorDid populated; async observer writes `ActorIdentityBindingEntry` with `entryType=EVENT`; second write uses cache; TTL expiry triggers re-resolution; `KeyRotationService.recordRotation()` invalidates cache and triggers new binding entry
- `ActorIdentityBindingRepositoryIT` — `latestBindingFor`, `bindingHistoryFor` ordering; multiple binding entries per actor
- `AgentIdentityVerificationServiceIT` — `TestDIDResolver` + `ConfiguredAgentSigner`: sign entry → bind DID → verify VALID; rotate key → KEY_MISMATCH; null actorDid → UNVERIFIABLE
- `LedgerIdentityEnforcementListenerIT` — ENFORCE mode + `TestDIDResolver`: VALID passes; IDENTITY_MISMATCH blocked with `LedgerIdentityViolationException`
- `InMemoryActorIdentityBindingRepositoryIT` — zero-datasource path

### Test helper pattern (all ITs)

```java
// Standard IT setup for identity binding
TestDIDResolver resolver = new TestDIDResolver();
DIDDocument doc = new DIDDocument(
    "did:example:test",
    List.of(new VerificationMethod("did:example:test#key-1", "Ed25519VerificationKey2020",
        keyPair.getPublic().getEncoded())),
    List.of("claude:test-agent@v1")   // alsoKnownAs — arbitrary, test-controlled
);
resolver.register("did:example:test", doc);
```

---

## Deferred Issues

| Issue | Description |
|---|---|
| casehubio/ledger#103 | Rotation-triggered cache invalidation via CDI event (currently a direct call from `KeyRotationService`; #103 upgrades this) |
| casehubio/ledger#107 | `ScimActorDIDProvider` — enterprise SCIM2 integration |
| casehubio/ledger#108 | `JwtVCValidator` — W3C VC JWT; must implement `validUntil`-bounded cache TTL |
| casehubio/ledger#109 | `ReactiveAgentIdentityVerificationService` — reactive parity |
| casehubio/parent#111 | Cross-repo `actorId` → DID migration (Approach B, long-term) |

---

## Known Gap — VC Expiry and Cache TTL

When `JwtVCValidator` (#108) is implemented, the VALID result cache TTL must be bounded
by the VC's `validUntil` field: `TTL = min(configuredTtl, vc.validUntil - Instant.now())`.
`EXPIRED` results must not be cached at all. This gap is explicitly flagged here so it
is not overlooked during #108 implementation.

---

## Protocol

PP-20260530-bf919d — `scim2-agent-identity-lookup` (casehubio/parent)
Governs: SCIM2 Agent endpoint lookup pattern for all casehub components.

---

## ADR Required — 0015

ADR 0015 must document:
- Two-field model rationale (actorId trust key unchanged; actorDid cryptographic binding)
- `alsoKnownAs` verification requirement (closes divergence attack)
- `ActorIdentityBindingEntry` as subclass, not supplement (canonical bytes, Merkle participation, lifecycle independence)
- Enricher ordering via `InjectableBean.getPriority()` (not reflection — Arc proxies break `getClass().getAnnotation()`)
- ENFORCE mode via `LedgerIdentityEnforcementListener` (not inside enricher — enricher contract forbids throwing)
- Binding entry persistence via async CDI observer in `REQUIRES_NEW` transaction (not inside `@PrePersist`)
- SPI module placement rule: simple-value-type SPIs in api; runtime-type-coupled SPIs in runtime
- `IdentityBindingStatus` (write path, stored) vs `IdentityVerificationResult` (read path, computed)
- Read path verifies DID↔key only — VC re-validation is not done on read
- #103 as dependency for event-driven cache invalidation
- Known gap: VC `validUntil`-bounded cache TTL (deferred to #108)
