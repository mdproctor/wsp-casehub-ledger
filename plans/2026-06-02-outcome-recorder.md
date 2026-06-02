# Lightweight Outcome-Tracking Mode Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add `OutcomeRecorder` / `ReactiveOutcomeRecorder` APIs that write a `LedgerEntry` + `LedgerAttestation` in a single atomic call, enabling trust-score-driven plugin routing in game-loop contexts without the compliance overhead stack.

**Architecture:** A pure-Java `OutcomeRecord` value type in `api/model/` captures the write payload. `DefaultOutcomeRecorder` (blocking, `@DefaultBean`) delegates transactional writes to a package-private `OutcomeRecordSaveService`. `BlockingToReactiveOutcomeRecorder` wraps the blocking impl on a worker pool. An `EigenTrustStartupValidator` logs a WARNING when eigentrust is enabled with too few pre-trusted actors. Trust freshness for routing is governed by `TrustScoreJob` schedule + `TrustScoreRoutingPublisher` — no incremental pipeline needed for QuarkMind.

**Tech Stack:** Java 21 records, Quarkus CDI (`@DefaultBean`, `@ApplicationScoped`), SmallRye Mutiny (`Uni`), JBoss Logging, Flyway (one new migration), JUnit 5 + AssertJ, `@QuarkusTest` with isolated H2 profiles.

**Spec correction:** `runtime/model/LedgerEntry` is abstract. `OutcomeRecordSaveService.buildEntry()` cannot instantiate it directly. A concrete subclass `PlainLedgerEntry` with Flyway migration `V1009` is required. The spec's module impact "Schema: No migrations" is incorrect — this plan adds `V1009`.

---

## File Map

| Action | File | Purpose |
|--------|------|---------|
| Create | `api/src/main/java/io/casehub/ledger/api/model/OutcomeRecord.java` | Value type for one decision+outcome write |
| Create | `api/src/main/java/io/casehub/ledger/api/spi/OutcomeRecorder.java` | Blocking write interface |
| Create | `api/src/main/java/io/casehub/ledger/api/spi/ReactiveOutcomeRecorder.java` | Reactive write interface |
| Modify | `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java` | Add `OutcomeConfig` inner interface |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/model/PlainLedgerEntry.java` | Concrete `LedgerEntry` subclass for generic event writes |
| Create | `runtime/src/main/resources/db/ledger/migration/V1009__plain_ledger_entry.sql` | Join table for `PlainLedgerEntry` |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/AttestorDefaults.java` | Package-private record holding resolved attestor id+type |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/OutcomeRecordSaveService.java` | Package-private `@Transactional` bean; isolation layer for transaction demarcation |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultOutcomeRecorder.java` | `@DefaultBean` blocking `OutcomeRecorder` impl |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/BlockingToReactiveOutcomeRecorder.java` | `@DefaultBean` reactive bridge |
| Create | `runtime/src/main/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidator.java` | Startup WARNING if eigentrust misconfigured |
| Modify | `runtime/src/test/resources/application.properties` | Add `outcome-recorder-test` and `outcome-no-attestor-test` profiles |
| Create | `api/src/test/java/io/casehub/ledger/api/model/OutcomeRecordTest.java` | Pure JUnit 5 unit tests for `OutcomeRecord` |
| Create | `runtime/src/test/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidatorTest.java` | Pure JUnit 5 unit tests for `shouldWarn()` (same package for access) |
| Create | `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerConfidenceTest.java` | Pure JUnit 5 unit test for confidence weighting ratio |
| Create | `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java` | Integration: blocking write + trust score |
| Create | `runtime/src/test/java/io/casehub/ledger/service/ReactiveOutcomeRecorderIT.java` | Integration: reactive write end-to-end |
| Create | `runtime/src/test/java/io/casehub/ledger/service/MultiCapabilityIT.java` | Integration: 4-plugin capability-differentiated scores |
| Create | `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderDefaultAttestorIT.java` | Integration: config-default attestor is applied |
| Create | `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderUnconfiguredAttestorIT.java` | Integration: missing attestor config throws `IllegalStateException` |

---

## Task 1: `OutcomeRecord` — unit test then implement

**Files:**
- Create: `api/src/test/java/io/casehub/ledger/api/model/OutcomeRecordTest.java`
- Create: `api/src/main/java/io/casehub/ledger/api/model/OutcomeRecord.java`

Build and run commands (from repo root):
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -q
```

- [ ] **Step 1.1: Write the failing unit test**

Create `api/src/test/java/io/casehub/ledger/api/model/OutcomeRecordTest.java`:

```java
package io.casehub.ledger.api.model;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static io.casehub.ledger.api.model.AttestationVerdict.SOUND;
import static io.casehub.ledger.api.model.AttestationVerdict.FLAGGED;

import java.time.Instant;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;

class OutcomeRecordTest {

    private final UUID subjectId = UUID.randomUUID();

    // ── Required field validation ──────────────────────────────────────────────

    @Test
    void nullActorId_throwsNPE() {
        assertThatThrownBy(() -> OutcomeRecord.of(null, subjectId, "strategy", SOUND, 0.7))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("actorId");
    }

    @Test
    void nullSubjectId_throwsNPE() {
        assertThatThrownBy(() -> OutcomeRecord.of("actor", null, "strategy", SOUND, 0.7))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("subjectId");
    }

    @Test
    void nullCapabilityTag_throwsNPE() {
        assertThatThrownBy(() -> OutcomeRecord.of("actor", subjectId, null, SOUND, 0.7))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("capabilityTag");
    }

    @Test
    void nullVerdict_throwsNPE() {
        assertThatThrownBy(() -> OutcomeRecord.of("actor", subjectId, "strategy", null, 0.7))
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("verdict");
    }

    // ── Confidence validation ─────────────────────────────────────────────────

    @Test
    void confidenceZero_throwsIAE() {
        assertThatThrownBy(() -> OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.0))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("confidence must be in (0.0, 1.0]");
    }

    @Test
    void confidenceAboveOne_throwsIAE() {
        assertThatThrownBy(() -> OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 1.1))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("confidence must be in (0.0, 1.0]");
    }

    @Test
    void confidenceOne_accepted() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 1.0);
        assertThat(r.confidence()).isEqualTo(1.0);
    }

    @Test
    void confidencePointSeven_accepted() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThat(r.confidence()).isEqualTo(0.7);
    }

    // ── Defaults ──────────────────────────────────────────────────────────────

    @Test
    void nullActorType_defaultsToAgent() {
        // Direct canonical constructor — bypass factory to pass null actorType
        OutcomeRecord r = new OutcomeRecord("actor", subjectId, SOUND, 0.7,
                "strategy", null, null, null, null, null);
        assertThat(r.actorType()).isEqualTo(ActorType.AGENT);
    }

    // ── Attestor pair invariant ───────────────────────────────────────────────

    @Test
    void attestorIdSetTypeNull_throwsIAE() {
        assertThatThrownBy(() -> new OutcomeRecord("actor", subjectId, SOUND, 0.7,
                "strategy", ActorType.AGENT, null, null, "some-attestor", null))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("attestorId and attestorType must be provided together");
    }

    @Test
    void attestorTypeSetIdNull_throwsIAE() {
        assertThatThrownBy(() -> new OutcomeRecord("actor", subjectId, SOUND, 0.7,
                "strategy", ActorType.AGENT, null, null, null, ActorType.SYSTEM))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("attestorId and attestorType must be provided together");
    }

    @Test
    void bothAttestorFieldsNull_accepted() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThat(r.attestorId()).isNull();
        assertThat(r.attestorType()).isNull();
    }

    // ── Factory methods ───────────────────────────────────────────────────────

    @Test
    void of_setsCapabilityTag() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThat(r.capabilityTag()).isEqualTo("strategy");
        assertThat(r.actorType()).isEqualTo(ActorType.AGENT);
        assertThat(r.actorRole()).isNull();
        assertThat(r.occurredAt()).isNull();
        assertThat(r.attestorId()).isNull();
    }

    @Test
    void ofGlobal_setsGlobalCapabilityTag() {
        OutcomeRecord r = OutcomeRecord.ofGlobal("actor", subjectId, SOUND, 0.7);
        assertThat(r.capabilityTag()).isEqualTo(CapabilityTag.GLOBAL);
    }

    // ── with* methods ─────────────────────────────────────────────────────────

    @Test
    void withActorType_null_throwsNPE() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThatThrownBy(() -> r.withActorType(null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void withActorType_setsType_preservesOtherFields() {
        OutcomeRecord orig = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        OutcomeRecord updated = orig.withActorType(ActorType.HUMAN);
        assertThat(updated.actorType()).isEqualTo(ActorType.HUMAN);
        assertThat(updated.actorId()).isEqualTo("actor");
        assertThat(updated.capabilityTag()).isEqualTo("strategy");
        assertThat(updated.confidence()).isEqualTo(0.7);
    }

    @Test
    void withActorRole_null_throwsNPE() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThatThrownBy(() -> r.withActorRole(null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void withActorRole_setsRole() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7)
                .withActorRole("strategist");
        assertThat(r.actorRole()).isEqualTo("strategist");
    }

    @Test
    void withOccurredAt_null_throwsNPE() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThatThrownBy(() -> r.withOccurredAt(null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void withOccurredAt_setsTimestamp() {
        Instant ts = Instant.parse("2026-06-02T10:00:00Z");
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7)
                .withOccurredAt(ts);
        assertThat(r.occurredAt()).isEqualTo(ts);
    }

    @Test
    void withAttestor_null_id_throwsNPE() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThatThrownBy(() -> r.withAttestor(null, ActorType.SYSTEM))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void withAttestor_null_type_throwsNPE() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        assertThatThrownBy(() -> r.withAttestor("attestor-id", null))
                .isInstanceOf(NullPointerException.class);
    }

    @Test
    void withAttestor_setsBothFields() {
        OutcomeRecord r = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7)
                .withAttestor("quarkmind:game-engine@v1", ActorType.SYSTEM);
        assertThat(r.attestorId()).isEqualTo("quarkmind:game-engine@v1");
        assertThat(r.attestorType()).isEqualTo(ActorType.SYSTEM);
    }

    @Test
    void withMethods_returnNewInstances() {
        OutcomeRecord orig = OutcomeRecord.of("actor", subjectId, "strategy", SOUND, 0.7);
        OutcomeRecord updated = orig.withActorType(ActorType.HUMAN);
        assertThat(updated).isNotSameAs(orig);
        assertThat(orig.actorType()).isEqualTo(ActorType.AGENT); // original unchanged
    }
}
```

- [ ] **Step 1.2: Run test — expect compilation failure** (`OutcomeRecord` does not exist)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -q 2>&1 | grep -E "ERROR|cannot find symbol" | head -10
```

Expected: `error: cannot find symbol` for `OutcomeRecord`.

- [ ] **Step 1.3: Implement `OutcomeRecord`**

Create `api/src/main/java/io/casehub/ledger/api/model/OutcomeRecord.java`:

```java
package io.casehub.ledger.api.model;

import java.time.Instant;
import java.util.Objects;
import java.util.UUID;

import io.casehub.platform.api.identity.ActorType;

/**
 * Captures a single plugin decision and its outcome for recording via {@link io.casehub.ledger.api.spi.OutcomeRecorder}.
 *
 * <p>Use {@link #of(String, UUID, String, AttestationVerdict, double)} for routing-aware writes.
 * Use {@link #ofGlobal(String, UUID, AttestationVerdict, double)} only when capability-differentiated
 * routing is not the goal — GLOBAL-scoped attestations do not reach {@code TrustScoreCache}
 * and therefore do not influence {@code TrustWeightedAgentStrategy}.
 *
 * <p>{@code capabilityTag} is required in the primary factory because omitting it silently
 * breaks routing; the compiler enforces routing correctness.
 *
 * <p>Confidence in (0.0, 1.0]. Recommended values: 0.1 (tick-level), 0.7 (game-level), 1.0 (session).
 * Higher confidence = stronger attestation weight in Bayesian Beta computation.
 */
public record OutcomeRecord(
        String actorId,
        UUID subjectId,
        AttestationVerdict verdict,
        double confidence,
        String capabilityTag,
        ActorType actorType,
        String actorRole,
        Instant occurredAt,
        String attestorId,
        ActorType attestorType
) {
    public OutcomeRecord {
        Objects.requireNonNull(actorId,       "actorId required");
        Objects.requireNonNull(subjectId,     "subjectId required");
        Objects.requireNonNull(verdict,       "verdict required");
        Objects.requireNonNull(capabilityTag, "capabilityTag required — use CapabilityTag.GLOBAL if intentional");
        if (confidence <= 0.0 || confidence > 1.0) {
            throw new IllegalArgumentException(
                    "confidence must be in (0.0, 1.0] — got " + confidence
                            + ". Recommended: 0.1 (tick), 0.7 (game), 1.0 (session).");
        }
        if ((attestorId == null) != (attestorType == null)) {
            throw new IllegalArgumentException(
                    "attestorId and attestorType must be provided together or both left null.");
        }
        if (actorType == null) {
            actorType = ActorType.AGENT;
        }
    }

    /**
     * Primary factory for routing-aware outcome recording.
     * capabilityTag is required — GLOBAL-scoped attestations do not reach TrustScoreCache.
     *
     * <p>Example: {@code OutcomeRecord.of("quarkmind:strategy@v1", gameId, "strategy", SOUND, 0.7)}
     */
    public static OutcomeRecord of(final String actorId, final UUID subjectId,
            final String capabilityTag, final AttestationVerdict verdict, final double confidence) {
        return new OutcomeRecord(actorId, subjectId, verdict, confidence,
                capabilityTag, ActorType.AGENT, null, null, null, null);
    }

    /**
     * Factory for outcomes that intentionally target the global Beta score only.
     * These do NOT reach TrustScoreCache or TrustWeightedAgentStrategy.
     */
    public static OutcomeRecord ofGlobal(final String actorId, final UUID subjectId,
            final AttestationVerdict verdict, final double confidence) {
        return new OutcomeRecord(actorId, subjectId, verdict, confidence,
                CapabilityTag.GLOBAL, ActorType.AGENT, null, null, null, null);
    }

    /** @throws NullPointerException if role is null — actorRole is nullable at construction but not clearable via with* */
    public OutcomeRecord withActorRole(final String role) {
        Objects.requireNonNull(role, "role");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, role, occurredAt, attestorId, attestorType);
    }

    /**
     * @throws NullPointerException if t is null.
     * Pass {@code ActorType.AGENT} explicitly to reset to the default rather than passing null.
     */
    public OutcomeRecord withActorType(final ActorType t) {
        Objects.requireNonNull(t, "actorType — use ActorType.AGENT to set the default explicitly");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                t, actorRole, occurredAt, attestorId, attestorType);
    }

    /** @throws NullPointerException if ts is null */
    public OutcomeRecord withOccurredAt(final Instant ts) {
        Objects.requireNonNull(ts, "occurredAt");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, actorRole, ts, attestorId, attestorType);
    }

    /**
     * Override the attestor for this record. Both id and type must be non-null —
     * they are always set together to maintain the pair invariant.
     */
    public OutcomeRecord withAttestor(final String id, final ActorType t) {
        Objects.requireNonNull(id, "attestorId");
        Objects.requireNonNull(t,  "attestorType");
        return new OutcomeRecord(actorId, subjectId, verdict, confidence, capabilityTag,
                actorType, actorRole, occurredAt, id, t);
    }
}
```

- [ ] **Step 1.4: Run test — expect all green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -q
```

Expected: `BUILD SUCCESS`, all `OutcomeRecordTest` tests pass.

- [ ] **Step 1.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add api/src/main/java/io/casehub/ledger/api/model/OutcomeRecord.java api/src/test/java/io/casehub/ledger/api/model/OutcomeRecordTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#114): add OutcomeRecord Java record with compact constructor validation

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: `OutcomeRecorder` + `ReactiveOutcomeRecorder` interfaces

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/spi/OutcomeRecorder.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/ReactiveOutcomeRecorder.java`

No separate test — interfaces; tested through DefaultOutcomeRecorder integration tests.

- [ ] **Step 2.1: Create `OutcomeRecorder`**

```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.model.OutcomeRecord;

/**
 * Records a plugin decision and its outcome as a single atomic operation.
 *
 * <p>The default implementation is {@code DefaultOutcomeRecorder} ({@code @DefaultBean}).
 * Applications may substitute a custom implementation by declaring an {@code @ApplicationScoped}
 * bean implementing this interface — CDI will prefer it over the {@code @DefaultBean}.
 *
 * @see ReactiveOutcomeRecorder for the non-blocking variant
 */
public interface OutcomeRecorder {

    /**
     * Write a {@link OutcomeRecord} as both a {@code LedgerEntry} (EVENT) and a
     * {@code LedgerAttestation}. Both writes commit in the same transaction.
     * For JPA consumers, both are visible in the database before this method returns.
     *
     * @throws IllegalStateException if {@code record.attestorId()} is null and
     *         {@code casehub.ledger.outcome.default-attestor-id} is not configured
     */
    void record(OutcomeRecord record);
}
```

- [ ] **Step 2.2: Create `ReactiveOutcomeRecorder`**

```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.model.OutcomeRecord;
import io.smallrye.mutiny.Uni;

/**
 * Non-blocking variant of {@link OutcomeRecorder}.
 *
 * <p>The default implementation wraps {@link OutcomeRecorder} on the Mutiny worker pool —
 * safe to call from the Vert.x event loop. Callers must subscribe to the returned {@code Uni}.
 *
 * <p>Failures (including {@code IllegalStateException} for unconfigured attestor) are
 * emitted as Uni failures, not thrown synchronously.
 */
public interface ReactiveOutcomeRecorder {

    /** @see OutcomeRecorder#record(OutcomeRecord) */
    Uni<Void> record(OutcomeRecord record);
}
```

- [ ] **Step 2.3: Build api module to confirm compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 2.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add api/src/main/java/io/casehub/ledger/api/spi/OutcomeRecorder.java api/src/main/java/io/casehub/ledger/api/spi/ReactiveOutcomeRecorder.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#114): add OutcomeRecorder and ReactiveOutcomeRecorder SPI interfaces

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: `LedgerConfig.OutcomeConfig` — config group for attestor defaults

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`

- [ ] **Step 3.1: Add `OutcomeConfig` to `LedgerConfig`**

In `LedgerConfig.java`, add a new top-level method alongside the existing ones (e.g., after `agentSigning()`):

```java
    /**
     * Combined write API settings for {@link io.casehub.ledger.runtime.service.DefaultOutcomeRecorder}.
     *
     * @return outcome sub-configuration
     */
    OutcomeConfig outcome();
```

And add the inner interface at the end of the file, before the closing `}`:

```java
    /** Settings for the {@link io.casehub.ledger.runtime.service.DefaultOutcomeRecorder} combined write API. */
    interface OutcomeConfig {

        /**
         * Default attestor identity used when {@code OutcomeRecord.attestorId()} is null.
         * For QuarkMind: {@code "quarkmind:game-engine@v1"}.
         *
         * <p>If this is absent and {@code OutcomeRecord.attestorId()} is also null,
         * {@code DefaultOutcomeRecorder.record()} throws {@code IllegalStateException} at call time.
         *
         * @return the default attestor ID, or empty if not configured
         */
        java.util.Optional<String> defaultAttestorId();

        /**
         * Default attestor type used when {@code OutcomeRecord.attestorType()} is null.
         * Ignored when {@code defaultAttestorId} is empty.
         *
         * @return SYSTEM by default
         */
        @WithDefault("SYSTEM")
        io.casehub.platform.api.identity.ActorType defaultAttestorType();
    }
```

- [ ] **Step 3.2: Build runtime module to check for config validation errors**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,runtime -am -q -DskipTests
```

Expected: `BUILD SUCCESS`. If `SRCFG00050` errors appear, the `@WithDefault` annotation or type import needs adjustment.

- [ ] **Step 3.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#114): add LedgerConfig.OutcomeConfig for attestor defaults

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: `PlainLedgerEntry` + Flyway migration

**Background:** `runtime/model/LedgerEntry` is abstract. `OutcomeRecordSaveService.buildEntry()` must instantiate a concrete subclass. `PlainLedgerEntry` is a minimal `@Entity` subclass with no extra columns — its join table has only the FK to `ledger_entry`.

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/PlainLedgerEntry.java`
- Create: `runtime/src/main/resources/db/ledger/migration/V1009__plain_ledger_entry.sql`

- [ ] **Step 4.1: Write the Flyway migration**

Create `runtime/src/main/resources/db/ledger/migration/V1009__plain_ledger_entry.sql`:

```sql
-- Plain concrete subclass of LedgerEntry for domain-agnostic event writes (OutcomeRecorder).
-- No domain-specific columns — all payload lives in ledger_entry and ledger_attestation.
CREATE TABLE plain_ledger_entry (
    id UUID NOT NULL,
    CONSTRAINT pk_plain_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_plain_ledger_entry FOREIGN KEY (id) REFERENCES ledger_entry (id)
);
```

- [ ] **Step 4.2: Create `PlainLedgerEntry`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/model/PlainLedgerEntry.java`:

```java
package io.casehub.ledger.runtime.model;

import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

/**
 * Concrete {@link LedgerEntry} subclass for domain-agnostic event writes.
 *
 * <p>Used by {@link io.casehub.ledger.runtime.service.OutcomeRecordSaveService} to persist
 * generic EVENT entries via {@link io.casehub.ledger.api.spi.OutcomeRecorder}. Has no
 * domain-specific fields — all payload lives in {@code ledger_entry} and
 * {@code ledger_attestation}.
 *
 * <p>In-memory deployments ({@code casehub-ledger-memory}) do not use Hibernate; this class
 * works there as a plain Java object with no schema requirement.
 */
@Entity
@Table(name = "plain_ledger_entry")
@DiscriminatorValue("PLAIN")
public class PlainLedgerEntry extends LedgerEntry {
}
```

- [ ] **Step 4.3: Build runtime to verify Hibernate entity discovery**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,runtime -am -q -DskipTests
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 4.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/model/PlainLedgerEntry.java \
  runtime/src/main/resources/db/ledger/migration/V1009__plain_ledger_entry.sql
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#114): add PlainLedgerEntry and V1009 migration for generic event writes

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: `AttestorDefaults` + `OutcomeRecordSaveService` + `DefaultOutcomeRecorder`

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/AttestorDefaults.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/OutcomeRecordSaveService.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultOutcomeRecorder.java`

All three are in the same package — `AttestorDefaults` is package-private so only `OutcomeRecordSaveService` and `DefaultOutcomeRecorder` can use it.

- [ ] **Step 5.1: Create `AttestorDefaults`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/AttestorDefaults.java`:

```java
package io.casehub.ledger.runtime.service;

import io.casehub.platform.api.identity.ActorType;

/** Resolved attestor identity — either from {@code OutcomeRecord} directly or from config defaults. */
record AttestorDefaults(String attestorId, ActorType attestorType) {
}
```

- [ ] **Step 5.2: Create `OutcomeRecordSaveService`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/OutcomeRecordSaveService.java`:

```java
package io.casehub.ledger.runtime.service;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.PlainLedgerEntry;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;

/**
 * Transactional inner service for {@link DefaultOutcomeRecorder}.
 *
 * <p>Package-private: not part of the public API. Exists solely to provide a clean
 * {@code @Transactional} boundary that commits before {@code DefaultOutcomeRecorder.record()}
 * returns — preventing race conditions if a future async trust update observer fires before
 * the writes are visible. See casehubio/ledger#115.
 *
 * <p>Quarkus ArC applies the {@code @Transactional} interceptor to package-private methods
 * via bytecode enhancement — no proxy required.
 */
@ApplicationScoped
class OutcomeRecordSaveService {

    @Inject
    LedgerEntryRepository ledgerRepo;

    @Transactional
    void save(final OutcomeRecord record, final AttestorDefaults attestor) {
        final LedgerEntry entry = buildEntry(record);
        ledgerRepo.save(entry);  // repo assigns sequenceNumber, UUID, occurredAt, enriches

        final LedgerAttestation attestation = buildAttestation(record, entry, attestor);
        ledgerRepo.saveAttestation(attestation);
    }

    private PlainLedgerEntry buildEntry(final OutcomeRecord record) {
        final PlainLedgerEntry entry = new PlainLedgerEntry();
        entry.actorId    = record.actorId();
        entry.actorRole  = record.actorRole();
        entry.actorType  = record.actorType();
        entry.subjectId  = record.subjectId();
        entry.entryType  = LedgerEntryType.EVENT;
        entry.occurredAt = record.occurredAt();  // null → @PrePersist fills at persist time
        return entry;
    }

    private LedgerAttestation buildAttestation(final OutcomeRecord record,
            final LedgerEntry saved, final AttestorDefaults attestor) {
        final LedgerAttestation a = new LedgerAttestation();
        a.id             = UUID.randomUUID();
        a.ledgerEntryId  = saved.id;
        a.subjectId      = saved.subjectId;
        a.attestorId     = attestor.attestorId();
        a.attestorType   = attestor.attestorType();
        a.verdict        = record.verdict();
        a.confidence     = record.confidence();
        a.capabilityTag  = record.capabilityTag();
        a.occurredAt     = record.occurredAt();  // null → LedgerAttestation @PrePersist fills it
        return a;
    }
}
```

- [ ] **Step 5.3: Create `DefaultOutcomeRecorder`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultOutcomeRecorder.java`:

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.quarkus.arc.DefaultBean;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.casehub.ledger.runtime.config.LedgerConfig;

/**
 * Default blocking implementation of {@link OutcomeRecorder}.
 *
 * <p>{@code @DefaultBean} — replaced by any {@code @ApplicationScoped} bean implementing
 * {@link OutcomeRecorder} that the consuming application provides.
 *
 * <p>Transaction boundary: this method is NOT {@code @Transactional}. It delegates to
 * {@link OutcomeRecordSaveService#save} which IS {@code @Transactional} and commits before
 * returning. This ensures that for JPA consumers both writes are committed and visible before
 * this method returns. For in-memory consumers {@code @Transactional} is a no-op.
 */
@DefaultBean
@ApplicationScoped
public class DefaultOutcomeRecorder implements OutcomeRecorder {

    @Inject
    OutcomeRecordSaveService saveService;

    @Inject
    LedgerConfig config;

    @Override
    public void record(final OutcomeRecord record) {
        final AttestorDefaults attestor = resolveAttestor(record);
        saveService.save(record, attestor);
    }

    private AttestorDefaults resolveAttestor(final OutcomeRecord record) {
        if (record.attestorId() != null) {
            return new AttestorDefaults(record.attestorId(), record.attestorType());
        }
        final String id = config.outcome().defaultAttestorId().orElseThrow(() ->
                new IllegalStateException(
                        "OutcomeRecord.attestorId is null and "
                                + "casehub.ledger.outcome.default-attestor-id is not configured. "
                                + "Set one of them before calling record()."));
        final ActorType type = config.outcome().defaultAttestorType();
        return new AttestorDefaults(id, type);
    }
}
```

- [ ] **Step 5.4: Build runtime to confirm compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,runtime -am -q -DskipTests
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 5.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/AttestorDefaults.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/OutcomeRecordSaveService.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/DefaultOutcomeRecorder.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#114): add DefaultOutcomeRecorder, OutcomeRecordSaveService, AttestorDefaults

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: `BlockingToReactiveOutcomeRecorder` — reactive bridge

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/BlockingToReactiveOutcomeRecorder.java`

- [ ] **Step 6.1: Create the bridge**

```java
package io.casehub.ledger.runtime.service;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import io.quarkus.arc.DefaultBean;
import io.smallrye.mutiny.Uni;
import io.smallrye.mutiny.infrastructure.Infrastructure;

import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.casehub.ledger.api.spi.ReactiveOutcomeRecorder;

/**
 * Default reactive bridge — wraps {@link OutcomeRecorder} on the Mutiny worker pool.
 *
 * <p>{@code @DefaultBean} with no {@code @IfBuildProperty} gate. This bridge has no
 * Hibernate Reactive dependency and must be active under all profiles. Per the
 * {@code reactive-spi-bridge-default-bean} platform protocol: bridges are always active;
 * native async adapters activate via {@code @Alternative @Priority(N)}.
 *
 * <p>Callers on the Vert.x event loop use this safely — the blocking delegate runs on the
 * worker pool, not the calling thread.
 */
@DefaultBean
@ApplicationScoped
public class BlockingToReactiveOutcomeRecorder implements ReactiveOutcomeRecorder {

    @Inject
    OutcomeRecorder blocking;

    @Override
    public Uni<Void> record(final OutcomeRecord record) {
        return Uni.createFrom()
                .item(() -> {
                    blocking.record(record);
                    return null;
                })
                .runSubscriptionOn(Infrastructure.getDefaultWorkerPool());
    }
}
```

- [ ] **Step 6.2: Build to confirm**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,runtime -am -q -DskipTests
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 6.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/main/java/io/casehub/ledger/runtime/service/BlockingToReactiveOutcomeRecorder.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#114): add BlockingToReactiveOutcomeRecorder worker-pool bridge

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: `EigenTrustStartupValidator` — unit test then implement

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidator.java`
- Create: `runtime/src/test/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidatorTest.java`

Note: The unit test is in `io.casehub.ledger.runtime.service` — the same package as the validator — to access the package-private `shouldWarn()` method. This is an intentional exception to the project's usual test package convention.

- [ ] **Step 7.1: Write the failing unit test**

Create `runtime/src/test/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidatorTest.java`:

```java
package io.casehub.ledger.runtime.service;

import static org.assertj.core.api.Assertions.assertThat;

import org.junit.jupiter.api.Test;

/**
 * Pure JUnit 5 unit tests for {@link EigenTrustStartupValidator#shouldWarn}.
 * In {@code io.casehub.ledger.runtime.service} to access the package-private static method.
 */
class EigenTrustStartupValidatorTest {

    @Test
    void eigentrustDisabled_noWarn() {
        assertThat(EigenTrustStartupValidator.shouldWarn(false, 0)).isFalse();
    }

    @Test
    void eigentrustEnabled_zeroActors_warns() {
        assertThat(EigenTrustStartupValidator.shouldWarn(true, 0)).isTrue();
    }

    @Test
    void eigentrustEnabled_oneActor_warns() {
        assertThat(EigenTrustStartupValidator.shouldWarn(true, 1)).isTrue();
    }

    @Test
    void eigentrustEnabled_twoActors_warns() {
        assertThat(EigenTrustStartupValidator.shouldWarn(true, 2)).isTrue();
    }

    @Test
    void eigentrustEnabled_threeActors_noWarn() {
        assertThat(EigenTrustStartupValidator.shouldWarn(true, 3)).isFalse();
    }

    @Test
    void eigentrustEnabled_manyActors_noWarn() {
        assertThat(EigenTrustStartupValidator.shouldWarn(true, 10)).isFalse();
    }
}
```

- [ ] **Step 7.2: Run unit test — expect compilation failure**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EigenTrustStartupValidatorTest -q 2>&1 | grep -E "ERROR|cannot find" | head -5
```

Expected: compilation error — `EigenTrustStartupValidator` does not exist.

- [ ] **Step 7.3: Create `EigenTrustStartupValidator`**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidator.java`:

```java
package io.casehub.ledger.runtime.service;

import java.util.List;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import org.jboss.logging.Logger;

import io.quarkus.runtime.StartupEvent;

import io.casehub.ledger.runtime.config.LedgerConfig;

/**
 * Logs a WARNING at startup if EigenTrust is enabled with an insufficient pre-trusted actor set.
 *
 * <p>EigenTrust requires a well-connected attestation graph with at least 3 pre-trusted actors
 * to converge correctly. A star graph (one attestor, N decision-makers) with fewer than 3
 * pre-trusted actors produces degenerate scores or non-convergent iteration. See ADR 0016.
 *
 * <p>{@link #shouldWarn} is package-private for direct unit testing.
 */
@ApplicationScoped
public class EigenTrustStartupValidator {

    private static final Logger LOG = Logger.getLogger(EigenTrustStartupValidator.class);

    @Inject
    LedgerConfig config;

    void onStart(@Observes final StartupEvent ev) {
        final boolean eigenEnabled = config.trustScore().eigentrust().enabled();
        final List<String> actors = config.trustScore().eigentrust().preTrustedActors()
                .orElse(List.of());
        if (shouldWarn(eigenEnabled, actors.size())) {
            LOG.warn("casehub-ledger: EigenTrust is enabled but pre-trusted-actors has fewer "
                    + "than 3 entries. EigenTrust is inappropriate for small agent graphs or "
                    + "single-attestor deployments — results may be degenerate or non-convergent. "
                    + "Disable with: casehub.ledger.trust-score.eigentrust.enabled=false "
                    + "(the default). See ADR 0016.");
        }
    }

    /** Package-private for unit testing without CDI. */
    static boolean shouldWarn(final boolean eigenTrustEnabled, final int preTrustedActorCount) {
        return eigenTrustEnabled && preTrustedActorCount < 3;
    }
}
```

- [ ] **Step 7.4: Run unit test — expect all green**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=EigenTrustStartupValidatorTest -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 7.5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/main/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidator.java \
  runtime/src/test/java/io/casehub/ledger/runtime/service/EigenTrustStartupValidatorTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "feat(#114): add EigenTrustStartupValidator with startup warning for small graphs

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: Test infrastructure — profiles + confidence unit test

**Files:**
- Modify: `runtime/src/test/resources/application.properties`
- Create: `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerConfidenceTest.java`

- [ ] **Step 8.1: Add test profiles to `application.properties`**

Append to `runtime/src/test/resources/application.properties`:

```properties
# Outcome recorder test profile — trust scoring enabled, default attestor configured
# Isolated DB prevents PK collisions with TrustScoreIT and other tests
%outcome-recorder-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:outcomerecordertestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%outcome-recorder-test.casehub.ledger.trust-score.enabled=true
%outcome-recorder-test.casehub.ledger.hash-chain.enabled=false
%outcome-recorder-test.casehub.ledger.outcome.default-attestor-id=quarkmind:game-engine@v1
%outcome-recorder-test.casehub.ledger.outcome.default-attestor-type=SYSTEM
%outcome-recorder-test.quarkus.scheduler.enabled=false

# No-attestor profile — for testing the unconfigured attestor error path
# default-attestor-id intentionally absent
%outcome-no-attestor-test.quarkus.datasource.jdbc.url=jdbc:h2:mem:outcomenoattestortestdb;MODE=PostgreSQL;DB_CLOSE_DELAY=-1
%outcome-no-attestor-test.casehub.ledger.trust-score.enabled=true
%outcome-no-attestor-test.casehub.ledger.hash-chain.enabled=false
%outcome-no-attestor-test.quarkus.scheduler.enabled=false
```

- [ ] **Step 8.2: Write `TrustScoreComputerConfidenceTest`**

Create `runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerConfidenceTest.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.LedgerAttestation;
import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.service.TrustScoreComputer;

/**
 * Pure JUnit 5 unit test verifying that {@code confidence} scales the Bayesian Beta
 * weight contribution proportionally. No Quarkus runtime, no CDI.
 *
 * <p>TrustScoreComputer initialises α = β = 1.0 (Jeffreys prior). After one SOUND
 * attestation at confidence C with zero age (weight = 1.0), stored alpha = 1.0 + C.
 * The ratio (A.alpha - 1.0) / (B.alpha - 1.0) equals the confidence ratio.
 */
class TrustScoreComputerConfidenceTest {

    private static final int HALF_LIFE_DAYS = 90;
    private final TrustScoreComputer computer = new TrustScoreComputer(HALF_LIFE_DAYS);
    private final Instant now = Instant.now();

    // Concrete subclass for testing (LedgerEntry is abstract)
    private static class EventEntry extends LedgerEntry {
    }

    private EventEntry decision(final String actorId) {
        final EventEntry e = new EventEntry();
        e.id = UUID.randomUUID();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.occurredAt = now;
        return e;
    }

    private LedgerAttestation attestation(final UUID entryId,
            final AttestationVerdict verdict, final double confidence) {
        final LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = entryId;
        a.subjectId = UUID.randomUUID();
        a.attestorId = "test-attestor";
        a.attestorType = ActorType.SYSTEM;
        a.verdict = verdict;
        a.confidence = confidence;
        a.capabilityTag = "strategy";
        a.occurredAt = now;  // zero age → decay weight = 1.0
        return a;
    }

    @Test
    void gameConfidence_contributes7xMoreThanTickConfidence() {
        // Actor A: game-level (confidence = 0.7)
        final EventEntry decisionA = decision("actor-A");
        final LedgerAttestation attA = attestation(decisionA.id, AttestationVerdict.SOUND, 0.7);
        final TrustScoreComputer.ActorScore scoreA = computer.compute(
                List.of(decisionA), Map.of(decisionA.id, List.of(attA)), now);

        // Actor B: tick-level (confidence = 0.1)
        final EventEntry decisionB = decision("actor-B");
        final LedgerAttestation attB = attestation(decisionB.id, AttestationVerdict.SOUND, 0.1);
        final TrustScoreComputer.ActorScore scoreB = computer.compute(
                List.of(decisionB), Map.of(decisionB.id, List.of(attB)), now);

        // Prior is 1.0 for both; subtract to isolate the attestation contribution
        final double contributionA = scoreA.alpha() - 1.0;
        final double contributionB = scoreB.alpha() - 1.0;

        assertThat(contributionA).isCloseTo(0.7, within(0.001));
        assertThat(contributionB).isCloseTo(0.1, within(0.001));
        assertThat(contributionA / contributionB).isCloseTo(7.0, within(0.01));
    }

    @Test
    void highConfidence_yieldsHigherScore() {
        final EventEntry d = decision("actor");
        final LedgerAttestation highConf = attestation(d.id, AttestationVerdict.SOUND, 1.0);
        final TrustScoreComputer.ActorScore score = computer.compute(
                List.of(d), Map.of(d.id, List.of(highConf)), now);
        assertThat(score.trustScore()).isGreaterThan(0.5);
    }
}
```

- [ ] **Step 8.3: Run confidence unit test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreComputerConfidenceTest -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 8.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/test/resources/application.properties \
  runtime/src/test/java/io/casehub/ledger/service/TrustScoreComputerConfidenceTest.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#114): add outcome-recorder test profiles and confidence weighting unit test

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 9: `OutcomeRecorderIT` — blocking write + trust score end-to-end

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java`

Test package `io.casehub.ledger.service` is the project convention for integration tests. `TrustScoreJob.runComputation()` is `public` — no package-private access required.

- [ ] **Step 9.1: Write the failing integration test**

Create `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.ledger.api.model.AttestationVerdict.SOUND;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration test: blocking OutcomeRecorder writes entry + attestation and
 * trust score is computable after TrustScoreJob.runComputation().
 */
@QuarkusTest
@TestProfile(OutcomeRecorderIT.Profile.class)
class OutcomeRecorderIT {

    public static class Profile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "outcome-recorder-test";
        }
    }

    @Inject OutcomeRecorder recorder;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject ActorTrustScoreRepository trustRepo;
    @Inject TrustScoreJob trustScoreJob;

    @Test
    void record_writesEntryAndAttestation_thenScoreComputedAfterJob() {
        final String pluginId = "quarkmind:strategy@v1-" + UUID.randomUUID();
        final UUID gameId = UUID.randomUUID();

        recorder.record(OutcomeRecord.of(pluginId, gameId, "strategy", SOUND, 0.7));

        // Verify LedgerEntry was written
        final var entry = ledgerRepo.findEntryById(
                ledgerRepo.findBySubjectId(gameId).get(0).id);
        assertThat(entry).isPresent();
        assertThat(entry.get().actorId).isEqualTo(pluginId);
        assertThat(entry.get().entryType).isEqualTo(LedgerEntryType.EVENT);

        // Verify attestation was written
        final var attestations = ledgerRepo.findAttestationsByEntryId(entry.get().id);
        assertThat(attestations).hasSize(1);
        final var att = attestations.get(0);
        assertThat(att.verdict).isEqualTo(SOUND);
        assertThat(att.confidence).isEqualTo(0.7);
        assertThat(att.capabilityTag).isEqualTo("strategy");
        assertThat(att.attestorId).isEqualTo("quarkmind:game-engine@v1");

        // Trigger batch trust computation
        trustScoreJob.runComputation();

        // Verify CAPABILITY score exists and is > 0.5 (one SOUND attestation → positive score)
        final var score = trustRepo.findCapabilityScore(pluginId, "strategy");
        assertThat(score).isPresent();
        assertThat(score.get().trustScore).isGreaterThan(0.5);
    }
}
```

- [ ] **Step 9.2: Run failing test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=OutcomeRecorderIT -q 2>&1 | grep -E "FAILED|ERROR|Tests run" | head -10
```

Expected: `FAILED` — `OutcomeRecorder` is in CDI graph but has no JPA implementation (we're using in-memory via the `InMemoryLedgerEntryRepository` which is `@Alternative @Priority(1)`). Actually, wait: the default test profile selects JPA repos. Let me check...

Looking at `application.properties`: `quarkus.arc.selected-alternatives=io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository,...`. The `outcome-recorder-test` profile doesn't override `selected-alternatives`, so JPA repos are active with H2.

The `OutcomeRecordSaveService` calls `ledgerRepo.save(entry)` where `entry` is a `PlainLedgerEntry`. Flyway migration `V1009` creates the `plain_ledger_entry` table. The test should pass once the implementation is complete.

Since the full runtime is already built (Task 5), the test should NOW run and pass (all implementation code exists). If it fails, check:
- Flyway migration ran (V1009 creates `plain_ledger_entry`)
- `PlainLedgerEntry` is registered as a Hibernate entity (verify `@Entity` annotation)

- [ ] **Step 9.3: If test passes, commit. If not, diagnose.**

Likely failures and fixes:
- `NullPointerException` on `entry.id` in `buildAttestation` → `ledgerRepo.save(entry)` did not assign `id` → check that `LedgerEntry @PrePersist` runs (for JPA, Hibernate calls `@PrePersist`; for in-memory, `InMemoryLedgerEntryRepository.save()` sets `entry.id`)
- `Flyway migration error` → check SQL syntax in V1009

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=OutcomeRecorderIT -q
```

Expected: `BUILD SUCCESS`, 1 test passed.

- [ ] **Step 9.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#114): add OutcomeRecorderIT — blocking write + trust score end-to-end

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 10: `ReactiveOutcomeRecorderIT`

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/ReactiveOutcomeRecorderIT.java`

- [ ] **Step 10.1: Write and run the test**

Create `runtime/src/test/java/io/casehub/ledger/service/ReactiveOutcomeRecorderIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.ledger.api.model.AttestationVerdict.SOUND;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.ReactiveOutcomeRecorder;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration test: ReactiveOutcomeRecorder returns a Uni that completes after both writes commit.
 * Uses the same outcome-recorder-test profile as OutcomeRecorderIT.
 */
@QuarkusTest
@TestProfile(OutcomeRecorderIT.Profile.class)
class ReactiveOutcomeRecorderIT {

    @Inject ReactiveOutcomeRecorder reactiveRecorder;
    @Inject LedgerEntryRepository ledgerRepo;
    @Inject ActorTrustScoreRepository trustRepo;
    @Inject TrustScoreJob trustScoreJob;

    @Test
    void reactiveRecord_completesAfterBothWritesCommit() {
        final String pluginId = "quarkmind:economics@v1-" + UUID.randomUUID();
        final UUID gameId = UUID.randomUUID();

        // Subscribe and wait for completion — no timeout needed in integration tests
        reactiveRecorder.record(
                OutcomeRecord.of(pluginId, gameId, "economics", SOUND, 0.7)
        ).await().indefinitely();

        // Same post-conditions as OutcomeRecorderIT
        final var entries = ledgerRepo.findBySubjectId(gameId);
        assertThat(entries).hasSize(1);
        assertThat(entries.get(0).actorId).isEqualTo(pluginId);
        assertThat(entries.get(0).entryType).isEqualTo(LedgerEntryType.EVENT);

        final var attestations = ledgerRepo.findAttestationsByEntryId(entries.get(0).id);
        assertThat(attestations).hasSize(1);
        assertThat(attestations.get(0).capabilityTag).isEqualTo("economics");
        assertThat(attestations.get(0).confidence).isEqualTo(0.7);

        trustScoreJob.runComputation();

        final var score = trustRepo.findCapabilityScore(pluginId, "economics");
        assertThat(score).isPresent();
        assertThat(score.get().trustScore).isGreaterThan(0.5);
    }
}
```

- [ ] **Step 10.2: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=ReactiveOutcomeRecorderIT -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 10.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/test/java/io/casehub/ledger/service/ReactiveOutcomeRecorderIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#114): add ReactiveOutcomeRecorderIT

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 11: `MultiCapabilityIT` — 4-plugin capability-differentiated scores

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/MultiCapabilityIT.java`

- [ ] **Step 11.1: Write and run the test**

Create `runtime/src/test/java/io/casehub/ledger/service/MultiCapabilityIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.ledger.api.model.AttestationVerdict.SOUND;
import static io.casehub.ledger.api.model.AttestationVerdict.FLAGGED;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.casehub.ledger.runtime.repository.ActorTrustScoreRepository;
import io.casehub.ledger.runtime.service.TrustScoreJob;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration test: 4 plugins with different win/loss ratios get distinct CAPABILITY scores.
 * Strategy (9W/1L) must score higher than Economics (3W/7L).
 */
@QuarkusTest
@TestProfile(OutcomeRecorderIT.Profile.class)
class MultiCapabilityIT {

    @Inject OutcomeRecorder recorder;
    @Inject ActorTrustScoreRepository trustRepo;
    @Inject TrustScoreJob trustScoreJob;

    @Test
    void fourPlugins_distinctCapabilityScores_strategyBeatsEconomics() {
        final UUID gameId = UUID.randomUUID();
        final String strategy  = "quarkmind:strategy@v1-"  + UUID.randomUUID();
        final String economics = "quarkmind:economics@v1-" + UUID.randomUUID();
        final String tactics   = "quarkmind:tactics@v1-"   + UUID.randomUUID();
        final String scouting  = "quarkmind:scouting@v1-"  + UUID.randomUUID();

        // Strategy: 9 SOUND, 1 FLAGGED
        for (int i = 0; i < 9; i++) {
            recorder.record(OutcomeRecord.of(strategy, UUID.randomUUID(), "strategy", SOUND, 0.7));
        }
        recorder.record(OutcomeRecord.of(strategy, UUID.randomUUID(), "strategy", FLAGGED, 0.7));

        // Economics: 3 SOUND, 7 FLAGGED (consistently bad)
        for (int i = 0; i < 3; i++) {
            recorder.record(OutcomeRecord.of(economics, UUID.randomUUID(), "economics", SOUND, 0.7));
        }
        for (int i = 0; i < 7; i++) {
            recorder.record(OutcomeRecord.of(economics, UUID.randomUUID(), "economics", FLAGGED, 0.7));
        }

        // Tactics: 5 SOUND, 5 FLAGGED (neutral)
        for (int i = 0; i < 5; i++) {
            recorder.record(OutcomeRecord.of(tactics, UUID.randomUUID(), "tactics", SOUND, 0.7));
            recorder.record(OutcomeRecord.of(tactics, UUID.randomUUID(), "tactics", FLAGGED, 0.7));
        }

        // Scouting: 7 SOUND, 3 FLAGGED
        for (int i = 0; i < 7; i++) {
            recorder.record(OutcomeRecord.of(scouting, UUID.randomUUID(), "scouting", SOUND, 0.7));
        }
        for (int i = 0; i < 3; i++) {
            recorder.record(OutcomeRecord.of(scouting, UUID.randomUUID(), "scouting", FLAGGED, 0.7));
        }

        trustScoreJob.runComputation();

        final double strategyScore  = trustRepo.findCapabilityScore(strategy,  "strategy").orElseThrow().trustScore;
        final double economicsScore = trustRepo.findCapabilityScore(economics, "economics").orElseThrow().trustScore;
        final double tacticsScore   = trustRepo.findCapabilityScore(tactics,   "tactics").orElseThrow().trustScore;
        final double scoutingScore  = trustRepo.findCapabilityScore(scouting,  "scouting").orElseThrow().trustScore;

        assertThat(strategyScore).isGreaterThan(economicsScore);
        assertThat(scoutingScore).isGreaterThan(economicsScore);

        // Distinct CAPABILITY rows exist for all four
        assertThat(trustRepo.findCapabilityScore(strategy,  "strategy")).isPresent();
        assertThat(trustRepo.findCapabilityScore(economics, "economics")).isPresent();
        assertThat(trustRepo.findCapabilityScore(tactics,   "tactics")).isPresent();
        assertThat(trustRepo.findCapabilityScore(scouting,  "scouting")).isPresent();

        // Tactics at 50% should be near neutral (0.5) — just verify it exists and is distinct
        assertThat(tacticsScore).isGreaterThan(0.0);
    }
}
```

- [ ] **Step 11.2: Run test**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=MultiCapabilityIT -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 11.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add runtime/src/test/java/io/casehub/ledger/service/MultiCapabilityIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#114): add MultiCapabilityIT — 4-plugin capability-differentiated scores

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 12: Attestor config tests — default attestor and unconfigured error path

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderDefaultAttestorIT.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderUnconfiguredAttestorIT.java`

- [ ] **Step 12.1: Write `OutcomeRecorderDefaultAttestorIT`**

Create `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderDefaultAttestorIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;
import static io.casehub.ledger.api.model.AttestationVerdict.SOUND;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration test: when OutcomeRecord has no explicit attestorId,
 * DefaultOutcomeRecorder resolves it from config.
 */
@QuarkusTest
@TestProfile(OutcomeRecorderIT.Profile.class)
class OutcomeRecorderDefaultAttestorIT {

    @Inject OutcomeRecorder recorder;
    @Inject LedgerEntryRepository ledgerRepo;

    @Test
    void nullAttestorId_usesConfigDefault() {
        final UUID gameId = UUID.randomUUID();
        // OutcomeRecord.of() does not call withAttestor() — attestorId is null
        recorder.record(OutcomeRecord.of("quarkmind:strategy@v1", gameId, "strategy", SOUND, 0.7));

        final var entry = ledgerRepo.findBySubjectId(gameId).get(0);
        final var attestations = ledgerRepo.findAttestationsByEntryId(entry.id);
        assertThat(attestations).hasSize(1);
        // Default attestor is configured in outcome-recorder-test profile
        assertThat(attestations.get(0).attestorId).isEqualTo("quarkmind:game-engine@v1");
    }

    @Test
    void explicitAttestorId_overridesConfigDefault() {
        final UUID gameId = UUID.randomUUID();
        recorder.record(
                OutcomeRecord.of("quarkmind:strategy@v1", gameId, "strategy", SOUND, 0.7)
                        .withAttestor("custom-attestor@v1",
                                io.casehub.platform.api.identity.ActorType.SYSTEM));

        final var entry = ledgerRepo.findBySubjectId(gameId).get(0);
        final var attestations = ledgerRepo.findAttestationsByEntryId(entry.id);
        assertThat(attestations.get(0).attestorId).isEqualTo("custom-attestor@v1");
    }
}
```

- [ ] **Step 12.2: Write `OutcomeRecorderUnconfiguredAttestorIT`**

Create `runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderUnconfiguredAttestorIT.java`:

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThatThrownBy;
import static io.casehub.ledger.api.model.AttestationVerdict.SOUND;

import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.api.model.OutcomeRecord;
import io.casehub.ledger.api.spi.OutcomeRecorder;
import io.quarkus.test.junit.QuarkusTest;
import io.quarkus.test.junit.QuarkusTestProfile;
import io.quarkus.test.junit.TestProfile;

/**
 * Integration test: when both OutcomeRecord.attestorId and config default are absent,
 * DefaultOutcomeRecorder.record() throws IllegalStateException at call time.
 *
 * Uses a dedicated test profile that deliberately omits default-attestor-id.
 */
@QuarkusTest
@TestProfile(OutcomeRecorderUnconfiguredAttestorIT.NoAttestorProfile.class)
class OutcomeRecorderUnconfiguredAttestorIT {

    public static class NoAttestorProfile implements QuarkusTestProfile {
        @Override
        public String getConfigProfile() {
            return "outcome-no-attestor-test";
        }
    }

    @Inject OutcomeRecorder recorder;

    @Test
    void nullAttestorId_noConfigDefault_throwsIllegalState() {
        assertThatThrownBy(() ->
                recorder.record(OutcomeRecord.of("plugin", UUID.randomUUID(), "strategy", SOUND, 0.7)))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("default-attestor-id is not configured");
    }
}
```

- [ ] **Step 12.3: Run both tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="OutcomeRecorderDefaultAttestorIT,OutcomeRecorderUnconfiguredAttestorIT" -q
```

Expected: `BUILD SUCCESS`.

- [ ] **Step 12.4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/ledger add \
  runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderDefaultAttestorIT.java \
  runtime/src/test/java/io/casehub/ledger/service/OutcomeRecorderUnconfiguredAttestorIT.java
git -C /Users/mdproctor/claude/casehub/ledger commit -m "test(#114): add attestor config tests — default resolved and unconfigured error

Co-Authored-By: Claude Sonnet 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Task 13: Full test suite run + build

- [ ] **Step 13.1: Run all tests across all modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test 2>&1 | tail -30
```

Expected: All modules build and test cleanly. Watch for:
- Any Flyway version conflicts (`V1009` clashing with another branch's migration) — check `git fetch && git log origin/main --oneline -5`
- `SRCFG00050` errors — malformed `@ConfigMapping` in `LedgerConfig.OutcomeConfig`
- `AmbiguousResolutionException` — if any existing test activates a `@Alternative` that conflicts with `DefaultOutcomeRecorder`

- [ ] **Step 13.2: If tests fail, diagnose and fix before continuing**

Common fixes:
- JPA `UnsatisfiedResolutionException` for `OutcomeRecorder` — check that `DefaultOutcomeRecorder` is `@DefaultBean` (not just `@ApplicationScoped`)
- `NullPointerException` on `saved.id` in `buildAttestation` — ensure `LedgerEntry @PrePersist` fired before `buildAttestation` uses `saved.id`. For JPA, `em.persist(entry)` triggers `@PrePersist` which sets `entry.id`. For in-memory, `InMemoryLedgerEntryRepository.save()` sets `entry.id` at line 67. Either way, `id` should be set before `buildAttestation` is called.
- `Flyway migration checksum error` — migration file was modified after first run; delete H2 mem DB by restarting (H2 in-memory resets per test profile)

---

## Self-Review Checklist

After completing all tasks, verify:

- [ ] `OutcomeRecord` fields match exactly what `OutcomeRecordSaveService.buildEntry()` and `buildAttestation()` consume — all field names use the Java record accessor form (`record.actorId()`, not `.actorId`)
- [ ] `PlainLedgerEntry` discriminator value `"PLAIN"` does not collide with any existing `@DiscriminatorValue` — check `TEST` (TestEntry), `KEY_ROTATION` (KeyRotationEntry), `IDENTITY_BINDING` (ActorIdentityBindingEntry)
- [ ] `V1009__plain_ledger_entry.sql` version does not collide — verify `git grep "V1009" runtime/src/main/resources/`
- [ ] `LedgerConfig.outcome()` method name matches `%outcome-recorder-test.casehub.ledger.outcome.default-attestor-id` key derivation
- [ ] `EigenTrustStartupValidator` uses `org.jboss.logging.Logger` (not `java.util.logging`)
- [ ] All integration tests use `@TestProfile` — none rely on the default profile
- [ ] `BlockingToReactiveOutcomeRecorder` has NO `@IfBuildProperty` — per `reactive-spi-bridge-default-bean` protocol
