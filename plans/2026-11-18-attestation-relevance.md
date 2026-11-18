# Attestation Relevance Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #231 — feat: org-scoped attestation weighting — weight trust by org proximity
**Issue group:** #231

**Goal:** Add an extensible per-attestation relevance weighting SPI to the trust computation pipeline, with write-time snapshot support and mass-preserving normalisation.

**Architecture:** Ledger ships the SPI (`AttestationRelevanceFactor`) in api, a `RelevanceComposer` aggregator in ledger-core, and a `LedgerAttestationEnricher` pipeline for write-time snapshots. The trust computation pipeline gains a fourth weight factor (relevance) with per-pass mass-preserving normalisation inside `TrustScoreComputer`. The org-proximity implementation is shipped by eidos (separate repo/issue) — this plan covers ledger only.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA/Hibernate, H2 (tests), Spring Boot (dual-framework parity)

## Global Constraints

- Java 21 source level, Java 26 JVM
- All schema changes go into base migration `V1000__ledger_initial_schema.sql` (no incremental scripts — no existing installations)
- ORM mappings must be updated in BOTH `runtime/src/main/resources/META-INF/orm.xml` AND `ledger-spring-jpa/src/main/resources/META-INF/orm.xml`
- Pre-release: breaking changes cost nothing — no compatibility shims
- All new types use `ide_insert_member` / `ide_replace_member` for structural editing
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl <module>`
- Ledger has zero eidos dependency — no imports from `io.casehub.eidos.*`

---

## Batch 1: SPI Foundation + Schema

All new types, records, and interfaces. After this batch: the api and ledger-core modules compile with the new types, schema includes the snapshot column, and all new types have unit tests.

### Task 1: RelevanceAssessment record + AttestationRelevanceContext record

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/spi/RelevanceAssessment.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/AttestationRelevanceContext.java`
- Test: `api/src/test/java/io/casehub/ledger/api/spi/RelevanceAssessmentTest.java`
- Test: `api/src/test/java/io/casehub/ledger/api/spi/AttestationRelevanceContextTest.java`

**Interfaces:**
- Consumes: `ActorType` from `io.casehub.platform.api.identity`
- Produces: `RelevanceAssessment(double weight, String reason)` with `NEUTRAL` constant; `AttestationRelevanceContext(UUID, String, ActorType, String, String, String, String, String, Instant, Map<String,Object>)` record

- [ ] **Step 1: Write RelevanceAssessment tests**

```java
package io.casehub.ledger.api.spi;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class RelevanceAssessmentTest {

    @Test
    void neutral_hasWeight1() {
        assertEquals(1.0, RelevanceAssessment.NEUTRAL.weight());
        assertNull(RelevanceAssessment.NEUTRAL.reason());
    }

    @Test
    void validWeight_accepted() {
        var a = new RelevanceAssessment(0.7, "org-distance:3");
        assertEquals(0.7, a.weight());
        assertEquals("org-distance:3", a.reason());
    }

    @Test
    void zeroWeight_accepted() {
        var a = new RelevanceAssessment(0.0, null);
        assertEquals(0.0, a.weight());
    }

    @Test
    void negativeWeight_rejected() {
        assertThrows(IllegalArgumentException.class,
                () -> new RelevanceAssessment(-0.1, null));
    }

    @Test
    void aboveOneWeight_rejected() {
        assertThrows(IllegalArgumentException.class,
                () -> new RelevanceAssessment(1.01, null));
    }

    @Test
    void nanWeight_rejected() {
        assertThrows(IllegalArgumentException.class,
                () -> new RelevanceAssessment(Double.NaN, null));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=RelevanceAssessmentTest -DfailIfNoTests=false`
Expected: compilation error — `RelevanceAssessment` does not exist

- [ ] **Step 3: Implement RelevanceAssessment**

```java
package io.casehub.ledger.api.spi;

public record RelevanceAssessment(double weight, String reason) {

    public static final RelevanceAssessment NEUTRAL = new RelevanceAssessment(1.0, null);

    public RelevanceAssessment {
        if (Double.isNaN(weight) || weight < 0.0 || weight > 1.0) {
            throw new IllegalArgumentException("weight must be in [0.0, 1.0], got: " + weight);
        }
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=RelevanceAssessmentTest`
Expected: all 6 tests PASS

- [ ] **Step 5: Write AttestationRelevanceContext test**

```java
package io.casehub.ledger.api.spi;

import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class AttestationRelevanceContextTest {

    @Test
    void construction_withNullSnapshot() {
        var ctx = new AttestationRelevanceContext(
                UUID.randomUUID(), "attestor-1", ActorType.AGENT, "reviewer",
                "actor-1", "tenant-1", "review", null, Instant.now(), null);
        assertNull(ctx.snapshot());
        assertEquals("attestor-1", ctx.attestorId());
    }

    @Test
    void construction_withSnapshot() {
        var snapshot = Map.<String, Object>of("org-distance", 2);
        var ctx = new AttestationRelevanceContext(
                UUID.randomUUID(), "attestor-1", ActorType.AGENT, "reviewer",
                "actor-1", "tenant-1", "review", "thoroughness", Instant.now(), snapshot);
        assertEquals(2, ctx.snapshot().get("org-distance"));
    }
}
```

- [ ] **Step 6: Implement AttestationRelevanceContext**

```java
package io.casehub.ledger.api.spi;

import io.casehub.platform.api.identity.ActorType;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;

public record AttestationRelevanceContext(
        UUID attestationId,
        String attestorId,
        ActorType attestorType,
        String attestorRole,
        String scoredActorId,
        String tenancyId,
        String capabilityTag,
        String trustDimension,
        Instant occurredAt,
        Map<String, Object> snapshot
) {}
```

- [ ] **Step 7: Run all api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/spi/RelevanceAssessment.java api/src/main/java/io/casehub/ledger/api/spi/AttestationRelevanceContext.java api/src/test/java/io/casehub/ledger/api/spi/RelevanceAssessmentTest.java api/src/test/java/io/casehub/ledger/api/spi/AttestationRelevanceContextTest.java
git commit -m "feat(api): add RelevanceAssessment and AttestationRelevanceContext records

Refs #231"
```

### Task 2: AttestationRelevanceFactor SPI + LedgerAttestationEnricher SPI

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/spi/AttestationRelevanceFactor.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/LedgerAttestationEnricher.java`

**Interfaces:**
- Consumes: `RelevanceAssessment`, `AttestationRelevanceContext` (from Task 1); `LedgerAttestation`, `LedgerEntry` (existing)
- Produces: `AttestationRelevanceFactor` interface (assess, assessBatch, name); `LedgerAttestationEnricher` interface (enrich, name)

- [ ] **Step 1: Create AttestationRelevanceFactor**

```java
package io.casehub.ledger.api.spi;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public interface AttestationRelevanceFactor {

    RelevanceAssessment assess(AttestationRelevanceContext ctx);

    default Map<UUID, RelevanceAssessment> assessBatch(List<AttestationRelevanceContext> ctxs) {
        Map<UUID, RelevanceAssessment> result = new LinkedHashMap<>();
        for (AttestationRelevanceContext ctx : ctxs) {
            result.put(ctx.attestationId(), assess(ctx));
        }
        return result;
    }

    String name();
}
```

- [ ] **Step 2: Create LedgerAttestationEnricher**

```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;

public interface LedgerAttestationEnricher {

    void enrich(LedgerAttestation attestation, LedgerEntry entry);

    String name();
}
```

- [ ] **Step 3: Compile api module**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/spi/AttestationRelevanceFactor.java api/src/main/java/io/casehub/ledger/api/spi/LedgerAttestationEnricher.java
git commit -m "feat(api): add AttestationRelevanceFactor and LedgerAttestationEnricher SPIs

Refs #231"
```

### Task 3: relevanceSnapshot field + schema + ORM

**Files:**
- Modify: `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java:47` — add field after `occurredAt`
- Modify: `ledger-jpa-common/src/main/resources/db/ledger/migration/V1000__ledger_initial_schema.sql:61` — add column before CONSTRAINT
- Modify: `runtime/src/main/resources/META-INF/orm.xml:110-112` — add mapping after `occurredAt`
- Modify: `ledger-spring-jpa/src/main/resources/META-INF/orm.xml` — add same mapping (find the attestation section)

**Interfaces:**
- Consumes: none
- Produces: `LedgerAttestation.relevanceSnapshot` (String, nullable) accessible by all modules

- [ ] **Step 1: Add relevanceSnapshot field to LedgerAttestation**

Add after `public Instant occurredAt;` (line 47):

```java
    public String relevanceSnapshot;
```

- [ ] **Step 2: Add column to V1000 migration**

In `ledger-jpa-common/src/main/resources/db/ledger/migration/V1000__ledger_initial_schema.sql`, add after `occurred_at TIMESTAMP NOT NULL,` (line 61) and before the CONSTRAINT line:

```sql
    relevance_snapshot  TEXT,
```

- [ ] **Step 3: Add ORM mapping to runtime orm.xml**

After the `occurredAt` mapping (after line 112, before `</attributes>`):

```xml
            <basic name="relevanceSnapshot">
                <column name="relevance_snapshot" column-definition="TEXT"/>
            </basic>
```

- [ ] **Step 4: Add ORM mapping to ledger-spring-jpa orm.xml**

Same mapping in `ledger-spring-jpa/src/main/resources/META-INF/orm.xml`, in the `LedgerAttestation` mapped-superclass section.

- [ ] **Step 5: Compile all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java ledger-jpa-common/src/main/resources/db/ledger/migration/V1000__ledger_initial_schema.sql runtime/src/main/resources/META-INF/orm.xml ledger-spring-jpa/src/main/resources/META-INF/orm.xml
git commit -m "feat(schema): add relevance_snapshot TEXT column to ledger_attestation

Write-time enricher JSON snapshot for attestation relevance factors.
Null = no enrichers ran (backward-compatible).

Refs #231"
```

### Task 4: AttestationWeights + RelevanceComposer

**Files:**
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/trust/AttestationWeights.java`
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/trust/RelevanceComposer.java`
- Test: `ledger-core/src/test/java/io/casehub/ledger/core/trust/RelevanceComposerTest.java`

**Interfaces:**
- Consumes: `AttestationRelevanceFactor`, `AttestationRelevanceContext`, `RelevanceAssessment` (from Tasks 1-2); `AttestorCredibilityPolicy.CredibilityAssessment` (existing)
- Produces: `AttestationWeights(Map<String, CredibilityAssessment>, Map<UUID, Double>)` with `NEUTRAL`; `RelevanceComposer.assess(List<AttestationRelevanceContext>) → Map<UUID, Double>`

- [ ] **Step 1: Write RelevanceComposer tests**

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.spi.AttestationRelevanceContext;
import io.casehub.ledger.api.spi.AttestationRelevanceFactor;
import io.casehub.ledger.api.spi.RelevanceAssessment;
import io.casehub.platform.api.identity.ActorType;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;

class RelevanceComposerTest {

    private static AttestationRelevanceContext ctx(UUID id) {
        return new AttestationRelevanceContext(
                id, "attestor", ActorType.AGENT, "reviewer",
                "actor", "tenant", "review", null, Instant.now(),
                Map.of("org-distance", 2));
    }

    @Test
    void noFactors_returnsNeutral() {
        var composer = new RelevanceComposer(List.of(), 0.5);
        var id = UUID.randomUUID();
        Map<UUID, Double> result = composer.assess(List.of(ctx(id)));
        assertEquals(1.0, result.get(id));
    }

    @Test
    void singleFactor_returnsFactorWeight() {
        AttestationRelevanceFactor factor = new AttestationRelevanceFactor() {
            @Override public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
                return new RelevanceAssessment(0.7, "test");
            }
            @Override public String name() { return "test"; }
        };
        var composer = new RelevanceComposer(List.of(factor), 0.5);
        var id = UUID.randomUUID();
        Map<UUID, Double> result = composer.assess(List.of(ctx(id)));
        assertEquals(0.7, result.get(id), 1e-9);
    }

    @Test
    void twoFactors_productsFloored() {
        AttestationRelevanceFactor f1 = new AttestationRelevanceFactor() {
            @Override public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
                return new RelevanceAssessment(0.6, "f1");
            }
            @Override public String name() { return "f1"; }
        };
        AttestationRelevanceFactor f2 = new AttestationRelevanceFactor() {
            @Override public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
                return new RelevanceAssessment(0.5, "f2");
            }
            @Override public String name() { return "f2"; }
        };
        var composer = new RelevanceComposer(List.of(f1, f2), 0.5);
        var id = UUID.randomUUID();
        Map<UUID, Double> result = composer.assess(List.of(ctx(id)));
        // 0.6 * 0.5 = 0.3 < floor 0.5 → clamped to 0.5
        assertEquals(0.5, result.get(id), 1e-9);
    }

    @Test
    void floorValidation_rejectsZero() {
        assertThrows(IllegalArgumentException.class,
                () -> new RelevanceComposer(List.of(), 0.0));
    }

    @Test
    void floorValidation_rejectsAboveOne() {
        assertThrows(IllegalArgumentException.class,
                () -> new RelevanceComposer(List.of(), 1.1));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=RelevanceComposerTest -DfailIfNoTests=false`
Expected: compilation error — `RelevanceComposer` / `AttestationWeights` do not exist

- [ ] **Step 3: Implement AttestationWeights**

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.spi.AttestorCredibilityPolicy;

import java.util.Map;
import java.util.UUID;

public record AttestationWeights(
        Map<String, AttestorCredibilityPolicy.CredibilityAssessment> credibilityByAttestorId,
        Map<UUID, Double> relevanceByAttestationId
) {
    public static final AttestationWeights NEUTRAL = new AttestationWeights(Map.of(), Map.of());
}
```

- [ ] **Step 4: Implement RelevanceComposer**

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.spi.AttestationRelevanceContext;
import io.casehub.ledger.api.spi.AttestationRelevanceFactor;
import io.casehub.ledger.api.spi.RelevanceAssessment;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

public class RelevanceComposer {

    private final List<AttestationRelevanceFactor> factors;
    private final double minWeight;

    public RelevanceComposer(List<AttestationRelevanceFactor> factors, double minWeight) {
        if (minWeight <= 0.0 || minWeight > 1.0) {
            throw new IllegalArgumentException(
                    "minWeight must be in (0.0, 1.0], got: " + minWeight);
        }
        this.factors = List.copyOf(factors);
        this.minWeight = minWeight;
    }

    public Map<UUID, Double> assess(List<AttestationRelevanceContext> contexts) {
        if (factors.isEmpty()) {
            Map<UUID, Double> neutral = new LinkedHashMap<>();
            for (AttestationRelevanceContext ctx : contexts) {
                neutral.put(ctx.attestationId(), 1.0);
            }
            return neutral;
        }

        Map<UUID, Double> result = new LinkedHashMap<>();
        for (AttestationRelevanceContext ctx : contexts) {
            double product = 1.0;
            for (AttestationRelevanceFactor factor : factors) {
                RelevanceAssessment assessment = factor.assess(ctx);
                product *= assessment.weight();
            }
            result.put(ctx.attestationId(), Math.max(minWeight, product));
        }
        return result;
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=RelevanceComposerTest`
Expected: all 5 tests PASS

- [ ] **Step 6: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/trust/AttestationWeights.java ledger-core/src/main/java/io/casehub/ledger/core/trust/RelevanceComposer.java ledger-core/src/test/java/io/casehub/ledger/core/trust/RelevanceComposerTest.java
git commit -m "feat(core): add AttestationWeights and RelevanceComposer

RelevanceComposer iterates AttestationRelevanceFactor beans, computes
product-with-floor, returns raw weights for per-pass normalisation.
Zero factors → NEUTRAL for all assessments.

Refs #231"
```

---

## Batch 2: Attestation Enricher Pipeline

Wire the write-time enricher pipeline into all three repository implementations. After this batch: attestation enrichers discovered on the classpath run before persist, `relevanceSnapshot` is cleared and re-populated by enrichers, and the pipeline round-trips through JPA and in-memory.

### Task 5: AttestationEnricherPipelineCore + CDI pipeline + repository integration

**Files:**
- Create: `ledger-core/src/main/java/io/casehub/ledger/core/enricher/AttestationEnricherPipelineCore.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerAttestationEnricherPipeline.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java:241-267`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java:202-233`
- Test: `ledger-core/src/test/java/io/casehub/ledger/core/enricher/AttestationEnricherPipelineCoreTest.java`
- Test: `runtime/src/test/java/io/casehub/ledger/service/LedgerAttestationEnricherPipelineIT.java`

**Interfaces:**
- Consumes: `LedgerAttestationEnricher` (Task 2), `LedgerAttestation.relevanceSnapshot` (Task 3)
- Produces: `AttestationEnricherPipelineCore.enrich(LedgerAttestation, LedgerEntry)` — iterates enrichers, non-fatal failures

- [ ] **Step 1: Write AttestationEnricherPipelineCore test**

```java
package io.casehub.ledger.core.enricher;

import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.LedgerAttestationEnricher;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class AttestationEnricherPipelineCoreTest {

    private static class TestEntry extends LedgerEntry {}

    @Test
    void noEnrichers_noOp() {
        var pipeline = new AttestationEnricherPipelineCore(List.of());
        var att = new LedgerAttestation();
        pipeline.enrich(att, new TestEntry());
        assertNull(att.relevanceSnapshot);
    }

    @Test
    void enricher_setsSnapshot() {
        LedgerAttestationEnricher enricher = new LedgerAttestationEnricher() {
            @Override public void enrich(LedgerAttestation a, LedgerEntry e) {
                a.relevanceSnapshot = "{\"org-distance\":2}";
            }
            @Override public String name() { return "test"; }
        };
        var pipeline = new AttestationEnricherPipelineCore(List.of(enricher));
        var att = new LedgerAttestation();
        pipeline.enrich(att, new TestEntry());
        assertEquals("{\"org-distance\":2}", att.relevanceSnapshot);
    }

    @Test
    void failingEnricher_nonFatal() {
        LedgerAttestationEnricher failing = new LedgerAttestationEnricher() {
            @Override public void enrich(LedgerAttestation a, LedgerEntry e) {
                throw new RuntimeException("boom");
            }
            @Override public String name() { return "failing"; }
        };
        var pipeline = new AttestationEnricherPipelineCore(List.of(failing));
        var att = new LedgerAttestation();
        assertDoesNotThrow(() -> pipeline.enrich(att, new TestEntry()));
        assertNull(att.relevanceSnapshot);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=AttestationEnricherPipelineCoreTest -DfailIfNoTests=false`
Expected: compilation error

- [ ] **Step 3: Implement AttestationEnricherPipelineCore**

```java
package io.casehub.ledger.core.enricher;

import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.LedgerAttestationEnricher;

import java.util.List;
import java.util.logging.Level;
import java.util.logging.Logger;

public class AttestationEnricherPipelineCore {

    private static final Logger log = Logger.getLogger(AttestationEnricherPipelineCore.class.getName());

    private final List<LedgerAttestationEnricher> enrichers;

    public AttestationEnricherPipelineCore(List<LedgerAttestationEnricher> enrichers) {
        this.enrichers = List.copyOf(enrichers);
    }

    public void enrich(LedgerAttestation attestation, LedgerEntry entry) {
        for (var enricher : enrichers) {
            try {
                enricher.enrich(attestation, entry);
            } catch (Exception ex) {
                log.log(Level.WARNING, "Attestation enricher {0} failed — snapshot key will be absent: {1}",
                        new Object[]{enricher.name(), ex.getMessage()});
            }
        }
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=AttestationEnricherPipelineCoreTest`
Expected: all 3 tests PASS

- [ ] **Step 5: Create CDI pipeline wrapper**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerAttestationEnricherPipeline.java`:

```java
package io.casehub.ledger.runtime.service;

import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.LedgerAttestationEnricher;
import io.casehub.ledger.core.enricher.AttestationEnricherPipelineCore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;

@ApplicationScoped
public class LedgerAttestationEnricherPipeline {

    private final AttestationEnricherPipelineCore core;

    @Inject
    LedgerAttestationEnricherPipeline(@Any Instance<LedgerAttestationEnricher> enrichers) {
        this.core = new AttestationEnricherPipelineCore(enrichers.stream().toList());
    }

    public void enrich(final LedgerAttestation attestation, final LedgerEntry entry) {
        core.enrich(attestation, entry);
    }
}
```

- [ ] **Step 6: Integrate into JpaLedgerEntryRepository.saveAttestation()**

In `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java`, add field injection for the pipeline and modify `saveAttestation()`.

Add field:
```java
@Inject
LedgerAttestationEnricherPipeline attestationEnricherPipeline;
```

Modify `saveAttestation()` — insert after the entry lookup (line 249) and before tokenisation (line 251):

```java
        attestation.relevanceSnapshot = null;
        attestationEnricherPipeline.enrich(attestation, entry);
```

- [ ] **Step 7: Integrate into InMemoryLedgerEntryRepository.saveAttestation()**

The in-memory repo currently tokenises at line 209 BEFORE the entry lookup at line 213. The enricher must run BEFORE tokenisation. Restructure to: entry lookup → clear snapshot → enrich → tokenise → store.

The in-memory repo needs an `AttestationEnricherPipelineCore` injected. Add constructor parameter and store as field. Insert before tokenisation:

```java
        attestation.relevanceSnapshot = null;
        if (entry != null) {
            attestationEnricherCore.enrich(attestation, entry);
        }
```

- [ ] **Step 8: Run all runtime tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,ledger-core -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS (existing tests still pass with enricher pipeline in place, no enrichers on classpath)

- [ ] **Step 9: Run persistence-memory tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,ledger-core -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl persistence-memory`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/enricher/AttestationEnricherPipelineCore.java runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerAttestationEnricherPipeline.java runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java ledger-core/src/test/java/io/casehub/ledger/core/enricher/AttestationEnricherPipelineCoreTest.java
git commit -m "feat: attestation enricher pipeline — write-time hooks for relevance snapshot

LedgerAttestationEnricher SPI in api, pipeline core in ledger-core,
CDI wrapper in runtime. Integrated into JPA and in-memory repositories.
Pipeline runs after entry lookup, before tokenisation and persist.
Caller-supplied relevanceSnapshot is cleared before enricher runs.

Refs #231"
```

---

## Batch 3: Trust Computation Integration

TrustScoreComputer gains self-attestation exclusion, per-pass mass-preserving normalisation, and the AttestationWeights signature. TrustScoreCalculator builds relevance context and wires through the composer. After this batch: trust computation uses the four-factor weight chain with normalisation.

### Task 6: TrustScoreComputer — AttestationWeights signature + self-attestation exclusion + normalisation

**Files:**
- Modify: `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputer.java`
- Test: `ledger-core/src/test/java/io/casehub/ledger/core/trust/TrustScoreComputerRelevanceTest.java`

**Interfaces:**
- Consumes: `AttestationWeights` (Task 4)
- Produces: `compute(List<LedgerEntry>, Map<UUID, List<LedgerAttestation>>, Instant, AttestationWeights) → ActorScore`; `computeDimensionScore(List<LedgerAttestation>, Instant, AttestationWeights) → OptionalDouble`

- [ ] **Step 1: Write failing tests for the new behaviour**

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class TrustScoreComputerRelevanceTest {

    private static final Instant NOW = Instant.now();
    private final TrustScoreComputer computer = new TrustScoreComputer(90);

    private static class TestEntry extends LedgerEntry {}

    private static TestEntry decision(String actorId) {
        TestEntry e = new TestEntry();
        e.id = UUID.randomUUID();
        e.actorId = actorId;
        return e;
    }

    private static LedgerAttestation attestation(UUID entryId, String attestorId, AttestationVerdict verdict) {
        LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = entryId;
        a.attestorId = attestorId;
        a.verdict = verdict;
        a.confidence = 1.0;
        a.occurredAt = NOW;
        return a;
    }

    @Test
    void neutralWeights_sameScoreAsNoWeights() {
        TestEntry d = decision("actor-1");
        LedgerAttestation a1 = attestation(d.id, "peer-1", AttestationVerdict.SOUND);
        LedgerAttestation a2 = attestation(d.id, "peer-2", AttestationVerdict.SOUND);
        LedgerAttestation a3 = attestation(d.id, "peer-3", AttestationVerdict.FLAGGED);

        var byEntry = Map.of(d.id, List.of(a1, a2, a3));
        var decisions = List.<LedgerEntry>of(d);

        var scoreNeutral = computer.compute(decisions, byEntry, NOW, AttestationWeights.NEUTRAL);
        var scoreWithUniform = computer.compute(decisions, byEntry, NOW,
                new AttestationWeights(Map.of(),
                        Map.of(a1.id, 0.7, a2.id, 0.7, a3.id, 0.7)));

        assertEquals(scoreNeutral.trustScore(), scoreWithUniform.trustScore(), 1e-9,
                "Uniform relevance must not change score (mass-preserving)");
    }

    @Test
    void selfAttestation_excluded() {
        TestEntry d = decision("actor-1");
        LedgerAttestation selfAtt = attestation(d.id, "actor-1", AttestationVerdict.SOUND);
        LedgerAttestation peerAtt = attestation(d.id, "peer-1", AttestationVerdict.FLAGGED);

        var byEntry = Map.of(d.id, List.of(selfAtt, peerAtt));
        var decisions = List.<LedgerEntry>of(d);
        var score = computer.compute(decisions, byEntry, NOW, AttestationWeights.NEUTRAL);

        // Only peer-1's FLAGGED counts: alpha=1, beta=1+1=2, score=1/3
        assertEquals(1.0 / 3.0, score.trustScore(), 1e-9);
        assertEquals(0, score.attestationPositive());
        assertEquals(1, score.attestationNegative());
    }

    @Test
    void zeroCredibility_noNaN() {
        TestEntry d = decision("actor-1");
        LedgerAttestation a = attestation(d.id, "peer-1", AttestationVerdict.SOUND);

        var credMap = Map.of("peer-1",
                new io.casehub.ledger.api.spi.AttestorCredibilityPolicy.CredibilityAssessment(0.0, null, Set.of()));
        var weights = new AttestationWeights(credMap, Map.of(a.id, 0.7));

        var score = computer.compute(List.<LedgerEntry>of(d),
                Map.of(d.id, List.of(a)), NOW, weights);
        assertEquals(0.5, score.trustScore(), 1e-9, "Zero mass → prior 0.5, no NaN");
        assertFalse(Double.isNaN(score.trustScore()));
    }

    @Test
    void credibilityRetention_notAffectedByRelevance() {
        TestEntry d = decision("actor-1");
        LedgerAttestation a1 = attestation(d.id, "peer-1", AttestationVerdict.SOUND);
        LedgerAttestation a2 = attestation(d.id, "peer-2", AttestationVerdict.SOUND);

        var scoreNoRelevance = computer.compute(List.<LedgerEntry>of(d),
                Map.of(d.id, List.of(a1, a2)), NOW, AttestationWeights.NEUTRAL);
        var scoreWithRelevance = computer.compute(List.<LedgerEntry>of(d),
                Map.of(d.id, List.of(a1, a2)), NOW,
                new AttestationWeights(Map.of(), Map.of(a1.id, 0.5, a2.id, 1.0)));

        assertEquals(scoreNoRelevance.credibilityRetention(),
                scoreWithRelevance.credibilityRetention(), 1e-9,
                "credibilityRetention must not be affected by relevance");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=TrustScoreComputerRelevanceTest -DfailIfNoTests=false`
Expected: compilation error (AttestationWeights not yet accepted by compute())

- [ ] **Step 3: Replace compute() with AttestationWeights signature + normalisation + self-attestation exclusion**

Replace the existing `compute()` overloads with a single canonical method. The key changes:
1. Remove the old `compute(decisions, byEntry, now)` and `compute(decisions, byEntry, now, credibilityMap)` overloads
2. Add `compute(decisions, byEntry, now, AttestationWeights weights)` with two-pass normalisation
3. Skip attestations where `attestation.attestorId` equals the entry's `actorId` (self-attestation exclusion)
4. Use separate accumulator for `credibilityRetention`

The two-pass normalisation inside `compute()`:
- Pass 1: iterate all attestations, compute `rawWeight × credWeight` and `rawWeight × credWeight × relevWeight`, sum both.
- If `sumRelevance == 0`: normFactor = 1.0 (guard).
- Else: normFactor = sumBase / sumRelevance.
- Pass 2: iterate again, compute `normalisedRelevance = relevWeight × normFactor` (do NOT clamp), use in effectiveWeight.

- [ ] **Step 4: Replace computeDimensionScore() with AttestationWeights signature**

Replace the existing overloads with a single `computeDimensionScore(attestations, now, AttestationWeights weights)`. Apply raw relevance weights directly (no normalisation for dimension scores — relative weights are what matters in a weighted average).

- [ ] **Step 5: Run the new tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=TrustScoreComputerRelevanceTest`
Expected: all 4 tests PASS

- [ ] **Step 6: Fix compilation in existing tests and callers**

Update all callers of the old overloads to pass `AttestationWeights.NEUTRAL`:
- `TrustScoreCalculator.java` — will be updated in Task 7
- `TrustScoreComputerTest.java` — update test calls
- `TrustScoreSourceContractTest.java` — updated in Batch 4

For now, temporarily keep a delegation overload `compute(decisions, byEntry, now, credibilityMap)` that wraps into `AttestationWeights` to avoid breaking everything at once.

- [ ] **Step 7: Run all ledger-core tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputer.java ledger-core/src/test/java/io/casehub/ledger/core/trust/TrustScoreComputerRelevanceTest.java ledger-core/src/test/java/io/casehub/ledger/core/trust/TrustScoreComputerTest.java
git commit -m "feat(core): TrustScoreComputer gains AttestationWeights, self-attestation exclusion, per-pass normalisation

Weight chain: decay × confidence × credibility × normalised_relevance.
Self-attestation (attestorId == entry.actorId) excluded from Beta passes.
Per-pass normalisation preserves evidence mass — uniform relevance has
zero effect on scores. credibilityRetention uses separate accumulator.

Refs #231"
```

### Task 7: TrustScoreCalculator — build relevance context + wire RelevanceComposer

**Files:**
- Modify: `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreCalculator.java`
- Test: `ledger-core/src/test/java/io/casehub/ledger/core/trust/TrustScoreCalculatorRelevanceTest.java`

**Interfaces:**
- Consumes: `RelevanceComposer` (Task 4), `AttestationRelevanceContext` (Task 1), `AttestationWeights` (Task 4)
- Produces: `computeAll(decisions, attestationsByEntry, now) → ComputedScores` (unchanged signature — context built internally from entry data)

- [ ] **Step 1: Write failing test**

```java
package io.casehub.ledger.core.trust;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.AttestationRelevanceContext;
import io.casehub.ledger.api.spi.AttestationRelevanceFactor;
import io.casehub.ledger.api.spi.RelevanceAssessment;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.junit.jupiter.api.Assertions.*;

class TrustScoreCalculatorRelevanceTest {

    private static final Instant NOW = Instant.now();

    private static class TestEntry extends LedgerEntry {}

    private static TestEntry decision(String actorId, String tenancyId) {
        TestEntry e = new TestEntry();
        e.id = UUID.randomUUID();
        e.actorId = actorId;
        e.tenancyId = tenancyId;
        return e;
    }

    private static LedgerAttestation attestation(UUID entryId, UUID subjectId,
                                                   String attestorId, String capTag) {
        LedgerAttestation a = new LedgerAttestation();
        a.id = UUID.randomUUID();
        a.ledgerEntryId = entryId;
        a.subjectId = subjectId;
        a.attestorId = attestorId;
        a.verdict = AttestationVerdict.SOUND;
        a.confidence = 1.0;
        a.capabilityTag = capTag;
        a.occurredAt = NOW;
        return a;
    }

    @Test
    void composerReceivesContext_withEntryActorAndTenancy() {
        TestEntry d = decision("scored-actor", "tenant-A");
        d.subjectId = UUID.randomUUID();
        LedgerAttestation att = attestation(d.id, d.subjectId, "attestor-1", "review");

        List<AttestationRelevanceContext> captured = new ArrayList<>();
        AttestationRelevanceFactor capturingFactor = new AttestationRelevanceFactor() {
            @Override public RelevanceAssessment assess(AttestationRelevanceContext ctx) {
                captured.add(ctx);
                return RelevanceAssessment.NEUTRAL;
            }
            @Override public String name() { return "capturing"; }
        };
        var composer = new RelevanceComposer(List.of(capturingFactor), 0.5);
        var calculator = new TrustScoreCalculator(
                (age, v) -> 1.0, new AllAttestationsGlobalStrategy(),
                new NoOpAttestorCredibilityPolicy(), composer);

        calculator.computeAll(List.of(d), Map.of(d.id, List.of(att)), NOW);

        assertFalse(captured.isEmpty());
        AttestationRelevanceContext ctx = captured.get(0);
        assertEquals("scored-actor", ctx.scoredActorId());
        assertEquals("tenant-A", ctx.tenancyId());
        assertEquals("attestor-1", ctx.attestorId());
        assertEquals(att.id, ctx.attestationId());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Expected: compilation error — TrustScoreCalculator constructor doesn't accept RelevanceComposer yet

- [ ] **Step 3: Add RelevanceComposer to TrustScoreCalculator**

Add `RelevanceComposer` as a fourth constructor parameter. In `computeAll()`, after the credibility batch resolution:

1. Build a `Map<UUID, LedgerEntry>` for entry-by-attestation lookup (from existing `decisions` list)
2. Build `List<AttestationRelevanceContext>` from all attestations
3. Call `relevanceComposer.assess(contexts)` → `Map<UUID, Double>`
4. Build `AttestationWeights(credibilityMap, relevanceMap)`
5. Pass `AttestationWeights` to all `computer.compute()` and `computer.computeDimensionScore()` calls

The context is built from entry data: `entry.actorId` as `scoredActorId`, `entry.tenancyId` as `tenancyId`. The snapshot comes from `attestation.relevanceSnapshot` parsed as JSON `Map<String, Object>`.

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger-core -Dtest=TrustScoreCalculatorRelevanceTest`
Expected: PASS

- [ ] **Step 5: Fix all TrustScoreCalculator construction sites**

Update every place that constructs `TrustScoreCalculator` to pass a `RelevanceComposer`:
- `LedgerCoreProducer.trustScoreCalculator()` at `runtime/.../LedgerCoreProducer.java:41` — inject `RelevanceComposer`
- `PerActorTrustComputer` at `runtime/.../PerActorTrustComputer.java:39` — inject `RelevanceComposer`
- `TrustScoreSourceContractTest` at `runtime/src/test/.../TrustScoreSourceContractTest.java:77` — pass `new RelevanceComposer(List.of(), 0.5)`
- All test classes in ledger-core that construct `TrustScoreCalculator` — pass `new RelevanceComposer(List.of(), 0.5)`

- [ ] **Step 6: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api,ledger-core -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreCalculator.java runtime/src/main/java/io/casehub/ledger/runtime/LedgerCoreProducer.java runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java runtime/src/main/java/io/casehub/ledger/runtime/service/ComputedTrustScoreSource.java ledger-core/src/test/java/io/casehub/ledger/core/trust/TrustScoreCalculatorRelevanceTest.java runtime/src/test/java/io/casehub/ledger/service/TrustScoreSourceContractTest.java
git commit -m "feat(core): TrustScoreCalculator builds relevance context from entry data

Calculator resolves relevance via RelevanceComposer alongside credibility.
Context built per attestation using entry.actorId (scored actor) and
entry.tenancyId. AttestationWeights flows to TrustScoreComputer.

Refs #231"
```

---

## Batch 4: Config + Spring Wiring + Full Test Suite

LedgerConfig gains the min-weight property. Spring wiring updated. All module tests pass.

### Task 8: LedgerConfig + Spring wiring + full test pass

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java:277` — add `RelevanceConfig` to `TrustScoreConfig`
- Modify: `ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerManualConfig.java` — add enricher pipeline + RelevanceComposer beans
- Modify: `ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerTrustConfig.java` — thread RelevanceComposer to TrustScoreCalculator bean
- Modify: `ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerConfigurationProperties.java` — add min-weight
- Modify: `ledger-core/src/main/java/io/casehub/ledger/core/config/LedgerProperties.java` — add relevance config to trust-score

**Interfaces:**
- Consumes: `RelevanceComposer` (Task 4), `AttestationEnricherPipelineCore` (Task 5)
- Produces: CDI + Spring beans wired

- [ ] **Step 1: Add RelevanceConfig to LedgerConfig**

In `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java`, inside `TrustScoreConfig` interface (after `eigentrust()` method):

```java
        RelevanceConfig relevance();

        interface RelevanceConfig {
            @WithDefault("0.5")
            double minWeight();
        }
```

- [ ] **Step 2: Add relevance config to LedgerProperties**

Add the equivalent record to `ledger-core/src/main/java/io/casehub/ledger/core/config/LedgerProperties.java`.

- [ ] **Step 3: Add RelevanceComposer CDI producer**

In `runtime/src/main/java/io/casehub/ledger/runtime/LedgerCoreProducer.java`, add:

```java
    @Produces
    @DefaultBean
    public RelevanceComposer relevanceComposer(
            @Any Instance<AttestationRelevanceFactor> factors,
            LedgerConfig config) {
        return new RelevanceComposer(
                factors.stream().toList(),
                config.trustScore().relevance().minWeight());
    }
```

- [ ] **Step 4: Wire Spring LedgerManualConfig**

In `LedgerManualConfig`, add:

```java
    @Bean
    AttestationEnricherPipelineCore attestationEnricherPipelineCore(
            ObjectProvider<LedgerAttestationEnricher> enrichers) {
        return new AttestationEnricherPipelineCore(enrichers.orderedStream().toList());
    }

    @Bean
    @ConditionalOnMissingBean(RelevanceComposer.class)
    RelevanceComposer relevanceComposer(
            ObjectProvider<AttestationRelevanceFactor> factors,
            LedgerProperties props) {
        return new RelevanceComposer(
                factors.orderedStream().toList(),
                props.trustScore().relevance().minWeight());
    }

    @Bean
    @ConditionalOnMissingBean(TrustScoreCalculator.class)
    TrustScoreCalculator trustScoreCalculator(DecayFunction decay,
                                               GlobalScoreStrategy global,
                                               io.casehub.ledger.api.spi.AttestorCredibilityPolicy credibility,
                                               RelevanceComposer composer) {
        return new TrustScoreCalculator(decay, global, credibility, composer);
    }
```

- [ ] **Step 5: Wire Spring LedgerTrustConfig**

In `LedgerTrustConfig`, update `perActorTrustComputerCore` and `computedTrustSource` to accept `TrustScoreCalculator` from the bean factory (no changes needed if bean is injected).

- [ ] **Step 6: Update LedgerConfigurationProperties for Spring**

Add relevance config nested class with `minWeight` defaulting to 0.5.

- [ ] **Step 7: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: BUILD SUCCESS — all modules compile and all tests pass

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java runtime/src/main/java/io/casehub/ledger/runtime/LedgerCoreProducer.java ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerManualConfig.java ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerTrustConfig.java ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerConfigurationProperties.java ledger-core/src/main/java/io/casehub/ledger/core/config/LedgerProperties.java
git commit -m "feat: wire relevance pipeline — config, CDI producers, Spring beans

casehub.ledger.trust-score.relevance.min-weight (default 0.5) governs
composition floor. RelevanceComposer and AttestationEnricherPipelineCore
wired in both Quarkus and Spring frameworks.

Refs #231"
```

---

## References

- [2026-11-18-attestation-relevance-design.md](../specs/issue-231-org-scoped-attestation-weighting/2026-11-18-attestation-relevance-design.md) — design spec this plan implements
- `api/src/main/java/io/casehub/ledger/api/model/LedgerAttestation.java` — attestation model (add relevanceSnapshot field)
- `api/src/main/java/io/casehub/ledger/api/spi/AttestorCredibilityPolicy.java` — SPI pattern precedent
- `ledger-core/src/main/java/io/casehub/ledger/core/enricher/LedgerEntryEnricher.java` — enricher SPI pattern
- `ledger-core/src/main/java/io/casehub/ledger/core/enricher/EnricherPipelineCore.java` — pipeline core pattern
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerEnricherPipeline.java` — CDI pipeline pattern
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreComputer.java` — weight chain
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/TrustScoreCalculator.java` — batch resolution, four-pass algorithm
- `ledger-core/src/main/java/io/casehub/ledger/core/trust/PerActorTrustComputerCore.java:42` — computeAll call site
- `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaLedgerEntryRepository.java:241` — JPA saveAttestation
- `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryLedgerEntryRepository.java:202` — in-memory saveAttestation
- `runtime/src/main/java/io/casehub/ledger/runtime/LedgerCoreProducer.java:41` — CDI TrustScoreCalculator producer
- `runtime/src/main/java/io/casehub/ledger/runtime/config/LedgerConfig.java:277` — TrustScoreConfig
- `ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerManualConfig.java` — Spring config
- `ledger-spring/src/main/java/io/casehub/ledger/spring/LedgerTrustConfig.java` — Spring trust config
- `runtime/src/main/resources/META-INF/orm.xml:72` — ORM attestation mapping
- `ledger-spring-jpa/src/main/resources/META-INF/orm.xml:72` — Spring ORM attestation mapping
- `ledger-jpa-common/src/main/resources/db/ledger/migration/V1000__ledger_initial_schema.sql:48` — attestation table
- `runtime/src/test/java/io/casehub/ledger/service/TrustScoreSourceContractTest.java:77` — contract test construction
- casehubio/ledger#231 — focal issue
- casehubio/casehub-ops#217 — parent epic
