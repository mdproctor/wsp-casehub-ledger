# Design: ScimActorDIDProvider, ReactiveAgentIdentityVerificationService, AgentKeyRotated CDI event

**Branch:** issue-107-scim-actor-did-provider  
**Issues:** casehubio/ledger#107, casehubio/ledger#109, casehubio/ledger#103 (rolled in), casehubio/parent#107  
**Date:** 2026-05-31

---

## Summary

Four changes on one branch:

1. **#103** — `AgentKeyRotated` CDI event + `KeyRotationService` refactor: replaces direct cache-invalidation calls with a fired CDI event; `AbstractCachingAgentSigner` and `ActorIdentityValidationEnricher` observe it.
2. **#107** — `ScimActorDIDProvider`: enterprise SCIM2 implementation of `ActorDIDProvider`; caches full `ScimAgentResource`; observes `AgentKeyRotated` for invalidation.
3. **#109** — `ReactiveAgentIdentityVerificationService`: `@DefaultBean` bridge wrapping the blocking `AgentIdentityVerificationService`.
4. **parent#107** — `docs/integration/scim2-agent-identity.md` + `PLATFORM.md` update in `casehubio/parent`.

---

## Section 1 — AgentKeyRotated CDI event (resolves #103)

### New type

```
runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyRotated.java
```

```java
public record AgentKeyRotated(String actorId, String previousKeyRef, String newKeyRef) {}
```

Lives in `runtime.service` — a CDI event, not an API contract. Consumers within the same CDI context observe it; cross-process consumers are not a use case.

### KeyRotationService changes

- Inject `Event<AgentKeyRotated> keyRotatedEvent`.
- Remove `@Inject ActorIdentityValidationEnricher identityEnricher` — `KeyRotationService` no longer knows about identity concerns.
- After `ledgerRepo.save(entry)`, fire: `keyRotatedEvent.fire(new AgentKeyRotated(actorId, previousKeyRef, newKeyRef))`.
- Observer is synchronous (`@Observes`) — observers only call `cache.remove()`, safe within the transaction boundary. The cache entry is evicted before the caller returns; the next cache miss reloads from the committed state.

### ReactiveKeyRotationService changes

- Inject `Event<AgentKeyRotated> keyRotatedEvent`.
- After the Uni completes, fire via `event.fireAsync(...)` — avoids blocking the Vert.x event loop with synchronous observer dispatch.

### AbstractCachingAgentSigner changes

Add an inherited CDI observer method:

```java
void onKeyRotated(@Observes AgentKeyRotated event) {
    invalidate(event.actorId());
}
```

CDI inherits observer methods into concrete `@ApplicationScoped` subclasses (e.g. `ConfiguredAgentKeyProvider`). Every signing key provider gets automatic rotation-triggered cache invalidation without subclass changes.

### ActorIdentityValidationEnricher changes

Add observer:

```java
void onKeyRotated(@Observes AgentKeyRotated event) {
    statusCache.invalidate(event.actorId());
}
```

Replaces the previous direct call from `KeyRotationService`.

---

## Section 2 — ScimActorDIDProvider (resolves #107)

### New types

**`ScimAgentResource`** — value type for the SCIM cache:

```java
public record ScimAgentResource(String did, byte[] publicKeyBytes) {}
```

`publicKeyBytes` is nullable — not all SCIM deployments populate `x509Certificates`.

**`ScimActorDIDProvider`**:

```java
@ApplicationScoped
@Alternative
public class ScimActorDIDProvider
        extends AbstractCachingIdentityProvider<ScimAgentResource>
        implements ActorDIDProvider {
    ...
}
```

CDI pattern: **Pattern C** (protocol `alternative-extension-patterns.md`). `NoOpActorDIDProvider @DefaultBean` is the inactive default; `ScimActorDIDProvider @Alternative` activates only when explicitly selected via `quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.service.identity.ScimActorDIDProvider`. Consumers who want SCIM declare the selection; all others get the no-op or config-map default without ambiguity.

### Config

New nested interface `ScimConfig` under `LedgerConfig.AgentIdentityConfig`:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `casehub.ledger.agent-identity.scim.endpoint` | `String` | — | Base URL of the SCIM server (e.g. `https://idp.example.com`). Must use HTTPS. |
| `casehub.ledger.agent-identity.scim.auth-token` | `String` | — | Bearer token for `Authorization` header. Static deploy-time credential — not a `Preferences` key. |
| `casehub.ledger.agent-identity.scim.timeout-ms` | `int` | `5000` | HTTP connect + read timeout in milliseconds. |

HTTPS enforcement: constructor validates `endpoint.startsWith("https://")` and throws `IllegalArgumentException` if not. No RFC 1918 blocking — the SCIM endpoint is admin-configured enterprise infrastructure, not attacker-controlled input.

### HTTP client

`java.net.HttpClient` — consistent with `WebDIDResolver`, zero new Quarkus extension dependencies. Avoids the `@Provider @ApplicationScoped` CDI injection bypass issue documented in GE-20260530-385dbb.

### loadContext algorithm

1. Build filter URL: `{endpoint}/scim/v2/Agents?filter=externalId%20eq%20%22{encoded-actorId}%22`
   - `actorId` is percent-encoded in the filter VALUE, never in a path segment (PP-20260530-bf919d — colons in `claude:reviewer@v1` would silently split a path segment).
2. `GET` with `Authorization: Bearer {authToken}` + `Accept: application/json`.
3. **401** → throw `ScimAuthenticationException` (not cached; next call retries; log WARN).
4. **404 or empty `totalResults`** → `Optional.empty()` (cached for full TTL — actor not registered).
5. **200, `totalResults > 0`**: parse `Resources[0]`:
   - DID: `urn:ietf:params:scim:schemas:extension:casehub:2.0:Agent` extension object → `did` field.
   - Public key: `x509Certificates[0].value` (Base64) → decode to `byte[]`; absent → `null`.
6. Return `Optional.of(new ScimAgentResource(did, publicKeyBytes))`.
7. Any other HTTP status or parse failure → throw (not cached; log WARN with status code).

### ActorDIDProvider implementation

```java
@Override
public Optional<String> didFor(String actorId) {
    return get(actorId).map(ScimAgentResource::did);
}
```

The caching base handles TTL and empty-result caching. `get()` calls `loadContext()` on miss.

### Cache invalidation

```java
void onKeyRotated(@Observes AgentKeyRotated event) {
    invalidate(event.actorId());
}
```

Clears the cache entry for the rotated actor. Next `didFor()` call triggers a fresh SCIM lookup.

### Testing

WireMock 3.4.2 (already in test deps). Dynamic port via `QuarkusTestResourceLifecycleManager` (GE-20260526-286ac7). The provider is instantiated directly in tests (not via CDI `@QuarkusTest`) since the `@Alternative` activation gate would require full Quarkus boot.

Test cases:
- Successful lookup — `didFor()` returns the DID from the SCIM extension; second call hits cache (WireMock verify call count = 1).
- 404 response — `didFor()` returns empty; cached (second call still empty, call count = 1).
- 401 response — throws; not cached (second call re-invokes WireMock, call count = 2).
- Malformed extension (missing `did` field) — throws (not cached).
- Key rotation: seed cache via `didFor()`, fire `AgentKeyRotated` via CDI event, assert next `didFor()` invokes WireMock again (call count = 2).
- HTTPS enforcement: constructor with `http://` endpoint throws.

---

## Section 3 — ReactiveAgentIdentityVerificationService (resolves #109)

```java
@DefaultBean
@ApplicationScoped
public class ReactiveAgentIdentityVerificationService {

    @Inject AgentIdentityVerificationService blockingService;

    public Uni<IdentityVerificationResult> verifyIdentityBindingAsync(LedgerEntry entry) {
        return Uni.createFrom()
            .item(() -> blockingService.verifyIdentityBinding(entry))
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

**CDI annotation: `@DefaultBean @ApplicationScoped`** — no `@IfBuildProperty`, not excluded by `LedgerProcessor`. This follows protocol `reactive-spi-bridge-default-bean.md`: the bridge has no Hibernate Reactive dependency; it wraps a blocking service that uses only `DIDResolver` (in-memory cache + HTTP, no DB). Always safe to activate.

This diverges from the issue description (which prescribes `LedgerProcessor` ExcludedTypeBuildItem). The issue is wrong: the ExcludedTypeBuildItem pattern is for beans with Hibernate Reactive dependencies. A bridge with no reactive DB dep must be `@DefaultBean` and always active.

**`InMemoryReactiveActorIdentityBindingRepository`** (mentioned in the issue) is **out of scope**. `ReactiveAgentIdentityVerificationService` does not use `ActorIdentityBindingRepository` — the blocking delegate reads `entry.actorDid` and `entry.agentPublicKey` from the supplied `LedgerEntry` and delegates to `DIDResolver`. No repository access. File a separate issue if reactive binding history queries are needed.

### Testing

`@QuarkusTest` with `@InjectMock DIDResolver`. One test per `IdentityVerificationResult` variant (UNVERIFIABLE, UNSIGNED, DID_UNRESOLVABLE, IDENTITY_MISMATCH, KEY_MISMATCH, VALID). One test verifying the call runs on a worker-pool thread (not the calling thread) — validates `.runSubscriptionOn()` is present.

---

## Section 4 — parent#107: SCIM2 integration doc + PLATFORM.md

### New file

`docs/integration/scim2-agent-identity.md` in `casehubio/parent`.

Stable raw URL: `https://raw.githubusercontent.com/casehubio/parent/main/docs/integration/scim2-agent-identity.md`

Contents:
1. CaseHub SCIM2 schema extension definition — schema URI, full field table (type, required, description).
2. Field mapping table — `externalId` → `actorId`, `{did}` → DID URI, `x509Certificates[0].value` → `agentPublicKey` bytes, `clientId`/`issuerUri` → OAuth signing credential reference, `name` → persona display name.
3. Canonical lookup pattern — `GET /scim/v2/Agents?filter=externalId eq "{actorId}"` with URL encoding rules; actorId in filter VALUES only, never in path segments (colons in `claude:reviewer@v1` silently split a path).
4. Caching expectations — TTL 5 min default; invalidate on `AgentKeyRotated` CDI event.
5. Security constraints — what MUST NOT be stored in SCIM (private keys); HTTPS required; auth token is a static deploy-time credential.
6. Example SCIM Agent resource JSON — `claude:tarkus-reviewer@v1` concrete example.
7. Link to protocol PP-20260530-bf919d (`casehub-agent-identity-lookup`).

### PLATFORM.md changes

- **Implementation Protocols table**: add `[SCIM2 agent identity lookup](integration/scim2-agent-identity.md)` row — "Agent identity attributes (DID, public key, capabilities) resolved via SCIM2 `Agent` endpoint using `actorId` as `externalId`. Schema extension: `urn:ietf:params:scim:schemas:extension:casehub:2.0:Agent`."
- **Capability Ownership table**: update the Agent Identity line — note SCIM2 as the resolution mechanism for `actorId → DID` and the `ScimActorDIDProvider` as the ledger-side implementation.
- **casehub-ledger repository entry**: note `ScimActorDIDProvider @Alternative` as the SCIM-based `ActorDIDProvider`.

**No per-repo CLAUDE.md updates** in this session — consuming repos (casehub-eidos, casehub-engine) should update their own `CLAUDE.md` fetch blocks when they implement SCIM integration. File follow-on issues at that point.

---

## Coherence review against PLATFORM.md and protocols

| Check | Result |
|-------|--------|
| `ScimActorDIDProvider` stays domain-agnostic (foundation rule) | ✅ Resolves DID from SCIM — no domain knowledge |
| `ActorDIDProvider` SPI stays in `api/spi/identity/` | ✅ Unchanged |
| `@Alternative` CDI pattern (Pattern C) | ✅ `@DefaultBean` no-op, `@Alternative` optional impls |
| SCIM actorId in filter value, not path segment (PP-20260530-bf919d) | ✅ URL-encoded filter value |
| Static credentials via `@ConfigProperty`, not `Preferences` (`static-credentials-config-property-not-preferences.md`) | ✅ `LedgerConfig.ScimConfig.authToken()` |
| Reactive bridge = `@DefaultBean`, no `@IfBuildProperty` (`reactive-spi-bridge-default-bean.md`) | ✅ |
| All `LedgerEntryEnricher` implementations carry `@Priority` (`ledger-enricher-priority-mandate.md`) | ✅ No new enrichers in this scope |
| No Flyway migrations — pure service layer | ✅ |
| Closes #103 side-effect: `KeyRotationService` no longer imports identity package | ✅ Cleaner coupling |

---

## Out of scope / follow-on issues

| Item | Why deferred |
|------|-------------|
| `InMemoryReactiveActorIdentityBindingRepository` | `ReactiveAgentIdentityVerificationService` doesn't use `ActorIdentityBindingRepository` at all |
| Per-repo CLAUDE.md updates (eidos, engine) | File when those repos implement SCIM integration |
| `JwtVCValidator` (#108) | Separate issue, deferred from #81 |
| `ReactiveAgentIdentityVerificationService` querying binding history | Separate issue if reactive binding history queries are needed |
