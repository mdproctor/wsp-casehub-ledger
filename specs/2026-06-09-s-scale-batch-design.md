# S-Scale Batch: #124, #125, #129

Three S-scale issues implemented on a single branch.

---

## #124 — Combined attestation query for ComputedTrustScoreSource

### Problem

`ComputedTrustScoreSource.computeFresh()` and `IncrementalTrustUpdateObserver.onAttestationRecorded()` both use a two-step pattern to load an actor's attestation data:

1. `ledgerRepo.findEventsByActorId(actorId)` → `List<LedgerEntry>`
2. Extract entry IDs → `ledgerRepo.findAttestationsForEntries(entryIds)` → `Map<UUID, List<LedgerAttestation>>`

The repository SPI exposes a leaky abstraction: callers must manually extract entry IDs from the first result and bridge them into the second call. Both methods exist, but the combination requires knowledge that attestations are fetched via entry IDs — an implementation detail the SPI should encapsulate.

The number of SQL queries stays at two (decisions + attestations). The improvement eliminates the manual ID bridging, not a round-trip.

### Design

Add `findAttestationsByActorId(String actorId)` to `CrossTenantLedgerEntryRepository`. Returns `Map<UUID, List<LedgerAttestation>>` — attestations grouped by entry ID, filtered to entries where `entryType = EVENT` and `actorId` matches.

**Blocking SPI** (`CrossTenantLedgerEntryRepository`):
```java
Map<UUID, List<LedgerAttestation>> findAttestationsByActorId(String actorId);
```

**Reactive SPI** (`CrossTenantReactiveLedgerEntryRepository`):
```java
Uni<Map<UUID, List<LedgerAttestation>>> findAttestationsByActorId(String actorId);
```

**@NamedQuery** on `LedgerAttestation` entity (matches existing convention — six named queries already declared on this entity, Hibernate validates at startup):

```java
@NamedQuery(
    name = "LedgerAttestation.findByActorIdEvents",
    query = "SELECT a FROM LedgerAttestation a WHERE a.ledgerEntryId IN ("
          + "SELECT e.id FROM LedgerEntry e WHERE e.actorId = :actorId AND e.entryType = :type)")
```

JPQL subquery — `LedgerAttestation.ledgerEntryId` is a plain UUID field with no `@ManyToOne` mapping to `LedgerEntry`, so an explicit JPQL JOIN is not possible. The DB optimizer converts `IN (subquery)` to a semi-join — equivalent performance.

**JPA implementation** (`JpaCrossTenantLedgerEntryRepository`): uses `em.createNamedQuery("LedgerAttestation.findByActorIdEvents", LedgerAttestation.class)`, matching the existing `findAttestationsForEntries()` pattern which uses `createNamedQuery("LedgerAttestation.findByEntryIds")`.

**Tokenisation:** The JPA implementation must call `actorIdentityProvider.tokeniseForQuery(actorId)` before binding the parameter, identically to `findEventsByActorId()`. Omitting this would silently return empty results when tokenisation is enabled.

**In-memory blocking** (`InMemoryCrossTenantLedgerEntryRepository`): filter `allEntries()` for actorId + EVENT, collect IDs, then filter `allAttestations()` by those IDs.

**In-memory reactive** (`InMemoryCrossTenantReactiveLedgerEntryRepository`): delegate to blocking via `Uni.createFrom().item(...)`.

**Callers updated:**
- `ComputedTrustScoreSource.computeFresh()` — replace manual ID extraction + `findAttestationsForEntries()` with single `findAttestationsByActorId()` call. Still calls `findEventsByActorId()` for the decisions list passed to `calculator.computeAll()`.
- `IncrementalTrustUpdateObserver.onAttestationRecorded()` — same replacement.

### #124 × #125 interaction

When materialization is disabled (#125), `IncrementalTrustUpdateObserver` returns immediately and never reaches its attestation queries. So #124's improvement to the observer's query pattern only benefits deployments with materialization enabled. For computed-only deployments, only `ComputedTrustScoreSource.computeFresh()` benefits from #124. Both changes are correct independently.

### Per-capability query: deferred

The original issue proposed per-capability filtering at the DB level. This is redundant given the per-actor cache in `ComputedTrustScoreSource` — after the first call, all subsequent score queries for the same actor are served from the cached `ComputedScores` without hitting the DB. DB hits are bounded by actor cardinality, not query frequency. The combined query addresses the real API problem (leaky abstraction). Per-capability filtering adds complexity with no measurable benefit at current scale.

`TrustScoreJob.runComputation()` has the same two-step pattern at batch scale (L98–125), but operates on all actors at once via `findAllEvents()` — the per-actor `findAttestationsByActorId()` does not apply there. The batch pattern is correctly left unchanged.

---

## #125 — Materialization flag for computed-only deployments

### Problem

When `ComputedTrustScoreSource` is active, `TrustScoreJob` and `IncrementalTrustUpdateObserver` still run — computing and writing scores to `ActorTrustScoreRepository` that nobody reads. This is harmless but wasteful for lightweight deployments.

The existing `casehub.ledger.trust-score.enabled` flag gates the entire trust feature, including config defaults. A deployment using `ComputedTrustScoreSource` may still want `trust-score.enabled=true` for configuration (decay, aggregation strategy) while skipping materialization.

### Design choice: runtime gating, not build-time CDI exclusion

Issue #125 proposed CDI exclusion (`ExcludedTypeBuildItem` at build time). This spec uses runtime guard clauses instead. Rationale:

- **Operational flexibility:** runtime toggleability without rebuild. A deployment can switch between materialized and computed strategies by changing config, without repackaging.
- **Consistency:** `TrustScoreJob` and `IncrementalTrustUpdateObserver` already use runtime config gates (`config.trustScore().enabled()`). Adding a second gate in the same pattern is natural.
- **Independence:** `@Alternative`-based source selection (`selected-alternatives`) is a build-time deployment decision. Adding a second build-time config property creates coupling between two independent config axes — the source selection and the materialization toggle would both need to be set consistently at build time.

### Config

Add `materialization` sub-config to `LedgerConfig.TrustScoreConfig`:

```java
interface MaterializationConfig {
    @WithDefault("true")
    boolean enabled();
}
```

Config key: `casehub.ledger.trust-score.materialization.enabled` (default `true`).

**When `false`:**
- `TrustScoreJob.computeTrustScores()` returns immediately after the `enabled` check — skips `runComputation()` entirely (Beta, bootstrap, EigenTrust, routing publisher).
- `IncrementalTrustUpdateObserver.onAttestationRecorded()` returns immediately — adds `materialization.enabled` to the existing gate check.

**When `true` (default):** no behaviour change.

`TrustScoreRoutingPublisher` is not gated — it only fires when `TrustScoreJob` calls `publish()`. With materialization disabled, the job never reaches `publish()`.

### Deployment matrix

| trust-score.enabled | materialization.enabled | TrustScoreSource | Effect |
|---------------------|------------------------|------------------|--------|
| false | (irrelevant) | any | No computation at all — job/observer skip |
| true | true (default) | Materialized/Cached | Full write path — current behaviour |
| true | false | Computed | Config active, no writes — computed reads only |
| true | true | Computed | Both paths run independently — see below |

**Row 4 — both paths active:** valid for migration validation. Run materialized and computed paths in parallel to verify that computed scores match materialized ones before cutting over. Also needed when `TrustExportService` or `TrustScoreRoutingPublisher` consumers depend on the materialized store while reads use the computed path — exports read from `ActorTrustScoreRepository` directly, not through `TrustScoreSource`.

---

## #129 — Carry tenancyId through CDI event chain

### Problem

`ActorIdentityBindingObserver` persists `ActorIdentityBindingEntry` via `@ObservesAsync` without tenant context. The `@ObservesAsync` handler has no CDI request scope, so there is no way to resolve tenancyId from `CurrentPrincipal`. The interim fix defaults to `TenancyConstants.DEFAULT_TENANT_ID` in `JpaActorIdentityBindingRepository.save()`.

### Design

**1. casehub-platform-api — event records gain tenancyId**

Per governing protocol PP-20260601-e368ea (tenancyId as 2nd component):

```java
// Before
public record AgentIdentityValidatedEvent(
    String actorId, String actorDid, IdentityBindingStatus status,
    boolean alsoKnownAsVerified, boolean keyMatchVerified,
    String verifiedKeyRef, CredentialValidationResult credentialResult,
    String didMethod) {}

// After
public record AgentIdentityValidatedEvent(
    String actorId, String tenancyId, String actorDid, IdentityBindingStatus status,
    boolean alsoKnownAsVerified, boolean keyMatchVerified,
    String verifiedKeyRef, CredentialValidationResult credentialResult,
    String didMethod) {}
```

Same for `AgentIdentityViolationEvent`:

```java
// Before
public record AgentIdentityViolationEvent(
    String actorId, String actorDid, IdentityBindingStatus status) {}

// After
public record AgentIdentityViolationEvent(
    String actorId, String tenancyId, String actorDid,
    IdentityBindingStatus status) {}
```

**2. ledger runtime — ActorIdentityValidationEnricher.fireEvent()**

Pass `entry.tenancyId` when constructing both event types:

```java
// VALID path
event.fireAsync(new AgentIdentityValidatedEvent(
    entry.actorId, entry.tenancyId, entry.actorDid, status,
    true, true, entry.agentKeyRef, null, didMethod));

// Violation path
event.fireAsync(new AgentIdentityViolationEvent(
    entry.actorId, entry.tenancyId, entry.actorDid, status));
```

**3. ledger runtime — ActorIdentityBindingObserver**

Both handlers must pass tenancyId. `persistBinding()` gains a `tenancyId` parameter as the first argument:

```java
void onValidated(@ObservesAsync AgentIdentityValidatedEvent event) {
    persistBinding(
        event.tenancyId(), event.actorId(), event.actorDid(), event.status(),
        event.alsoKnownAsVerified(), event.keyMatchVerified(),
        event.verifiedKeyRef(), event.credentialResult(), event.didMethod());
}

void onViolation(@ObservesAsync AgentIdentityViolationEvent event) {
    persistBinding(
        event.tenancyId(), event.actorId(), event.actorDid(), event.status(),
        false, false, null, null, extractDidMethod(event.actorDid()));
}
```

The `persistBinding()` method body sets `entry.tenancyId` explicitly alongside the other 12 fields:

```java
@Transactional(REQUIRES_NEW)
void persistBinding(
        final String tenancyId,
        final String actorId,
        ...) {
    try {
        final ActorIdentityBindingEntry entry = new ActorIdentityBindingEntry();
        entry.id = UUID.randomUUID();
        entry.tenancyId = tenancyId;
        entry.subjectId = UUID.nameUUIDFromBytes(actorId.getBytes(StandardCharsets.UTF_8));
        entry.actorId = actorId;
        // ... remaining 10 fields unchanged ...
        repository.save(entry);
    } catch (final Exception e) {
        LOG.warnf("ActorIdentityBindingObserver failed to persist binding for %s: %s",
            actorId, e.getMessage());
    }
}
```

**4. JpaActorIdentityBindingRepository.save() — remove fallback**

Remove the `if (entry.tenancyId == null)` fallback (L51–53). The tenancyId is now always provided through the event chain. A null tenancyId at save time is a bug — let it fail visibly rather than silently defaulting.

### Cross-repo impact

Cross-repo search confirms **no consumers exist outside ledger.** Searched engine, work, qhorus, platform, and eidos — `AgentIdentityValidatedEvent` and `AgentIdentityViolationEvent` are defined in `casehub-platform-api` and consumed only by `ActorIdentityValidationEnricher` (fires) and `ActorIdentityBindingObserver` (observes) in `casehub-ledger`. The compile break from the record component addition is fully contained to this repo.

### Cross-repo sequence

1. Edit `AgentIdentityValidatedEvent` and `AgentIdentityViolationEvent` in `casehub-platform` repo.
2. `mvn install` casehub-platform-api to publish SNAPSHOT to local `.m2`.
3. Consume in ledger — update enricher, observer, repository.
