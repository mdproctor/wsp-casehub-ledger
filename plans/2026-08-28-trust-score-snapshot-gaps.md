# Trust Score Snapshot Gaps — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #203 — feat: trust score snapshot table for historical trust trajectories
**Issue group:** #203

**Goal:** Close gaps between the existing `TrustScoreSnapshot` implementation (#183) and the full spec — add scoreType/dimension columns, capture all four score types, add time-range queries, and add retention config.

**Architecture:** The entity, migration, three repo tiers (JPA/InMemory/NoOp), integration test, and writer wiring in `PerActorTrustComputer` already exist. This work adds missing columns (scoreType, dimensionKey), extends the writer to capture DIMENSION and CAPABILITY_DIMENSION snapshots, adds a time-range query for the compliance report consumer (qhorus#402), and adds optional retention trimming.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (test), PostgreSQL (IT)

## Global Constraints

- All JPQL must use `@NamedQuery` on the entity — never `em.createQuery()` (PP-20260618-51c673)
- No Panache — EntityManager + JPQL only
- Schema changes go directly into V1000 (no production database — see CLAUDE.md schema convention)
- `@Alternative` on JPA repos, `@DefaultBean` on NoOp repos, `@Alternative @Priority(1)` on InMemory repos

## Architectural Decision: No tenancyId on trust_score_snapshot

Issue #203 spec lists `tenancyId` as a column. After reading the architecture:

- Trust computation is cross-tenant — `TrustScoreJob` uses `CrossTenantLedgerEntryRepository.findAllEvents()`
- `ActorTrustScore` has no tenancyId column
- Protocol PP-20260616-05dc6a applies to **per-subject** tables (keyed by `subject_id`), not per-actor tables
- Snapshots record the result of cross-tenant computation — there is no meaningful tenancyId to populate

Adding tenancyId to snapshots without adding it to `ActorTrustScore` would be architecturally inconsistent. If per-tenant trust computation is added in the future, tenancyId should go on `ActorTrustScore` first, and snapshots would follow. For now, the compliance report in qhorus shows cross-tenant trust trajectories per actor — correct for a cross-tenant trust model.

---

## Batch 1: Schema + Entity + Repository tiers

### Task 1: Add scoreType and dimensionKey to TrustScoreSnapshot

**Files:**
- Modify: `runtime/src/main/resources/db/ledger/migration/V1000__ledger_initial_schema.sql:247-258`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/TrustScoreSnapshot.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/TrustScoreSnapshotRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaTrustScoreSnapshotRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/NoOpTrustScoreSnapshotRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryTrustScoreSnapshotRepository.java`
- Test: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreSnapshotIT.java`

**Interfaces:**
- Produces: `TrustScoreSnapshot(String actorId, ScoreType scoreType, String capabilityTag, String dimensionKey, double score, double previousScore, Instant occurredAt)` — new constructor
- Produces: `TrustScoreSnapshotRepository.findByActorAndTimeRange(String actorId, Instant from, Instant to)` — new SPI method
- Produces: `TrustScoreSnapshotRepository.findDimensionSnapshots(String actorId, String dimensionKey)` — new SPI method

- [ ] **Step 1: Update V1000 migration — add columns and update index**

Replace the `trust_score_snapshot` block in V1000:

```sql
-- ── trust_score_snapshot ─────────────────────────────────────────────────────

CREATE TABLE trust_score_snapshot (
    id              UUID             NOT NULL,
    actor_id        VARCHAR(255)     NOT NULL,
    score_type      VARCHAR(50)      NOT NULL,
    capability_tag  VARCHAR(255),
    dimension_key   VARCHAR(255),
    score           DOUBLE PRECISION NOT NULL,
    previous_score  DOUBLE PRECISION NOT NULL,
    occurred_at     TIMESTAMP WITH TIME ZONE NOT NULL,
    CONSTRAINT pk_trust_score_snapshot PRIMARY KEY (id)
);

CREATE INDEX idx_trust_score_snapshot_actor
    ON trust_score_snapshot (actor_id, score_type, occurred_at DESC);
```

- [ ] **Step 2: Update TrustScoreSnapshot entity — add fields, update constructor, add NamedQueries**

Add `scoreType` (enum) and `dimensionKey` (String) fields. Add new `@NamedQuery` annotations for time-range and dimension queries. Update the existing queries to filter by `scoreType`.

```java
@Entity
@Table(name = "trust_score_snapshot")
@NamedQuery(
        name = "TrustScoreSnapshot.findByActorGlobal",
        query = "SELECT s FROM TrustScoreSnapshot s WHERE s.actorId = :actorId"
                + " AND s.scoreType = io.casehub.ledger.api.model.ScoreType.GLOBAL"
                + " ORDER BY s.occurredAt DESC")
@NamedQuery(
        name = "TrustScoreSnapshot.findByActorAndCapability",
        query = "SELECT s FROM TrustScoreSnapshot s WHERE s.actorId = :actorId"
                + " AND s.scoreType = io.casehub.ledger.api.model.ScoreType.CAPABILITY"
                + " AND s.capabilityTag = :capabilityTag ORDER BY s.occurredAt DESC")
@NamedQuery(
        name = "TrustScoreSnapshot.findByActorAndDimension",
        query = "SELECT s FROM TrustScoreSnapshot s WHERE s.actorId = :actorId"
                + " AND s.scoreType = io.casehub.ledger.api.model.ScoreType.DIMENSION"
                + " AND s.dimensionKey = :dimensionKey ORDER BY s.occurredAt DESC")
@NamedQuery(
        name = "TrustScoreSnapshot.findByActorAndTimeRange",
        query = "SELECT s FROM TrustScoreSnapshot s WHERE s.actorId = :actorId"
                + " AND s.occurredAt >= :from AND s.occurredAt <= :to"
                + " ORDER BY s.occurredAt DESC")
public class TrustScoreSnapshot {

    @Id
    @Column(name = "id", nullable = false)
    public UUID id;

    @Column(name = "actor_id", nullable = false)
    public String actorId;

    @Enumerated(EnumType.STRING)
    @Column(name = "score_type", nullable = false)
    public ScoreType scoreType;

    @Column(name = "capability_tag")
    public String capabilityTag;

    @Column(name = "dimension_key")
    public String dimensionKey;

    @Column(name = "score", nullable = false)
    public double score;

    @Column(name = "previous_score", nullable = false)
    public double previousScore;

    @Column(name = "occurred_at", nullable = false)
    public Instant occurredAt;

    protected TrustScoreSnapshot() {}

    public TrustScoreSnapshot(final String actorId, final ScoreType scoreType,
            final String capabilityTag, final String dimensionKey,
            final double score, final double previousScore, final Instant occurredAt) {
        this.id = UUID.randomUUID();
        this.actorId = actorId;
        this.scoreType = scoreType;
        this.capabilityTag = capabilityTag;
        this.dimensionKey = dimensionKey;
        this.score = score;
        this.previousScore = previousScore;
        this.occurredAt = occurredAt;
    }
}
```

- [ ] **Step 3: Update TrustScoreSnapshotRepository SPI — add new methods**

```java
public interface TrustScoreSnapshotRepository {

    void save(TrustScoreSnapshot snapshot);

    List<TrustScoreSnapshot> findGlobalSnapshots(String actorId);

    List<TrustScoreSnapshot> findCapabilitySnapshots(String actorId, String capabilityTag);

    List<TrustScoreSnapshot> findDimensionSnapshots(String actorId, String dimensionKey);

    List<TrustScoreSnapshot> findByActorAndTimeRange(String actorId, Instant from, Instant to);
}
```

- [ ] **Step 4: Update JpaTrustScoreSnapshotRepository — implement new methods**

```java
@Override
public List<TrustScoreSnapshot> findDimensionSnapshots(final String actorId,
        final String dimensionKey) {
    return em.createNamedQuery("TrustScoreSnapshot.findByActorAndDimension", TrustScoreSnapshot.class)
            .setParameter("actorId", actorId)
            .setParameter("dimensionKey", dimensionKey)
            .getResultList();
}

@Override
public List<TrustScoreSnapshot> findByActorAndTimeRange(final String actorId,
        final Instant from, final Instant to) {
    return em.createNamedQuery("TrustScoreSnapshot.findByActorAndTimeRange", TrustScoreSnapshot.class)
            .setParameter("actorId", actorId)
            .setParameter("from", from)
            .setParameter("to", to)
            .getResultList();
}
```

- [ ] **Step 5: Update InMemoryTrustScoreSnapshotRepository — implement new methods, update filters**

Update `findGlobalSnapshots` to filter by `scoreType == GLOBAL` (instead of `capabilityTag == null`). Add `findDimensionSnapshots` and `findByActorAndTimeRange`.

```java
@Override
public List<TrustScoreSnapshot> findGlobalSnapshots(final String actorId) {
    return store.stream()
            .filter(s -> actorId.equals(s.actorId))
            .filter(s -> s.scoreType == ScoreType.GLOBAL)
            .sorted(Comparator.comparing((TrustScoreSnapshot s) -> s.occurredAt).reversed())
            .toList();
}

@Override
public List<TrustScoreSnapshot> findCapabilitySnapshots(final String actorId,
        final String capabilityTag) {
    return store.stream()
            .filter(s -> actorId.equals(s.actorId))
            .filter(s -> s.scoreType == ScoreType.CAPABILITY)
            .filter(s -> capabilityTag.equals(s.capabilityTag))
            .sorted(Comparator.comparing((TrustScoreSnapshot s) -> s.occurredAt).reversed())
            .toList();
}

@Override
public List<TrustScoreSnapshot> findDimensionSnapshots(final String actorId,
        final String dimensionKey) {
    return store.stream()
            .filter(s -> actorId.equals(s.actorId))
            .filter(s -> s.scoreType == ScoreType.DIMENSION)
            .filter(s -> dimensionKey.equals(s.dimensionKey))
            .sorted(Comparator.comparing((TrustScoreSnapshot s) -> s.occurredAt).reversed())
            .toList();
}

@Override
public List<TrustScoreSnapshot> findByActorAndTimeRange(final String actorId,
        final Instant from, final Instant to) {
    return store.stream()
            .filter(s -> actorId.equals(s.actorId))
            .filter(s -> !s.occurredAt.isBefore(from) && !s.occurredAt.isAfter(to))
            .sorted(Comparator.comparing((TrustScoreSnapshot s) -> s.occurredAt).reversed())
            .toList();
}
```

- [ ] **Step 6: Update NoOpTrustScoreSnapshotRepository — add empty implementations**

```java
@Override
public List<TrustScoreSnapshot> findDimensionSnapshots(final String actorId,
        final String dimensionKey) {
    return List.of();
}

@Override
public List<TrustScoreSnapshot> findByActorAndTimeRange(final String actorId,
        final Instant from, final Instant to) {
    return List.of();
}
```

- [ ] **Step 7: Update PerActorTrustComputer — pass ScoreType to constructor, capture DIMENSION and CAP_DIM snapshots**

Update the two existing `snapshotRepo.save()` calls to use the new constructor with `ScoreType`. Add snapshot saves for DIMENSION and CAPABILITY_DIMENSION scores.

For **CAPABILITY** (line 79-80), change to:
```java
snapshotRepo.save(new TrustScoreSnapshot(actorId, ScoreType.CAPABILITY,
        entry.getKey(), null, score.trustScore(), previous, now));
```

For **DIMENSION** (after line 89), add:
```java
final double previousDim = trustRepo.findDimensionScore(actorId, entry.getKey())
        .map(s -> s.trustScore).orElse(0.0);
snapshotRepo.save(new TrustScoreSnapshot(actorId, ScoreType.DIMENSION,
        null, entry.getKey(), entry.getValue(), previousDim, now));
```

For **CAPABILITY_DIMENSION** (after line 101), add:
```java
final double previousCapDim = trustRepo.findCapabilityDimension(
        actorId, capEntry.getKey(), dimEntry.getKey())
        .map(s -> s.trustScore).orElse(0.0);
snapshotRepo.save(new TrustScoreSnapshot(actorId, ScoreType.CAPABILITY_DIMENSION,
        capEntry.getKey(), dimEntry.getKey(), dimEntry.getValue(), previousCapDim, now));
```

For **GLOBAL** (line 115-116), change to:
```java
snapshotRepo.save(new TrustScoreSnapshot(actorId, ScoreType.GLOBAL,
        null, null, global.trustScore(), previousGlobal, now));
```

- [ ] **Step 8: Update existing IT tests — adapt to new constructor**

The existing 3 tests in `TrustScoreSnapshotIT` assert on `capabilityTag` and `previousScore`. Update assertions to also check `scoreType`:

```java
// In firstComputation_capturesSnapshotWithZeroPreviousScore:
assertThat(snapshots.get(0).scoreType).isEqualTo(ScoreType.GLOBAL);

// In capabilitySnapshot_capturedSeparately:
assertThat(capSnapshots.get(0).scoreType).isEqualTo(ScoreType.CAPABILITY);
```

- [ ] **Step 9: Write new IT test — dimension snapshot captured**

Add to `TrustScoreSnapshotIT`:

```java
@Test
@Transactional
void dimensionSnapshot_capturedSeparately() {
    final String actorId = "snapshot-dim-" + UUID.randomUUID();
    final Instant now = Instant.now();

    LedgerTestFixtures.seedDecisionWithDimension(actorId, now.minus(1, ChronoUnit.DAYS),
            "review-thoroughness", 0.85, "code-review", repo, em);

    trustScoreJob.runComputation();

    final List<TrustScoreSnapshot> dimSnapshots =
            snapshotRepo.findDimensionSnapshots(actorId, "review-thoroughness");
    assertThat(dimSnapshots).isNotEmpty();
    assertThat(dimSnapshots.get(0).scoreType).isEqualTo(ScoreType.DIMENSION);
    assertThat(dimSnapshots.get(0).dimensionKey).isEqualTo("review-thoroughness");
    assertThat(dimSnapshots.get(0).score).isGreaterThan(0.0);
}
```

- [ ] **Step 10: Write new IT test — time-range query**

Add to `TrustScoreSnapshotIT`:

```java
@Test
@Transactional
void findByActorAndTimeRange_returnsSnapshotsWithinWindow() {
    final String actorId = "snapshot-range-" + UUID.randomUUID();
    final Instant now = Instant.now();

    LedgerTestFixtures.seedDecision(actorId, now.minus(2, ChronoUnit.DAYS),
            AttestationVerdict.SOUND, repo, em);
    trustScoreJob.runComputation();

    final List<TrustScoreSnapshot> inRange = snapshotRepo.findByActorAndTimeRange(
            actorId, now.minus(1, ChronoUnit.HOURS), now.plus(1, ChronoUnit.HOURS));
    assertThat(inRange).isNotEmpty();
    assertThat(inRange).allSatisfy(s -> {
        assertThat(s.occurredAt).isAfterOrEqualTo(now.minus(1, ChronoUnit.HOURS));
        assertThat(s.occurredAt).isBeforeOrEqualTo(now.plus(1, ChronoUnit.HOURS));
    });

    final List<TrustScoreSnapshot> outOfRange = snapshotRepo.findByActorAndTimeRange(
            actorId, now.minus(5, ChronoUnit.DAYS), now.minus(4, ChronoUnit.DAYS));
    assertThat(outOfRange).isEmpty();
}
```

- [ ] **Step 11: Update PerActorTrustComputerTest — adapt constructor calls**

The unit test at `runtime/src/test/java/io/casehub/ledger/runtime/service/PerActorTrustComputerTest.java` uses `NoOpTrustScoreSnapshotRepository`. No changes needed to the test logic — `NoOpTrustScoreSnapshotRepository` already discards saves. Just verify it compiles cleanly.

- [ ] **Step 12: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all tests pass including the 5 snapshot IT tests (3 existing + 2 new)

- [ ] **Step 13: Commit**

```bash
git add runtime/ persistence-memory/
git commit -m "feat(#203): add scoreType, dimensionKey to TrustScoreSnapshot; capture all score types

Adds score_type and dimension_key columns to trust_score_snapshot. Updates
PerActorTrustComputer to capture DIMENSION and CAPABILITY_DIMENSION snapshots
alongside GLOBAL and CAPABILITY. Adds time-range and dimension query methods
to TrustScoreSnapshotRepository with JPA, InMemory, and NoOp implementations.

Refs #203"
```

---

## Batch 2: Retention config + docs

### Task 2: Snapshot retention config and CLAUDE.md update

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/TrustScoreSnapshotRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaTrustScoreSnapshotRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/NoOpTrustScoreSnapshotRepository.java`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryTrustScoreSnapshotRepository.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/TrustScoreSnapshot.java` (add NamedQuery for delete)
- Modify: `CLAUDE.md` — fix stale V1012 reference
- Test: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreSnapshotIT.java`

**Interfaces:**
- Consumes: `TrustScoreSnapshotRepository` from Task 1
- Produces: `TrustScoreSnapshotRepository.deleteOlderThan(Instant cutoff)` — new SPI method
- Produces: `LedgerConfig.TrustScoreConfig.SnapshotConfig.retentionDays()` — config key

- [ ] **Step 1: Add SnapshotConfig to LedgerConfig.TrustScoreConfig**

Add inside the `TrustScoreConfig` interface:

```java
SnapshotConfig snapshot();

interface SnapshotConfig {
    @WithDefault("365")
    int retentionDays();
}
```

- [ ] **Step 2: Add deleteOlderThan to TrustScoreSnapshotRepository**

```java
int deleteOlderThan(Instant cutoff);
```

- [ ] **Step 3: Add NamedQuery for bulk delete to TrustScoreSnapshot entity**

```java
@NamedQuery(
        name = "TrustScoreSnapshot.deleteOlderThan",
        query = "DELETE FROM TrustScoreSnapshot s WHERE s.occurredAt < :cutoff")
```

- [ ] **Step 4: Implement in JpaTrustScoreSnapshotRepository**

```java
@Override
public int deleteOlderThan(final Instant cutoff) {
    return em.createNamedQuery("TrustScoreSnapshot.deleteOlderThan")
            .setParameter("cutoff", cutoff)
            .executeUpdate();
}
```

- [ ] **Step 5: Implement in InMemoryTrustScoreSnapshotRepository**

```java
@Override
public int deleteOlderThan(final Instant cutoff) {
    final int before = store.size();
    store.removeIf(s -> s.occurredAt.isBefore(cutoff));
    return before - store.size();
}
```

- [ ] **Step 6: Implement in NoOpTrustScoreSnapshotRepository**

```java
@Override
public int deleteOlderThan(final Instant cutoff) {
    return 0;
}
```

- [ ] **Step 7: Wire retention into TrustScoreJob.runComputation()**

Add at the end of `runComputation()`, after the routing signals block:

```java
final int retentionDays = config.trustScore().snapshot().retentionDays();
if (retentionDays > 0) {
    snapshotRepo.deleteOlderThan(now.minus(retentionDays, java.time.temporal.ChronoUnit.DAYS));
}
```

Inject `TrustScoreSnapshotRepository` into `TrustScoreJob`:

```java
@Inject
TrustScoreSnapshotRepository snapshotRepo;
```

- [ ] **Step 8: Write IT test — retention trims old snapshots**

Add to `TrustScoreSnapshotIT`:

```java
@Test
@Transactional
void retention_deletesSnapshotsOlderThanCutoff() {
    final String actorId = "snapshot-retention-" + UUID.randomUUID();
    final Instant now = Instant.now();

    snapshotRepo.save(new TrustScoreSnapshot(actorId, ScoreType.GLOBAL,
            null, null, 0.7, 0.5, now.minus(400, ChronoUnit.DAYS)));
    snapshotRepo.save(new TrustScoreSnapshot(actorId, ScoreType.GLOBAL,
            null, null, 0.8, 0.7, now.minus(100, ChronoUnit.DAYS)));

    final int deleted = snapshotRepo.deleteOlderThan(now.minus(365, ChronoUnit.DAYS));
    assertThat(deleted).isEqualTo(1);

    final List<TrustScoreSnapshot> remaining = snapshotRepo.findGlobalSnapshots(actorId);
    assertThat(remaining).hasSize(1);
    assertThat(remaining.get(0).score).isEqualTo(0.8);
}
```

- [ ] **Step 9: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all tests pass

- [ ] **Step 10: Fix CLAUDE.md — remove stale V1012 reference**

CLAUDE.md references `V1012__trust_score_snapshot.sql` in the project structure and migration listing. Remove these references — the table is defined in V1000. Update the `TrustScoreSnapshotRepository` SPI description to include the new methods.

- [ ] **Step 11: Update consumer guide if snapshot queries are consumer-facing**

Check `docs/guides/consumer-guide.md` — if `TrustScoreSnapshotRepository` is documented there, add the new methods. If not (it's an internal concern), skip.

- [ ] **Step 12: Commit**

```bash
git add runtime/ persistence-memory/ CLAUDE.md docs/
git commit -m "feat(#203): snapshot retention config + fix stale V1012 CLAUDE.md refs

Adds casehub.ledger.trust-score.snapshot.retention-days (default 365) and
wires deleteOlderThan into TrustScoreJob. Fixes CLAUDE.md references to
the now-consolidated V1012 migration.

Closes #203"
```

---

## References

- `runtime/src/main/java/io/casehub/ledger/runtime/model/TrustScoreSnapshot.java` — existing entity
- `runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java` — snapshot writer
- `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java` — scheduled computation
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/TrustScoreSnapshotRepository.java` — SPI
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/ActorTrustScoreRepository.java` — for previous score lookups
- `runtime/src/test/java/io/casehub/ledger/service/TrustScoreSnapshotIT.java` — existing IT
- `runtime/src/main/resources/db/ledger/migration/V1000__ledger_initial_schema.sql:247-258` — schema
- PP-20260616-05dc6a — per-subject table tenancy rule (not applicable: snapshot is per-actor)
- PP-20260618-51c673 — NamedQuery rule (applicable: all new queries use @NamedQuery)
- GitHub #203 — focal issue
- GitHub casehubio/qhorus#402 — downstream consumer (compliance evidence export)
