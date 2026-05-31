# Design: ScimActorDIDProvider, ReactiveAgentIdentityVerificationService, AgentKeyRotatedEvent CDI event

**Branch:** issue-107-scim-actor-did-provider  
**Issues:** casehubio/ledger#107, casehubio/ledger#109, casehubio/ledger#103 (rolled in), casehubio/parent#107  
**Date:** 2026-05-31  
**Rev:** 2 — post spec review

---

## Summary

Four changes on one branch:

1. **#103** — `AgentKeyRotatedEvent` CDI event + `KeyRotationService` refactor: replaces direct cache-invalidation calls with a fired CDI event; `AbstractCachingAgentSigner` and `ActorIdentityValidationEnricher` observe it.
2. **#107** — `ScimActorDIDProvider`: enterprise SCIM2 implementation of `ActorDIDProvider`; observes `AgentKeyRotatedEvent` for cache invalidation.
3. **#109** — `ReactiveAgentIdentityVerificationService`: `@DefaultBean @Unremovable` bridge wrapping the blocking `AgentIdentityVerificationService`.
4. **parent#107** — `docs/integration/scim2-agent-identity.md` + `PLATFORM.md` update in `casehubio/parent`.

---

## Section 1 — AgentKeyRotatedEvent CDI event (resolves #103)

### New type

```
runtime/src/main/java/io/casehub/ledger/runtime/service/AgentKeyRotatedEvent.java
```

```java
/**
 * @param previousKeyRef keyRef of the retired key; {@code null} if unknown
 * @param newKeyRef      keyRef of the replacement key; {@code null} for pure revocation
 */
public record AgentKeyRotatedEvent(String actorId, String previousKeyRef, String newKeyRef) {}
```

`previousKeyRef` and `newKeyRef` are nullable — documented explicitly because future observers using these fields (not just `actorId`) must null-check. Lives in `runtime.service` — a CDI event, not an API contract.

### KeyRotationService changes

- Inject `Event<AgentKeyRotatedEvent> keyRotatedEvent`.
- Remove `@Inject ActorIdentityValidationEnricher identityEnricher` — `KeyRotationService` no longer imports the identity package.
- After `ledgerRepo.save(entry)`, fire: `keyRotatedEvent.fire(new AgentKeyRotatedEvent(actorId, previousKeyRef, newKeyRef))`.
- Observer is synchronous (`@Observes`) — observers only call `cache.remove()`, safe within the transaction boundary. The cache entry is evicted before the caller returns; the next cache miss reloads from the committed state.

### ReactiveKeyRotationService changes

- Inject `Event<AgentKeyRotatedEvent> keyRotatedEvent`.
- After the Uni completes, fire via `event.fireAsync(new AgentKeyRotatedEvent(...))`.
- **Fire-and-forget semantics are intentional.** `fireAsync()` returns a `CompletionStage` that is not chained. Observer failure is invisible to the reactive pipeline. This is correct: cache eviction is best-effort and benign — a non-committed rotation that evicts the cache causes a cache miss that reloads from the correct committed state. Chaining on the `CompletionStage` would complicate the Uni pipeline for no correctness gain.

### AbstractCachingAgentSigner changes

Add an inherited CDI observer method:

```java
void onKeyRotated(@Observes AgentKeyRotatedEvent event) {
    invalidate(event.actorId());
}
```

**CDI observer inheritance dependency:** CDI 4.0 (§3.5) specifies that observer methods defined in superclasses are inherited by managed bean subclasses. `AbstractCachingAgentSigner` is a non-CDI abstract class; its concrete subclass `ConfiguredAgentKeyProvider @ApplicationScoped` is a managed bean and inherits this observer. This behaviour is confirmed by Quarkus ARC. An explicit `@QuarkusTest` integration test must verify that rotating a key fires the event and invalidates the signer cache — do not assume CDI inheritance works without a test.

### ActorIdentityValidationEnricher changes

Add observer:

```java
void onKeyRotated(@Observes AgentKeyRotatedEvent event) {
    statusCache.invalidate(event.actorId());
}
```

Replaces the previous direct call from `KeyRotationService`.

---

## Section 2 — ScimActorDIDProvider (resolves #107)

### New types

**`ScimAgentResource`** — value type for the SCIM cache. Contains only the DID string for the current scope. `publicKeyBytes` is intentionally excluded: SCIM `x509Certificates[0].value` (RFC 7643 §4.1.2) is a DER-encoded X.509 certificate — not `SubjectPublicKeyInfo` bytes as required by `LedgerEntry.agentPublicKey`. Extracting the key bytes requires `CertificateFactory.getInstance("X.509").generateCertificate(...).getPublicKey().getEncoded()`. No current consumer needs this field; add it when there is one.

```java
public record ScimAgentResource(String did) {}
```

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

CDI pattern: **Pattern C** (protocol `alternative-extension-patterns.md`). `NoOpActorDIDProvider @DefaultBean` is the inactive default; `ScimActorDIDProvider @Alternative` activates only when explicitly selected via `quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.service.identity.ScimActorDIDProvider`. When activated, `ConfiguredActorDIDProvider @ApplicationScoped` is superseded — any `casehub.ledger.agent-identity.dids.*` properties are silently ignored. The integration doc (Section 4) documents this explicitly.

### Config

New nested interface `ScimConfig` under `LedgerConfig.AgentIdentityConfig`:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `casehub.ledger.agent-identity.scim.endpoint` | `String` | — | Base URL of the SCIM server (`https://idp.example.com`). Must use HTTPS — validated in `@PostConstruct`. |
| `casehub.ledger.agent-identity.scim.auth-token` | `String` | — | Bearer token for `Authorization` header. Static deploy-time credential per `static-credentials-config-property-not-preferences.md`. |
| `casehub.ledger.agent-identity.scim.timeout-ms` | `int` | `5000` | HTTP connect + read timeout in milliseconds. |
| `casehub.ledger.agent-identity.scim.cache-ttl-minutes` | `int` | `5` | TTL for cached SCIM lookups. Separate from `didResolverCacheTtlMinutes` — SCIM is not a DID resolver. |

**HTTPS validation:** Performed in `@PostConstruct`, not the constructor. For `@Alternative @ApplicationScoped`, the contextual instance is created lazily on first injection use — not at Quarkus startup. `@PostConstruct` fires at the same time but is the correct hook. Misconfigured HTTPS surfaces on the first `didFor()` call, not at boot. This is a known limitation of the lazy-instantiation model for library beans; document it in the Javadoc.

No RFC 1918 blocking — the SCIM endpoint is admin-configured enterprise infrastructure, not attacker-controlled input (unlike `WebDIDResolver` where the DID URI is operator-provided but the document URL is attacker-influenced).

### HTTP client

`java.net.HttpClient` — consistent with `WebDIDResolver`, zero new Quarkus extension dependencies. Avoids the `@Provider @ApplicationScoped` CDI injection bypass issue (GE-20260530-385dbb).

### loadContext algorithm

1. URL-encode `actorId` for the filter value: `URLEncoder.encode(actorId, StandardCharsets.UTF_8).replace("+", "%20")`. Colons in `claude:reviewer@v1` encode to `%3A`; `@` encodes to `%40`. actorId appears in the filter VALUE only, never in a path segment (PP-20260530-bf919d).

2. Build filter URL: `{endpoint}/scim/v2/Agents?filter=externalId%20eq%20%22{encoded-actorId}%22`

3. `GET` with `Authorization: Bearer {authToken}` + `Accept: application/json`.

4. **401** → throw `IllegalStateException("SCIM authentication failed (HTTP 401) for actorId: " + actorId)`, log WARN. Not cached — next call retries. No named exception type: the exception is always caught as `Exception` by `AbstractCachingIdentityProvider`; a named type has no call-site value.

5. **404** → log WARN (`"SCIM endpoint returned 404 — possible misconfiguration: " + endpoint`). Throw (not cached). 404 on a filter endpoint indicates a wrong base URL or unsupported resource type, not "actor not found" — caching it as absent would hide misconfiguration for a full TTL.

6. **200, `totalResults == 0`** → `Optional.empty()`. Cached for full TTL — actor is not registered in SCIM. Normal case.

7. **200, `totalResults > 1`** → log WARN (`"SCIM returned " + totalResults + " results for externalId " + actorId + " — externalId should be unique; using first result"`). Parse `Resources[0]`.

8. **200, `totalResults == 1`**: parse `Resources[0]`:
   - DID: extension object `urn:ietf:params:scim:schemas:extension:casehub:2.0:Agent` → `did` field.
   - If DID field absent or blank: throw (malformed SCIM resource — not cached).

9. Return `Optional.of(new ScimAgentResource(did))`.

10. Any other HTTP status or parse failure → throw, log WARN with status code. Not cached — next call retries.

### ActorDIDProvider implementation

```java
@Override
public Optional<String> didFor(String actorId) {
    return get(actorId).map(ScimAgentResource::did);
}
```

### Cache invalidation

```java
void onKeyRotated(@Observes AgentKeyRotatedEvent event) {
    invalidate(event.actorId());
}
```

### Testing

WireMock 3.4.2 (already in test deps — raw WireMock, not Quarkiverse WireMock, per GE-20260530-29545c). Dynamic port via `QuarkusTestResourceLifecycleManager` (GE-20260526-286ac7).

**Test split:**

Unit tests (direct instantiation — no CDI context needed):
- Successful lookup — `didFor()` returns the DID; second call hits cache (WireMock verify call count = 1).
- `totalResults == 0` — returns empty; cached (second call count = 1).
- 401 — throws; not cached (second call re-invokes WireMock, count = 2).
- 404 — throws; not cached (count = 2).
- `totalResults > 1` — logs WARN, returns first result.
- Malformed extension (missing `did` field) — throws; not cached.
- Cache invalidation: seed cache via `didFor()`, call `provider.onKeyRotated(new AgentKeyRotatedEvent(actorId, null, null))` directly, assert next `didFor()` invokes WireMock again (count = 2). This tests cache logic, not CDI wiring.
- HTTPS enforcement: `@PostConstruct` with `http://` endpoint throws.
- URL encoding: `actorId` with colon and `@` is encoded correctly in the WireMock request URL.

`@QuarkusTest` integration test (verifies CDI observer wiring):
- Activate `ScimActorDIDProvider` via `quarkus.arc.selected-alternatives`.
- Inject `KeyRotationService` and `ActorDIDProvider`.
- Seed cache, call `keyRotationService.recordRotation(actorId, ...)`, assert next `actorDIDProvider.didFor(actorId)` triggers a fresh WireMock call.
- This verifies the full CDI event path: `KeyRotationService` → `AgentKeyRotatedEvent` → `ScimActorDIDProvider.onKeyRotated()`.
- **Side-effect WireMock note:** When `ScimActorDIDProvider @Alternative` is active, `ActorDIDEnricher @Priority(40)` runs on every `ledgerRepo.save()`. `KeyRotationService.recordRotation()` calls `ledgerRepo.save(KeyRotationEntry)`, which triggers the enricher pipeline, which calls `ScimActorDIDProvider.didFor(actorId)`. The WireMock stub must handle this lookup (it fires before the cache is seeded and before the rotation completes). Account for this call in WireMock verify counts — the integration test will see at minimum: 1 call from the enricher during save, 1 call to seed the cache for the pre-rotation assert, and 1 call after invalidation to confirm the re-fetch.

---

## Section 3 — ReactiveAgentIdentityVerificationService (resolves #109)

```java
@DefaultBean
@ApplicationScoped
@Unremovable
public class ReactiveAgentIdentityVerificationService {

    @Inject AgentIdentityVerificationService blockingService;

    public Uni<IdentityVerificationResult> verifyIdentityBindingAsync(LedgerEntry entry) {
        return Uni.createFrom()
            .item(() -> blockingService.verifyIdentityBinding(entry))
            .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

**CDI annotation: `@DefaultBean @ApplicationScoped @Unremovable`.**
- `@DefaultBean`: follows protocol `reactive-spi-bridge-default-bean.md` — bridge has no Hibernate Reactive dep, always safe to activate, no `@IfBuildProperty` gate.
- `@Unremovable`: required because no code within the extension itself injects `ReactiveAgentIdentityVerificationService`. Without it, Quarkus ARC may dead-code-eliminate the bean in extension context (no build-time injection point). Consumer code injecting this type at application build time won't be visible at extension build time.
- This diverges from the issue description (which prescribes `LedgerProcessor` ExcludedTypeBuildItem). The ExcludedTypeBuildItem pattern is for beans with Hibernate Reactive dependencies. A pure bridge must be `@DefaultBean` and always active.

**`InMemoryReactiveActorIdentityBindingRepository`** is **out of scope.** `ReactiveAgentIdentityVerificationService` wraps `AgentIdentityVerificationService.verifyIdentityBinding(LedgerEntry entry)`, which reads `entry.actorDid` / `entry.agentPublicKey` from the supplied `LedgerEntry` and delegates to `DIDResolver`. No repository access. File a separate issue if reactive binding history queries are needed.

### Testing

`@QuarkusTest` with `@InjectMock DIDResolver`. One test per `IdentityVerificationResult` variant (UNVERIFIABLE, UNSIGNED, DID_UNRESOLVABLE, IDENTITY_MISMATCH, KEY_MISMATCH, VALID). Drop the worker-pool thread assertion — thread names in Vert.x worker pools are not stable across versions; the presence of `.runSubscriptionOn()` in the code is the correctness guarantee.

---

## Section 4 — parent#107: SCIM2 integration doc + PLATFORM.md

### New file

`docs/integration/scim2-agent-identity.md` in `casehubio/parent`.

Stable raw URL: `https://raw.githubusercontent.com/casehubio/parent/main/docs/integration/scim2-agent-identity.md`

Contents:
1. **CaseHub SCIM2 schema extension** — schema URI `urn:ietf:params:scim:schemas:extension:casehub:2.0:Agent`, full field table:

   | Field | Type | Required | Description |
   |-------|------|----------|-------------|
   | `did` | String | Yes | DID URI for the agent (e.g. `did:web:example.com:agents:tarkus`) |
   | `clientId` | String | No | OAuth client ID referencing the signing credential location (deferred to #108 JwtVCValidator) |
   | `issuerUri` | String | No | OAuth issuer URI for signing credential verification (deferred to #108 JwtVCValidator) |

   Note: `clientId` and `issuerUri` are defined in the schema for IdP operators to configure ahead of #108. The ledger `ScimAgentResource` record does not currently parse these fields — they are consumed only when `JwtVCValidator (#108)` is implemented.

2. **Field mapping table** — `externalId` → `actorId`, `{did}` → DID URI.

3. **Canonical lookup pattern** — `GET /scim/v2/Agents?filter=externalId eq "{actorId}"`. actorId in filter VALUES only, never in path segments. URL encoding: `URLEncoder.encode(actorId, UTF_8).replace("+", "%20")`.

4. **Caching expectations** — TTL configurable (default 5 min); invalidate on `AgentKeyRotatedEvent` CDI event.

5. **Security constraints** — private keys MUST NOT be stored in SCIM; HTTPS required; auth token is a static deploy-time credential (`@ConfigProperty`).

6. **ConfiguredActorDIDProvider interaction** — when `ScimActorDIDProvider` is activated via `quarkus.arc.selected-alternatives`, configuration-based DID mappings (`casehub.ledger.agent-identity.dids.*`) are ignored. Do not configure both.

7. **IdP-side setup requirements** — the `/scim/v2/Agents` endpoint requires a custom SCIM resource type. Most enterprise SCIM providers (Okta, Azure AD / Entra, JumpCloud) do not enable custom resource types by default. Operators must:
   - Register the `Agent` resource type and schema extension with their IdP
   - Configure the `urn:ietf:params:scim:schemas:extension:casehub:2.0:Agent` schema
   - Populate `externalId` with the `actorId` string for each registered agent
   
   Self-hosted SCIM servers (e.g. Gluu, mid-point, UnboundID) support custom resource types without IdP-side gating.

8. **Example SCIM Agent resource JSON** — `claude:tarkus-reviewer@v1` concrete example.

9. **Link to protocol PP-20260530-bf919d** (`casehub-agent-identity-lookup`).

### PLATFORM.md changes

- **Implementation Protocols table**: add `[SCIM2 agent identity lookup](integration/scim2-agent-identity.md)` row.
- **Capability Ownership table**: update Agent Identity line — note SCIM2 as the resolution mechanism and `ScimActorDIDProvider @Alternative` as the ledger-side implementation.
- **casehub-ledger repository entry**: note `ScimActorDIDProvider @Alternative`.

**No per-repo CLAUDE.md updates** — consuming repos (casehub-eidos, casehub-engine) update their own fetch blocks when they implement SCIM integration.

---

## Coherence review against PLATFORM.md and protocols

| Check | Result |
|-------|--------|
| `ScimActorDIDProvider` stays domain-agnostic (foundation rule) | ✅ |
| `ActorDIDProvider` SPI stays in `api/spi/identity/` | ✅ Unchanged |
| `@Alternative` CDI pattern (Pattern C) | ✅ `@DefaultBean` no-op, `@Alternative` optional impl |
| SCIM actorId in filter value, not path segment (PP-20260530-bf919d) | ✅ URL-encoded filter value, spec. calls out `URLEncoder.replace("+", "%20")` |
| Static credentials via config, not Preferences (`static-credentials-config-property-not-preferences.md`) | ✅ `LedgerConfig.ScimConfig.authToken()` |
| Reactive bridge = `@DefaultBean @Unremovable`, no `@IfBuildProperty` (`reactive-spi-bridge-default-bean.md`) | ✅ |
| All `LedgerEntryEnricher` implementations carry `@Priority` (`ledger-enricher-priority-mandate.md`) | ✅ No new enrichers |
| No Flyway migrations | ✅ |
| `KeyRotationService` no longer imports identity package | ✅ Direct call removed |

---

## Out of scope / follow-on issues

| Item | Action |
|------|--------|
| `publicKeyBytes` extraction from SCIM (DER cert → SubjectPublicKeyInfo) | File issue when concrete consumer exists |
| `InMemoryReactiveActorIdentityBindingRepository` | File separate issue if reactive binding history queries needed |
| `ScimDIDResolver @Alternative` (SCIM as sole source of truth, eliminating external DID hosting) | File as follow-on — current approach is correct for W3C compliance; enterprise operational simplification is a separate design decision |
| `clientId`/`issuerUri` parsing into `ScimAgentResource` | Deferred to #108 JwtVCValidator |
| Per-repo CLAUDE.md updates (eidos, engine) | File when those repos implement SCIM integration |
| `JwtVCValidator` (#108) | Separate issue |
