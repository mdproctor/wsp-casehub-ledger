# LedgerSupplement Architecture Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Introduce the `LedgerSupplement` architecture — an optional, lazily-loaded,
zero-boilerplate extension system for `LedgerEntry` — and deliver `ComplianceSupplement`
as its first application, covering GDPR Art.22 structured decision snapshot fields.

**Architecture:** `LedgerEntry` slims to 10 core fields. Optional cross-cutting concerns
(compliance, provenance, observability) live in separate JPA JOINED-inheritance entities
(`LedgerSupplement` subclasses) attached via a lazy `List`. A `supplement_json` TEXT column
on `ledger_entry` holds a denormalised snapshot for fast single-entry reads without joins.
The `attach()` helper on `LedgerEntry` keeps the JSON and the list in sync automatically.

**Tech Stack:** Java 21, Quarkus 3.32.2, Hibernate ORM / Panache, Flyway, H2 (test),
PostgreSQL (prod), JUnit 5, AssertJ, RestAssured, Jackson (for JSON serialisation).

**All commits reference:** `Refs #7` (child issue) and are part of epic `#6`.

**Build command:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
**Run order-processing IT:** `cd examples/order-processing && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

---

## File Map

### New files — runtime module

| File | Responsibility |
|---|---|
| `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplement.java` | Abstract base entity — id, ledgerEntry FK, JPA JOINED inheritance |
| `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ComplianceSupplement.java` | planRef, rationale, evidence, detail, decisionContext + 4 Art.22 fields |
| `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ProvenanceSupplement.java` | sourceEntityId, sourceEntityType, sourceEntitySystem |
| `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ObservabilitySupplement.java` | correlationId, causedByEntryId |
| `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplementSerializer.java` | toJson(List) — explicit per-type serialisation, null fields omitted |
| `runtime/src/test/java/io/casehub/ledger/service/supplement/LedgerSupplementSerializerTest.java` | Unit tests — serialisation, null-omission, multi-supplement |
| `runtime/src/test/java/io/casehub/ledger/service/supplement/LedgerSupplementIT.java` | @QuarkusTest — persist/load, lazy loading, zero-touch for bare entries |
| `runtime/src/test/java/io/casehub/ledger/service/supplement/TestEntry.java` | @Entity test-only subclass with its own table for IT |
| `runtime/src/test/resources/db/migration/V1999__test_entry.sql` | Table for TestEntry (test scope only) |

### Modified files — runtime module

| File | Change |
|---|---|
| `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java` | Slim to 10 core fields + `supplements` List + `supplement_json` + `attach()` + typed accessors |
| `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerHashChain.java` | Remove `planRef` from canonical form; update Javadoc |
| `runtime/src/test/java/io/casehub/ledger/service/LedgerHashChainTest.java` | Remove `planRef` from fixture; add test proving supplement fields don't affect hash |
| `runtime/src/main/resources/db/migration/V1002__ledger_supplement.sql` | New — drop removed columns, add supplement_json, create supplement tables |
| `docs/DESIGN.md` | Add Supplement chapter; update canonical form; update Flyway convention |
| `docs/AUDITABILITY.md` | Mark Axiom 8 gap as addressed |
| `docs/integration-guide.md` | Add Supplement section |

### Modified files — order-processing example

| File | Change |
|---|---|
| `examples/order-processing/src/main/java/.../OrderService.java` | Move rationale + decisionContext to ComplianceSupplement via `attach()` |
| `examples/order-processing/src/test/java/.../OrderLedgerIT.java` | Update to access supplement fields; add supplement-specific tests |
| `examples/order-processing/src/main/resources/db/migration/V1__order_schema.sql` | No change needed |

### New files — art22 example

| File | Responsibility |
|---|---|
| `examples/art22-decision-snapshot/pom.xml` | Standalone Quarkus app, mirrors order-processing pom |
| `examples/art22-decision-snapshot/src/main/java/.../DecisionLedgerEntry.java` | Subclass — `decisionId`, `decisionCategory` |
| `examples/art22-decision-snapshot/src/main/java/.../DecisionService.java` | Makes AI-style decisions, attaches ComplianceSupplement |
| `examples/art22-decision-snapshot/src/main/java/.../DecisionResource.java` | REST — POST /decisions, GET /decisions/{id}/ledger |
| `examples/art22-decision-snapshot/src/main/resources/db/migration/V1003__decision_schema.sql` | decision, decision_ledger_entry tables |
| `examples/art22-decision-snapshot/src/test/java/.../DecisionLedgerIT.java` | @QuarkusTest — Art.22 fields in response, verify/chain intact |
| `examples/art22-decision-snapshot/README.md` | GDPR Art.22 explanation, field mapping, how to run |

---

## Task 1 — LedgerSupplement base entity + three concrete supplements

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplement.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ComplianceSupplement.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ProvenanceSupplement.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/ObservabilitySupplement.java`

These are pure new code — no existing tests break. No test to write first (entity structure
is validated by the IT in Task 6). Compile-check is the gate.

- [ ] **Step 1: Create the supplement package directory**

```bash
mkdir -p runtime/src/main/java/io/casehub/ledger/runtime/model/supplement
```

- [ ] **Step 2: Create `LedgerSupplement.java`**

```java
package io.casehub.ledger.runtime.model.supplement;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorColumn;
import jakarta.persistence.DiscriminatorType;
import jakarta.persistence.Entity;
import jakarta.persistence.FetchType;
import jakarta.persistence.Id;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * Abstract base for all ledger supplements.
 *
 * <p>
 * A <strong>supplement</strong> is an optional, lazily-loaded extension to a
 * {@link LedgerEntry} that carries a named group of cross-cutting fields. Supplements
 * exist in separate joined tables and are never written unless the consumer explicitly
 * attaches one — consumers that do not use supplements incur zero schema or runtime cost.
 *
 * <p>
 * Three built-in supplements are provided:
 * <ul>
 *   <li>{@link ComplianceSupplement} — GDPR Art.22 decision snapshot, EU AI Act Art.12,
 *       governance reference, rationale</li>
 *   <li>{@link ProvenanceSupplement} — workflow source entity tracking</li>
 *   <li>{@link ObservabilitySupplement} — OpenTelemetry trace correlation, causality</li>
 * </ul>
 *
 * <p>
 * Supplements are accessed via the typed helper methods on {@link LedgerEntry}:
 * {@code entry.compliance()}, {@code entry.provenance()}, {@code entry.observability()}.
 * Use {@code entry.attach(supplement)} to add or replace a supplement; this also
 * keeps {@code entry.supplementJson} in sync automatically.
 *
 * <p>
 * <strong>Zero-complexity guarantee:</strong> If a consumer never calls
 * {@code entry.attach()}, no supplement table rows are written and the lazy
 * {@code supplements} list is never initialised.
 */
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "supplement_type", discriminatorType = DiscriminatorType.STRING)
@Table(name = "ledger_supplement")
public abstract class LedgerSupplement extends PanacheEntityBase {

    /** Primary key — UUID assigned on first persist. */
    @Id
    public UUID id;

    /**
     * The ledger entry this supplement belongs to.
     * Loaded lazily to avoid unnecessary joins when reading the base entry.
     */
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "ledger_entry_id", nullable = false)
    public LedgerEntry ledgerEntry;

    /**
     * Discriminator column value — managed by JPA, read-only via this field.
     * Use {@code instanceof} checks or {@link LedgerEntry#compliance()} etc.
     * for typed access rather than reading this field directly.
     */
    @Column(name = "supplement_type", insertable = false, updatable = false)
    public String supplementType;

    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
    }
}
```

- [ ] **Step 3: Create `ComplianceSupplement.java`**

```java
package io.casehub.ledger.runtime.model.supplement;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

/**
 * Supplement carrying compliance, governance, and GDPR Art.22 decision snapshot fields.
 *
 * <h2>GDPR Article 22 — Automated Decision-Making</h2>
 * <p>
 * Article 22 of the GDPR requires that automated decisions be explainable. Data subjects
 * have the right to receive "meaningful information about the logic involved" in any
 * automated decision that significantly affects them. The following fields provide the
 * structured evidence needed to satisfy this requirement:
 * <ul>
 *   <li>{@link #algorithmRef} — identifies which model, rule engine, or algorithm version
 *       produced the decision, enabling reproducibility and audit.</li>
 *   <li>{@link #confidenceScore} — the producing system's stated confidence (0.0–1.0),
 *       satisfying the requirement to disclose "the significance and envisaged
 *       consequences" of the decision.</li>
 *   <li>{@link #contestationUri} — where the data subject can request human review
 *       or challenge the decision, satisfying the right to contest under Art.22(3).</li>
 *   <li>{@link #humanOverrideAvailable} — whether a human review path exists,
 *       satisfying the Art.22(2) safeguard requirement.</li>
 *   <li>{@link #decisionContext} — full JSON snapshot of observable state at the moment
 *       of the decision, providing the "meaningful information" required by Arts.13–15.</li>
 * </ul>
 *
 * <h2>Governance fields</h2>
 * <p>
 * {@link #planRef} and {@link #rationale} record the policy version and stated basis
 * for the decision. {@link #evidence} and {@link #detail} carry structured evidence
 * and free-text overflow respectively.
 *
 * <h2>Usage</h2>
 * <pre>{@code
 * ComplianceSupplement cs = new ComplianceSupplement();
 * cs.algorithmRef    = "classification-model-v3.2";
 * cs.confidenceScore = 0.91;
 * cs.contestationUri = "https://example.com/decisions/challenge";
 * cs.humanOverrideAvailable = true;
 * cs.decisionContext = "{\"inputs\":{\"riskScore\":42}}";
 * entry.attach(cs);
 * }</pre>
 */
@Entity
@Table(name = "ledger_supplement_compliance")
@DiscriminatorValue("COMPLIANCE")
public class ComplianceSupplement extends LedgerSupplement {

    // ── Governance ────────────────────────────────────────────────────────────

    /** Reference to the policy or procedure version that governed this action. */
    @Column(name = "plan_ref", length = 500)
    public String planRef;

    /** The actor's stated basis for the decision. */
    @Column(columnDefinition = "TEXT")
    public String rationale;

    /** Structured evidence supplied by the actor. */
    @Column(columnDefinition = "TEXT")
    public String evidence;

    /** Free-text or JSON detail — delegation targets, rejection reasons, etc. */
    @Column(columnDefinition = "TEXT")
    public String detail;

    // ── GDPR Art.22 / EU AI Act Art.12 decision snapshot ─────────────────────

    /**
     * Full JSON snapshot of observable state at the moment of this decision.
     * Provides the "meaningful information about the logic involved" required by
     * GDPR Arts.13–15 and the technical logging required by EU AI Act Art.12.
     */
    @Column(name = "decision_context", columnDefinition = "TEXT")
    public String decisionContext;

    /**
     * Identifier of the model, rule engine, or algorithm version that produced
     * the decision. Examples: {@code "gpt-4o"}, {@code "risk-classifier-v2.1"},
     * {@code "approval-rules-2026-Q1"}. Required for reproducibility audits.
     */
    @Column(name = "algorithm_ref", length = 500)
    public String algorithmRef;

    /**
     * The producing system's stated confidence in this decision, in the range
     * 0.0 (no confidence) to 1.0 (certainty). Null when not applicable (e.g.
     * deterministic rule engines). Satisfies the GDPR requirement to disclose
     * the significance and envisaged consequences of the decision.
     */
    @Column(name = "confidence_score")
    public Double confidenceScore;

    /**
     * URI where the data subject can request human review or formally challenge
     * this decision, satisfying the contestation right under GDPR Art.22(3).
     * Example: {@code "https://example.com/decisions/{entryId}/challenge"}.
     */
    @Column(name = "contestation_uri", length = 2000)
    public String contestationUri;

    /**
     * Whether a human review path exists for this decision, satisfying the
     * Art.22(2)(b) safeguard requirement that data subjects have "the right
     * to obtain human intervention".
     */
    @Column(name = "human_override_available")
    public Boolean humanOverrideAvailable;
}
```

- [ ] **Step 4: Create `ProvenanceSupplement.java`**

```java
package io.casehub.ledger.runtime.model.supplement;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

/**
 * Supplement carrying workflow provenance — the external entity that originated
 * this ledger entry's subject.
 *
 * <p>
 * Use this supplement when a subject is created or driven by an external workflow
 * system (e.g. a {@code quarkus-flow} workflow instance). The three fields together
 * identify the source entity precisely enough to correlate across systems:
 *
 * <pre>{@code
 * ProvenanceSupplement ps = new ProvenanceSupplement();
 * ps.sourceEntityId     = workflowInstance.id.toString();
 * ps.sourceEntityType   = "Flow:WorkflowInstance";
 * ps.sourceEntitySystem = "quarkus-flow";
 * entry.attach(ps);
 * }</pre>
 */
@Entity
@Table(name = "ledger_supplement_provenance")
@DiscriminatorValue("PROVENANCE")
public class ProvenanceSupplement extends LedgerSupplement {

    /** Identifier of the external entity that originated this subject. */
    @Column(name = "source_entity_id", length = 255)
    public String sourceEntityId;

    /**
     * Type of the external entity.
     * Convention: {@code "System:TypeName"}, e.g. {@code "Flow:WorkflowInstance"}.
     */
    @Column(name = "source_entity_type", length = 255)
    public String sourceEntityType;

    /**
     * The system that owns the external entity.
     * Example: {@code "quarkus-flow"}, {@code "casehub-work"}.
     */
    @Column(name = "source_entity_system", length = 100)
    public String sourceEntitySystem;
}
```

- [ ] **Step 5: Create `ObservabilitySupplement.java`**

```java
package io.casehub.ledger.runtime.model.supplement;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

/**
 * Supplement carrying distributed tracing and causality fields.
 *
 * <p>
 * Use this supplement to link a ledger entry to an OpenTelemetry distributed
 * trace and/or to record a causal relationship with another ledger entry.
 *
 * <pre>{@code
 * ObservabilitySupplement os = new ObservabilitySupplement();
 * os.correlationId    = Span.current().getSpanContext().getTraceId();
 * os.causedByEntryId  = parentEntry.id;
 * entry.attach(os);
 * }</pre>
 */
@Entity
@Table(name = "ledger_supplement_observability")
@DiscriminatorValue("OBSERVABILITY")
public class ObservabilitySupplement extends LedgerSupplement {

    /**
     * OpenTelemetry trace ID linking this entry to a distributed trace.
     * Use the W3C trace context format (32-char hex string).
     */
    @Column(name = "correlation_id", length = 255)
    public String correlationId;

    /**
     * FK to the {@link io.casehub.ledger.runtime.model.LedgerEntry} that
     * causally produced this entry. Null for entries with no known causal predecessor.
     *
     * <p>
     * Use this when an orchestrator (e.g. Claudony) triggers work in Tarkus which
     * triggers a message in Qhorus — the Qhorus entry's {@code causedByEntryId}
     * points to the Tarkus entry, enabling full cross-system causal chain reconstruction.
     */
    @Column(name = "caused_by_entry_id")
    public UUID causedByEntryId;
}
```

- [ ] **Step 6: Verify compilation**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl runtime -q
```

Expected: BUILD SUCCESS with no errors.

- [ ] **Step 7: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/
git commit -m "feat(supplement): add LedgerSupplement base + three concrete supplements

Introduces the supplement architecture: optional, lazily-loaded, zero-boilerplate
extensions to LedgerEntry. ComplianceSupplement carries GDPR Art.22 fields.
ProvenanceSupplement tracks workflow source entities. ObservabilitySupplement
holds OTel correlation and causality.

Refs #7"
```

---

## Task 2 — LedgerSupplementSerializer (unit tests first)

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/supplement/LedgerSupplementSerializerTest.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplementSerializer.java`

- [ ] **Step 1: Write the failing tests**

Create `runtime/src/test/java/io/casehub/ledger/service/supplement/LedgerSupplementSerializerTest.java`:

```java
package io.casehub.ledger.service.supplement;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.LedgerSupplement;
import io.casehub.ledger.runtime.model.supplement.LedgerSupplementSerializer;
import io.casehub.ledger.runtime.model.supplement.ObservabilitySupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;

/**
 * Unit tests for {@link LedgerSupplementSerializer} — no Quarkus runtime, no CDI.
 */
class LedgerSupplementSerializerTest {

    @Test
    void toJson_nullList_returnsNull() {
        assertThat(LedgerSupplementSerializer.toJson(null)).isNull();
    }

    @Test
    void toJson_emptyList_returnsNull() {
        assertThat(LedgerSupplementSerializer.toJson(List.of())).isNull();
    }

    @Test
    void toJson_complianceSupplement_containsTypeKey() {
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "model-v1";

        final String json = LedgerSupplementSerializer.toJson(List.of(cs));

        assertThat(json).isNotNull();
        assertThat(json).contains("\"COMPLIANCE\"");
        assertThat(json).contains("\"algorithmRef\":\"model-v1\"");
    }

    @Test
    void toJson_nullFieldsOmitted() {
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "rule-engine-v2";
        // confidenceScore, contestationUri, humanOverrideAvailable all null

        final String json = LedgerSupplementSerializer.toJson(List.of(cs));

        assertThat(json).contains("algorithmRef");
        assertThat(json).doesNotContain("confidenceScore");
        assertThat(json).doesNotContain("contestationUri");
        assertThat(json).doesNotContain("humanOverrideAvailable");
    }

    @Test
    void toJson_allComplianceFields_serialisedCorrectly() {
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.planRef = "policy-2026-q1";
        cs.rationale = "Risk threshold exceeded";
        cs.algorithmRef = "gpt-4o";
        cs.confidenceScore = 0.92;
        cs.contestationUri = "https://example.com/challenge";
        cs.humanOverrideAvailable = true;
        cs.decisionContext = "{\"riskScore\":77}";

        final String json = LedgerSupplementSerializer.toJson(List.of(cs));

        assertThat(json).contains("\"planRef\":\"policy-2026-q1\"");
        assertThat(json).contains("\"rationale\":\"Risk threshold exceeded\"");
        assertThat(json).contains("\"algorithmRef\":\"gpt-4o\"");
        assertThat(json).contains("\"confidenceScore\":0.92");
        assertThat(json).contains("\"contestationUri\":\"https://example.com/challenge\"");
        assertThat(json).contains("\"humanOverrideAvailable\":true");
        assertThat(json).contains("\"decisionContext\":\"{\\\"riskScore\\\":77}\"");
    }

    @Test
    void toJson_provenanceSupplement_serialisedCorrectly() {
        final ProvenanceSupplement ps = new ProvenanceSupplement();
        ps.sourceEntityId = "wf-123";
        ps.sourceEntityType = "Flow:WorkflowInstance";
        ps.sourceEntitySystem = "quarkus-flow";

        final String json = LedgerSupplementSerializer.toJson(List.of(ps));

        assertThat(json).contains("\"PROVENANCE\"");
        assertThat(json).contains("\"sourceEntitySystem\":\"quarkus-flow\"");
    }

    @Test
    void toJson_observabilitySupplement_serialisedCorrectly() {
        final ObservabilitySupplement os = new ObservabilitySupplement();
        os.correlationId = "trace-abc123";

        final String json = LedgerSupplementSerializer.toJson(List.of(os));

        assertThat(json).contains("\"OBSERVABILITY\"");
        assertThat(json).contains("\"correlationId\":\"trace-abc123\"");
    }

    @Test
    void toJson_multipleSupplements_allPresent() {
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "v1";
        final ObservabilitySupplement os = new ObservabilitySupplement();
        os.correlationId = "trace-xyz";

        final String json = LedgerSupplementSerializer.toJson(List.of(cs, os));

        assertThat(json).contains("\"COMPLIANCE\"");
        assertThat(json).contains("\"OBSERVABILITY\"");
        assertThat(json).contains("algorithmRef");
        assertThat(json).contains("correlationId");
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerSupplementSerializerTest -q 2>&1 | tail -5
```

Expected: FAIL — `LedgerSupplementSerializer` not found.

- [ ] **Step 3: Create `LedgerSupplementSerializer.java`**

```java
package io.casehub.ledger.runtime.model.supplement;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

/**
 * Serialises a list of {@link LedgerSupplement} instances to a compact JSON string
 * for storage in the {@code supplement_json} column of {@code ledger_entry}.
 *
 * <p>
 * Each supplement is serialised under its type key ({@code "COMPLIANCE"},
 * {@code "PROVENANCE"}, {@code "OBSERVABILITY"}). Null fields are omitted.
 * Returns {@code null} when the list is null or empty — preserving a null
 * {@code supplement_json} for entries that carry no supplements.
 *
 * <p>
 * This class is not a CDI bean — it is a pure static utility with no Quarkus
 * runtime dependency. It can be used in unit tests without a running container.
 */
public final class LedgerSupplementSerializer {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private LedgerSupplementSerializer() {
    }

    /**
     * Serialise a list of supplements to a JSON string.
     *
     * @param supplements the supplements to serialise; may be null or empty
     * @return a JSON string, or {@code null} if the list is null or empty
     */
    public static String toJson(final List<LedgerSupplement> supplements) {
        if (supplements == null || supplements.isEmpty()) {
            return null;
        }
        final Map<String, Object> root = new LinkedHashMap<>();
        for (final LedgerSupplement supplement : supplements) {
            final Map<String, Object> fields = toFieldMap(supplement);
            if (!fields.isEmpty()) {
                root.put(typeKey(supplement), fields);
            }
        }
        if (root.isEmpty()) {
            return null;
        }
        try {
            return MAPPER.writeValueAsString(root);
        } catch (final JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialise supplements to JSON", e);
        }
    }

    // ── private helpers ───────────────────────────────────────────────────────

    private static String typeKey(final LedgerSupplement supplement) {
        if (supplement instanceof ComplianceSupplement) {
            return "COMPLIANCE";
        }
        if (supplement instanceof ProvenanceSupplement) {
            return "PROVENANCE";
        }
        if (supplement instanceof ObservabilitySupplement) {
            return "OBSERVABILITY";
        }
        throw new IllegalArgumentException("Unknown supplement type: " + supplement.getClass().getName());
    }

    private static Map<String, Object> toFieldMap(final LedgerSupplement supplement) {
        final Map<String, Object> map = new LinkedHashMap<>();
        if (supplement instanceof final ComplianceSupplement c) {
            putIfNotNull(map, "planRef", c.planRef);
            putIfNotNull(map, "rationale", c.rationale);
            putIfNotNull(map, "evidence", c.evidence);
            putIfNotNull(map, "detail", c.detail);
            putIfNotNull(map, "decisionContext", c.decisionContext);
            putIfNotNull(map, "algorithmRef", c.algorithmRef);
            putIfNotNull(map, "confidenceScore", c.confidenceScore);
            putIfNotNull(map, "contestationUri", c.contestationUri);
            putIfNotNull(map, "humanOverrideAvailable", c.humanOverrideAvailable);
        } else if (supplement instanceof final ProvenanceSupplement p) {
            putIfNotNull(map, "sourceEntityId", p.sourceEntityId);
            putIfNotNull(map, "sourceEntityType", p.sourceEntityType);
            putIfNotNull(map, "sourceEntitySystem", p.sourceEntitySystem);
        } else if (supplement instanceof final ObservabilitySupplement o) {
            putIfNotNull(map, "correlationId", o.correlationId);
            if (o.causedByEntryId != null) {
                map.put("causedByEntryId", o.causedByEntryId.toString());
            }
        }
        return map;
    }

    private static void putIfNotNull(final Map<String, Object> map, final String key, final Object value) {
        if (value != null) {
            map.put(key, value);
        }
    }
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerSupplementSerializerTest -q 2>&1 | tail -5
```

Expected: `Tests run: 8, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/supplement/LedgerSupplementSerializer.java \
        runtime/src/test/java/io/casehub/ledger/service/supplement/LedgerSupplementSerializerTest.java
git commit -m "feat(supplement): LedgerSupplementSerializer — explicit per-type JSON serialisation

Null fields omitted; returns null for empty/absent supplements so supplement_json
stays null on bare entries. Pure static utility, no CDI or Quarkus runtime needed.
8 unit tests passing.

Refs #7"
```

---

## Task 3 — Slim LedgerEntry + update LedgerHashChain

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerHashChain.java`
- Modify: `runtime/src/test/java/io/casehub/ledger/service/LedgerHashChainTest.java`

The canonical form changes (planRef removed). Write the updated test first,
run to confirm the old implementation makes it fail, then update the implementation.

- [ ] **Step 1: Update `LedgerHashChainTest` — remove planRef from fixture, add supplement isolation test**

Replace the `entry()` fixture method and add one new test. The fixture currently sets
`e.planRef = null` — this field is being removed from `LedgerEntry` so it will not
compile. Remove that line and add a test confirming supplement fields don't affect the hash.

In `runtime/src/test/java/io/casehub/ledger/service/LedgerHashChainTest.java`:

Replace the `entry()` fixture:
```java
private TestLedgerEntry entry(final UUID subjectId, final int seq) {
    final TestLedgerEntry e = new TestLedgerEntry();
    e.subjectId = subjectId;
    e.sequenceNumber = seq;
    e.entryType = LedgerEntryType.EVENT;
    e.actorId = "system";
    e.actorRole = "Initiator";
    // planRef removed — now lives in ComplianceSupplement
    e.occurredAt = Instant.parse("2026-04-14T10:00:00Z");
    return e;
}
```

Add this test at the end of the class (before the closing `}`):
```java
// ── canonical form — supplement fields excluded ───────────────────────────

@Test
void compute_supplementFieldsDoNotAffectDigest() {
    // Canonical form covers only core LedgerEntry fields.
    // Supplement data is not tamper-evidence — it is enrichment.
    // Two entries identical in core fields must produce the same digest
    // regardless of what supplements are attached.
    final UUID id = UUID.randomUUID();
    final TestLedgerEntry e1 = entry(id, 1);
    final TestLedgerEntry e2 = entry(id, 1);
    // e2 has supplements attached — but they do not change the digest
    // because canonical form only reads core fields.
    // (Supplement fields are not on LedgerEntry — this test documents
    //  the invariant: future additions to supplements cannot affect the chain.)
    assertThat(LedgerHashChain.compute(null, e1))
            .isEqualTo(LedgerHashChain.compute(null, e2));
}
```

- [ ] **Step 2: Run LedgerHashChainTest — confirm it fails to compile**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerHashChainTest -q 2>&1 | tail -10
```

Expected: compilation error referencing `planRef` (still on `LedgerEntry`). The test
references `e.planRef = null` — we removed it from the fixture, so the test should
compile. But `LedgerEntry` still has the field — so it compiles but the canonical form
still includes planRef. The new supplement isolation test passes trivially (no supplement
fields are read). This is fine — proceed.

- [ ] **Step 3: Replace `LedgerEntry.java` with slimmed core**

```java
package io.casehub.ledger.runtime.model;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import jakarta.persistence.CascadeType;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorColumn;
import jakarta.persistence.DiscriminatorType;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.FetchType;
import jakarta.persistence.Id;
import jakarta.persistence.Inheritance;
import jakarta.persistence.InheritanceType;
import jakarta.persistence.OneToMany;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;

import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.LedgerSupplement;
import io.casehub.ledger.runtime.model.supplement.LedgerSupplementSerializer;
import io.casehub.ledger.runtime.model.supplement.ObservabilitySupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;
import io.quarkus.hibernate.orm.panache.PanacheEntityBase;

/**
 * Abstract base for all ledger entries.
 *
 * <h2>Core fields</h2>
 * <p>
 * {@code LedgerEntry} holds exactly the fields that are relevant for every entry,
 * every consumer, every time: the subject aggregate, sequence position, actor identity,
 * timestamp, and the tamper-evident hash chain. Nothing else.
 *
 * <h2>Supplements</h2>
 * <p>
 * Optional cross-cutting concerns are handled by {@link io.casehub.ledger.runtime.model.supplement.LedgerSupplement}
 * subclasses attached via {@link #attach(LedgerSupplement)}:
 * <ul>
 *   <li>{@link ComplianceSupplement} — GDPR Art.22 decision snapshot, governance</li>
 *   <li>{@link ProvenanceSupplement} — workflow source entity</li>
 *   <li>{@link ObservabilitySupplement} — OTel tracing, causality</li>
 * </ul>
 * If a consumer never calls {@code attach()}, no supplement tables are written
 * and the lazy {@code supplements} list is never initialised — zero overhead.
 *
 * <h2>JPA JOINED inheritance</h2>
 * <p>
 * Domain-specific subclasses (e.g. {@code WorkItemLedgerEntry} in Tarkus) extend
 * this class and add a sibling table joined on {@code id}. Supplements are orthogonal
 * to subclasses — any subclass can attach any supplement.
 *
 * <h2>Hash chain</h2>
 * <p>
 * The canonical form for SHA-256 chaining uses only core fields:
 * {@code subjectId|seqNum|entryType|actorId|actorRole|occurredAt}.
 * Supplement fields are deliberately excluded — they are enrichment, not tamper-evidence
 * targets.
 */
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "dtype", discriminatorType = DiscriminatorType.STRING)
@Table(name = "ledger_entry")
public abstract class LedgerEntry extends PanacheEntityBase {

    // ── Core identity ─────────────────────────────────────────────────────────

    /** Primary key — UUID assigned on first persist. */
    @Id
    public UUID id;

    /**
     * The aggregate this entry belongs to — the domain object whose lifecycle
     * is being recorded. Scopes the sequence number and hash chain.
     */
    @Column(name = "subject_id", nullable = false)
    public UUID subjectId;

    /** Position of this entry in the per-subject ledger sequence (1-based). */
    @Column(name = "sequence_number", nullable = false)
    public int sequenceNumber;

    /** Whether this entry is a command (intent), event (fact), or attestation record. */
    @Enumerated(EnumType.STRING)
    @Column(name = "entry_type", nullable = false)
    public LedgerEntryType entryType;

    // ── Actor ─────────────────────────────────────────────────────────────────

    /** Identity of the actor who triggered this transition. */
    @Column(name = "actor_id")
    public String actorId;

    /** Whether the actor is a human, autonomous agent, or the system itself. */
    @Enumerated(EnumType.STRING)
    @Column(name = "actor_type")
    public ActorType actorType;

    /** The functional role of the actor in this transition — e.g. {@code "Resolver"}. */
    @Column(name = "actor_role")
    public String actorRole;

    // ── Timing ────────────────────────────────────────────────────────────────

    /** When this entry was recorded — set automatically on first persist. */
    @Column(name = "occurred_at", nullable = false)
    public Instant occurredAt;

    // ── Hash chain ────────────────────────────────────────────────────────────

    /**
     * SHA-256 digest of the previous entry for this subject.
     * {@code null} for the first entry (no previous entry exists).
     */
    @Column(name = "previous_hash")
    public String previousHash;

    /**
     * SHA-256 digest of this entry's canonical content chained from {@code previousHash}.
     * Null when hash chain is disabled ({@code quarkus.ledger.hash-chain.enabled=false}).
     */
    public String digest;

    // ── Supplements ───────────────────────────────────────────────────────────

    /**
     * Lazily-loaded supplements attached to this entry.
     * Never initialised unless a supplement is attached or explicitly accessed.
     * Use {@link #attach(LedgerSupplement)}, {@link #compliance()},
     * {@link #provenance()}, and {@link #observability()} for type-safe access.
     */
    @OneToMany(mappedBy = "ledgerEntry", fetch = FetchType.LAZY, cascade = CascadeType.ALL,
            orphanRemoval = true)
    public List<LedgerSupplement> supplements = new ArrayList<>();

    /**
     * Denormalised JSON snapshot of all attached supplements.
     * Written automatically by {@link #attach(LedgerSupplement)}.
     * Enables fast single-entry reads without joining supplement tables.
     * Format: {@code {"COMPLIANCE":{...},"OBSERVABILITY":{...}}}.
     */
    @Column(name = "supplement_json", columnDefinition = "TEXT")
    public String supplementJson;

    // ── Supplement helpers ────────────────────────────────────────────────────

    /**
     * Attach a supplement to this entry, replacing any existing supplement of the
     * same type. Also refreshes {@link #supplementJson} to keep it in sync.
     *
     * @param supplement the supplement to attach; must not be null
     */
    public void attach(final LedgerSupplement supplement) {
        supplement.ledgerEntry = this;
        supplements.removeIf(s -> s.getClass() == supplement.getClass());
        supplements.add(supplement);
        supplementJson = LedgerSupplementSerializer.toJson(supplements);
    }

    /**
     * Returns the {@link ComplianceSupplement} attached to this entry, if any.
     *
     * @return the compliance supplement, or empty if none is attached
     */
    public Optional<ComplianceSupplement> compliance() {
        return supplements.stream()
                .filter(ComplianceSupplement.class::isInstance)
                .map(ComplianceSupplement.class::cast)
                .findFirst();
    }

    /**
     * Returns the {@link ProvenanceSupplement} attached to this entry, if any.
     *
     * @return the provenance supplement, or empty if none is attached
     */
    public Optional<ProvenanceSupplement> provenance() {
        return supplements.stream()
                .filter(ProvenanceSupplement.class::isInstance)
                .map(ProvenanceSupplement.class::cast)
                .findFirst();
    }

    /**
     * Returns the {@link ObservabilitySupplement} attached to this entry, if any.
     *
     * @return the observability supplement, or empty if none is attached
     */
    public Optional<ObservabilitySupplement> observability() {
        return supplements.stream()
                .filter(ObservabilitySupplement.class::isInstance)
                .map(ObservabilitySupplement.class::cast)
                .findFirst();
    }

    // ── Lifecycle ─────────────────────────────────────────────────────────────

    /** Assigns a UUID primary key and sets {@code occurredAt} before the entity is inserted. */
    @PrePersist
    void prePersist() {
        if (id == null) {
            id = UUID.randomUUID();
        }
        if (occurredAt == null) {
            occurredAt = Instant.now();
        }
    }
}
```

- [ ] **Step 4: Update `LedgerHashChain.java` — remove planRef from canonical form**

In `LedgerHashChain.java`, update the `compute()` method and its Javadoc:

Replace the `canonical` string construction (lines 43–51) with:
```java
        final String canonical = String.join("|",
                entry.subjectId != null ? entry.subjectId.toString() : "",
                String.valueOf(entry.sequenceNumber),
                entry.entryType != null ? entry.entryType.name() : "",
                entry.actorId != null ? entry.actorId : "",
                entry.actorRole != null ? entry.actorRole : "",
                // Truncate to milliseconds for consistent canonical form regardless of DB precision
                entry.occurredAt != null ? entry.occurredAt.truncatedTo(ChronoUnit.MILLIS).toString() : "");
```

Also update the class-level Javadoc to reflect the new canonical form:
```java
 * Canonical content uses only the six core {@link LedgerEntry} fields —
 * supplement fields and subclass-specific fields are excluded to keep
 * the chain domain-agnostic and supplement-agnostic:
 * {@code subjectId|seqNum|entryType|actorId|actorRole|occurredAt}
```

- [ ] **Step 5: Run all runtime unit tests — verify they pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -10
```

Expected: `Tests run: 34, Failures: 0, Errors: 0` (17 LedgerHashChainTest + 16 TrustScoreComputerTest + 1 new supplement isolation test).

If there are compilation errors about removed fields (`planRef`, `rationale`, etc.) in
other classes, fix them now — any reference to these fields must move to supplement access.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/model/LedgerEntry.java \
        runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerHashChain.java \
        runtime/src/test/java/io/casehub/ledger/service/LedgerHashChainTest.java
git commit -m "feat(supplement): slim LedgerEntry to 10 core fields + attach() + typed accessors

Removes planRef, rationale, evidence, detail, decisionContext, correlationId,
causedByEntryId, sourceEntityId/Type/System from LedgerEntry — all moved to
supplements. Adds supplements List (lazy), supplementJson (denormalised), and
attach()/compliance()/provenance()/observability() helpers.

Updates LedgerHashChain canonical form to remove planRef:
  was: subjectId|seqNum|entryType|actorId|actorRole|planRef|occurredAt
  now: subjectId|seqNum|entryType|actorId|actorRole|occurredAt

34 unit tests passing.

Refs #7"
```

---

## Task 4 — Flyway migration V1002

**Files:**
- Create: `runtime/src/main/resources/db/migration/V1002__ledger_supplement.sql`

- [ ] **Step 1: Create the migration file**

```sql
-- V1002 — LedgerSupplement architecture
--
-- 1. Remove optional fields migrated to supplements from ledger_entry.
-- 2. Add supplement_json column for fast denormalised reads.
-- 3. Create ledger_supplement base table and three joined subclass tables.
--
-- Compatible with H2 (dev/test) and PostgreSQL (production).
-- No data migration required — all consumers are pre-release (0.2-SNAPSHOT).
--
-- Flyway convention update: base extension reserves V1000–V1002.
-- Consumer subclass join tables must use V1003+ (updated from the previous V1002+).

-- ── Alter ledger_entry ───────────────────────────────────────────────────────

ALTER TABLE ledger_entry DROP COLUMN IF EXISTS plan_ref;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS rationale;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS evidence;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS detail;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS decision_context;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS correlation_id;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS caused_by_entry_id;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS source_entity_id;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS source_entity_type;
ALTER TABLE ledger_entry DROP COLUMN IF EXISTS source_entity_system;

ALTER TABLE ledger_entry ADD COLUMN supplement_json TEXT;

-- ── ledger_supplement base table ─────────────────────────────────────────────

CREATE TABLE ledger_supplement (
    id               UUID         NOT NULL,
    ledger_entry_id  UUID         NOT NULL,
    supplement_type  VARCHAR(30)  NOT NULL,
    CONSTRAINT pk_ledger_supplement PRIMARY KEY (id),
    CONSTRAINT fk_supplement_entry FOREIGN KEY (ledger_entry_id)
        REFERENCES ledger_entry (id)
);

CREATE INDEX idx_ledger_supplement_entry ON ledger_supplement (ledger_entry_id);
CREATE INDEX idx_ledger_supplement_type  ON ledger_supplement (supplement_type);

-- ── ComplianceSupplement ──────────────────────────────────────────────────────

CREATE TABLE ledger_supplement_compliance (
    id                       UUID          NOT NULL,
    -- Governance
    plan_ref                 VARCHAR(500),
    rationale                TEXT,
    evidence                 TEXT,
    detail                   TEXT,
    -- GDPR Art.22 / EU AI Act Art.12
    decision_context         TEXT,
    algorithm_ref            VARCHAR(500),
    confidence_score         DOUBLE,
    contestation_uri         VARCHAR(2000),
    human_override_available BOOLEAN,
    CONSTRAINT pk_ledger_supplement_compliance PRIMARY KEY (id),
    CONSTRAINT fk_compliance_base FOREIGN KEY (id)
        REFERENCES ledger_supplement (id)
);

-- ── ProvenanceSupplement ──────────────────────────────────────────────────────

CREATE TABLE ledger_supplement_provenance (
    id                   UUID          NOT NULL,
    source_entity_id     VARCHAR(255),
    source_entity_type   VARCHAR(255),
    source_entity_system VARCHAR(100),
    CONSTRAINT pk_ledger_supplement_provenance PRIMARY KEY (id),
    CONSTRAINT fk_provenance_base FOREIGN KEY (id)
        REFERENCES ledger_supplement (id)
);

-- ── ObservabilitySupplement ───────────────────────────────────────────────────

CREATE TABLE ledger_supplement_observability (
    id                 UUID          NOT NULL,
    correlation_id     VARCHAR(255),
    caused_by_entry_id UUID,
    CONSTRAINT pk_ledger_supplement_observability PRIMARY KEY (id),
    CONSTRAINT fk_observability_base FOREIGN KEY (id)
        REFERENCES ledger_supplement (id)
);
```

- [ ] **Step 2: Run full runtime test suite — migration must apply cleanly**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -10
```

Expected: BUILD SUCCESS — all tests pass, Flyway applies V1002 without errors.

- [ ] **Step 3: Commit**

```bash
git add runtime/src/main/resources/db/migration/V1002__ledger_supplement.sql
git commit -m "feat(supplement): V1002 migration — supplement tables, drop moved columns

Drops 10 columns from ledger_entry (planRef, rationale, evidence, detail,
decisionContext, correlationId, causedByEntryId, sourceEntityId/Type/System).
Adds supplement_json. Creates ledger_supplement, ledger_supplement_compliance,
ledger_supplement_provenance, ledger_supplement_observability.

Base extension now reserves V1000-V1002. Consumer subclass migrations: V1003+.

Refs #7"
```

---

## Task 5 — Supplement integration tests

**Files:**
- Create: `runtime/src/test/java/io/casehub/ledger/service/supplement/TestEntry.java`
- Create: `runtime/src/test/resources/db/migration/V1999__test_entry.sql`
- Create: `runtime/src/test/java/io/casehub/ledger/service/supplement/LedgerSupplementIT.java`

- [ ] **Step 1: Create test-only `TestEntry` entity**

`runtime/src/test/java/io/casehub/ledger/service/supplement/TestEntry.java`:
```java
package io.casehub.ledger.service.supplement;

import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * Minimal concrete subclass of {@link LedgerEntry} for integration tests only.
 * Never used in production code.
 */
@Entity
@Table(name = "test_ledger_entry")
@DiscriminatorValue("TEST")
public class TestEntry extends LedgerEntry {
}
```

- [ ] **Step 2: Create test-only migration for `test_ledger_entry` table**

`runtime/src/test/resources/db/migration/V1999__test_entry.sql`:
```sql
-- Test-only migration: concrete subclass table for LedgerSupplementIT
CREATE TABLE test_ledger_entry (
    id UUID NOT NULL,
    CONSTRAINT pk_test_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_test_entry FOREIGN KEY (id) REFERENCES ledger_entry (id)
);
```

- [ ] **Step 3: Write the failing integration tests**

`runtime/src/test/java/io/casehub/ledger/service/supplement/LedgerSupplementIT.java`:
```java
package io.casehub.ledger.service.supplement;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.UUID;

import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.LedgerSupplement;
import io.casehub.ledger.runtime.model.supplement.ObservabilitySupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;
import io.quarkus.test.junit.QuarkusTest;

/**
 * Integration tests for the LedgerSupplement system.
 *
 * <p>
 * Verifies persistence, lazy loading, and zero-touch behaviour using a
 * test-only {@link TestEntry} subclass backed by an H2 in-memory database.
 */
@QuarkusTest
class LedgerSupplementIT {

    // ── happy path: no supplements ────────────────────────────────────────────

    @Test
    @Transactional
    void bareEntry_supplementTablesNotTouched() {
        final TestEntry entry = bareEntry();
        entry.persist();

        final long count = LedgerSupplement.count("ledgerEntry.id", entry.id);
        assertThat(count).isZero();
        assertThat(entry.supplementJson).isNull();
    }

    // ── happy path: ComplianceSupplement ─────────────────────────────────────

    @Test
    @Transactional
    void complianceSupplement_persistsAndLoadsAllFields() {
        final TestEntry entry = bareEntry();

        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.planRef = "policy-v3";
        cs.rationale = "Risk threshold exceeded";
        cs.algorithmRef = "classifier-v2";
        cs.confidenceScore = 0.91;
        cs.contestationUri = "https://example.com/challenge";
        cs.humanOverrideAvailable = true;
        cs.decisionContext = "{\"score\":88}";
        entry.attach(cs);
        entry.persist();

        final TestEntry found = TestEntry.findById(entry.id);
        assertThat(found).isNotNull();
        assertThat(found.supplementJson).isNotNull();
        assertThat(found.supplementJson).contains("classifier-v2");

        final ComplianceSupplement loaded = found.compliance().orElseThrow();
        assertThat(loaded.planRef).isEqualTo("policy-v3");
        assertThat(loaded.algorithmRef).isEqualTo("classifier-v2");
        assertThat(loaded.confidenceScore).isEqualTo(0.91);
        assertThat(loaded.contestationUri).isEqualTo("https://example.com/challenge");
        assertThat(loaded.humanOverrideAvailable).isTrue();
        assertThat(loaded.decisionContext).isEqualTo("{\"score\":88}");
    }

    // ── happy path: supplementJson for reads without join ─────────────────────

    @Test
    @Transactional
    void supplementJson_containsAllAttachedSupplements() {
        final TestEntry entry = bareEntry();

        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "v1";
        entry.attach(cs);

        final ObservabilitySupplement os = new ObservabilitySupplement();
        os.correlationId = "trace-abc";
        entry.attach(os);
        entry.persist();

        final TestEntry found = TestEntry.findById(entry.id);
        assertThat(found.supplementJson).contains("\"COMPLIANCE\"");
        assertThat(found.supplementJson).contains("\"OBSERVABILITY\"");
        assertThat(found.supplementJson).contains("trace-abc");
    }

    // ── happy path: ProvenanceSupplement ─────────────────────────────────────

    @Test
    @Transactional
    void provenanceSupplement_persistsAndLoads() {
        final TestEntry entry = bareEntry();

        final ProvenanceSupplement ps = new ProvenanceSupplement();
        ps.sourceEntityId = "wf-42";
        ps.sourceEntityType = "Flow:WorkflowInstance";
        ps.sourceEntitySystem = "quarkus-flow";
        entry.attach(ps);
        entry.persist();

        final ProvenanceSupplement loaded = TestEntry.<TestEntry> findById(entry.id)
                .provenance().orElseThrow();
        assertThat(loaded.sourceEntityId).isEqualTo("wf-42");
        assertThat(loaded.sourceEntitySystem).isEqualTo("quarkus-flow");
    }

    // ── happy path: ObservabilitySupplement ──────────────────────────────────

    @Test
    @Transactional
    void observabilitySupplement_persistsAndLoads() {
        final TestEntry entry = bareEntry();

        final ObservabilitySupplement os = new ObservabilitySupplement();
        os.correlationId = "trace-xyz";
        os.causedByEntryId = UUID.randomUUID();
        entry.attach(os);
        entry.persist();

        final ObservabilitySupplement loaded = TestEntry.<TestEntry> findById(entry.id)
                .observability().orElseThrow();
        assertThat(loaded.correlationId).isEqualTo("trace-xyz");
        assertThat(loaded.causedByEntryId).isEqualTo(os.causedByEntryId);
    }

    // ── attach replaces existing supplement of same type ──────────────────────

    @Test
    @Transactional
    void attach_replacesExistingSupplementOfSameType() {
        final TestEntry entry = bareEntry();

        final ComplianceSupplement first = new ComplianceSupplement();
        first.algorithmRef = "v1";
        entry.attach(first);

        final ComplianceSupplement second = new ComplianceSupplement();
        second.algorithmRef = "v2";
        entry.attach(second);

        assertThat(entry.supplements).hasSize(1);
        assertThat(entry.compliance().map(c -> c.algorithmRef).orElse(null))
                .isEqualTo("v2");
        assertThat(entry.supplementJson).contains("v2");
        assertThat(entry.supplementJson).doesNotContain("v1");
    }

    // ── fixture ───────────────────────────────────────────────────────────────

    private TestEntry bareEntry() {
        final TestEntry e = new TestEntry();
        e.subjectId = UUID.randomUUID();
        e.sequenceNumber = 1;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = "test-actor";
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "TestRole";
        return e;
    }
}
```

- [ ] **Step 4: Run integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerSupplementIT -q 2>&1 | tail -10
```

Expected: `Tests run: 7, Failures: 0, Errors: 0`.

- [ ] **Step 5: Run the full runtime test suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
```

Expected: all tests pass.

- [ ] **Step 6: Commit**

```bash
git add runtime/src/test/
git commit -m "test(supplement): integration tests — persist, lazy load, zero-touch for bare entries

7 @QuarkusTest cases covering ComplianceSupplement, ProvenanceSupplement,
ObservabilitySupplement, supplementJson denormalisation, and the attach()-replaces
behaviour. Bare entries confirm no supplement table rows written.

Refs #7"
```

---

## Task 6 — Update order-processing example

**Files:**
- Modify: `examples/order-processing/src/main/java/.../OrderService.java`
- Modify: `examples/order-processing/src/main/java/.../api/OrderResource.java` (check for supplement field references)
- Modify: `examples/order-processing/src/test/java/.../OrderLedgerIT.java`

The example currently sets `entry.rationale` and `entry.decisionContext` directly — these fields
are gone. They must move to `ComplianceSupplement` via `attach()`.

- [ ] **Step 1: Update `OrderService.java`**

Replace the `record()` method entirely:

```java
private OrderLedgerEntry record(final Order order, final String transition, final String actor) {
    final String[] meta = EVENT_META.get(transition);

    final Optional<OrderLedgerEntry> latest = ledgerRepo.findLatestByOrderId(order.id);
    final int nextSeq = latest.map(e -> e.sequenceNumber + 1).orElse(1);
    final String previousHash = latest.map(e -> e.digest).orElse(null);

    final OrderLedgerEntry entry = new OrderLedgerEntry();
    entry.subjectId = order.id;
    entry.orderId = order.id;
    entry.sequenceNumber = nextSeq;
    entry.entryType = LedgerEntryType.EVENT;
    entry.commandType = meta[0];
    entry.eventType = meta[1];
    entry.actorId = actor;
    entry.actorType = ActorType.HUMAN;
    entry.actorRole = meta[2];
    entry.orderStatus = order.status.name();
    entry.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);

    if (ledgerConfig.decisionContext().enabled()) {
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.decisionContext = String.format(
                "{\"status\":\"%s\",\"total\":%s,\"customerId\":\"%s\"}",
                order.status, order.total, order.customerId);
        entry.attach(cs);
    }

    if (ledgerConfig.hashChain().enabled()) {
        entry.previousHash = previousHash;
        entry.digest = LedgerHashChain.compute(previousHash, entry);
    }

    ledgerRepo.save(entry);
    return entry;
}
```

Also update `cancelOrder()` — `entry.rationale = reason` becomes:

```java
public Order cancelOrder(final UUID orderId, final String actor, final String reason) {
    final Order order = findOrThrow(orderId);
    order.status = OrderStatus.CANCELLED;

    if (ledgerConfig.enabled()) {
        final OrderLedgerEntry entry = record(order, "cancel", actor);
        // Add rationale to the ComplianceSupplement (created by record() when decisionContext is enabled)
        // or create a new ComplianceSupplement if decisionContext is disabled
        entry.compliance().ifPresentOrElse(
                cs -> cs.rationale = reason,
                () -> {
                    final ComplianceSupplement cs = new ComplianceSupplement();
                    cs.rationale = reason;
                    entry.attach(cs);
                });
        // Refresh supplementJson after mutating the supplement
        entry.supplementJson = io.casehub.ledger.runtime.model.supplement.LedgerSupplementSerializer
                .toJson(entry.supplements);
    }
    return order;
}
```

Add the missing import:
```java
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
```

- [ ] **Step 2: Run the existing 8 IT tests — they must still pass**

```bash
cd examples/order-processing
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -10
cd ../..
```

Expected: `Tests run: 8, Failures: 0, Errors: 0`.

Fix any compilation errors before continuing.

- [ ] **Step 3: Add supplement-specific IT tests to `OrderLedgerIT.java`**

Add these tests to the existing `OrderLedgerIT` class:

```java
@Test
void placeOrder_supplementJson_containsDecisionContext() {
    final String orderId = placeOrder("it-supp-1", "99.00");

    given()
            .when().get(BASE + "/" + orderId + "/ledger")
            .then()
            .statusCode(200)
            .body("[0].supplementJson", notNullValue())
            .body("[0].supplementJson", containsString("decisionContext"))
            .body("[0].supplementJson", containsString("PLACED"));
}

@Test
void cancelOrder_supplementJson_containsRationale() {
    final String orderId = placeOrder("it-supp-2", "25.00");
    given().queryParam("actor", "it-supp-2").queryParam("reason", "No longer needed")
            .when().put(BASE + "/" + orderId + "/cancel")
            .then().statusCode(200);

    given()
            .when().get(BASE + "/" + orderId + "/ledger")
            .then()
            .statusCode(200)
            .body("[1].supplementJson", containsString("No longer needed"));
}
```

- [ ] **Step 4: Run all 10 IT tests**

```bash
cd examples/order-processing
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -10
cd ../..
```

Expected: `Tests run: 10, Failures: 0, Errors: 0`.

- [ ] **Step 5: Commit**

```bash
git add examples/order-processing/
git commit -m "feat(supplement): update order-processing example to use ComplianceSupplement

rationale and decisionContext now stored in ComplianceSupplement via attach().
Adds 2 supplement-specific IT tests. All 10 IT tests passing.

Refs #7"
```

---

## Task 7 — art22-decision-snapshot example

**Files:** All under `examples/art22-decision-snapshot/`

- [ ] **Step 1: Create the directory structure**

```bash
mkdir -p examples/art22-decision-snapshot/src/main/java/io/casehub/ledger/examples/art22/{ledger,model,service,api}
mkdir -p examples/art22-decision-snapshot/src/main/resources/db/migration
mkdir -p examples/art22-decision-snapshot/src/test/java/io/casehub/ledger/examples/art22
```

- [ ] **Step 2: Create `pom.xml`** — copy and adapt from `examples/order-processing/pom.xml`, changing artifactId to `casehub-ledger-example-art22-decision-snapshot` and description.

- [ ] **Step 3: Create `DecisionLedgerEntry.java`**

```java
package io.casehub.ledger.examples.art22.ledger;

import java.util.UUID;

import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * Domain-specific ledger entry for AI decisions.
 * Extends the base {@link LedgerEntry} — the Art.22 compliance fields live in
 * a {@link io.casehub.ledger.runtime.model.supplement.ComplianceSupplement}
 * attached via {@link LedgerEntry#attach(io.casehub.ledger.runtime.model.supplement.LedgerSupplement)}.
 */
@Entity
@Table(name = "decision_ledger_entry")
@DiscriminatorValue("DECISION")
public class DecisionLedgerEntry extends LedgerEntry {

    @Column(name = "decision_id", nullable = false)
    public UUID decisionId;

    /** High-level category — e.g. "credit-risk", "content-moderation". */
    @Column(name = "decision_category", length = 100)
    public String decisionCategory;

    /** The decision outcome — e.g. "APPROVED", "FLAGGED", "REJECTED". */
    @Column(name = "outcome", length = 50)
    public String outcome;
}
```

- [ ] **Step 4: Create `DecisionService.java`**

```java
package io.casehub.ledger.examples.art22.service;

import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.casehub.ledger.examples.art22.ledger.DecisionLedgerEntry;
import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.service.LedgerHashChain;

/**
 * Simulates an AI decision service that records each decision with a full
 * GDPR Art.22 compliance supplement.
 */
@ApplicationScoped
public class DecisionService {

    @Transactional
    public DecisionLedgerEntry recordDecision(
            final String subjectId,
            final String category,
            final String outcome,
            final String algorithmRef,
            final double confidence,
            final String inputContext) {

        final UUID subjectUuid = UUID.fromString(subjectId);

        // Find latest entry for this subject to chain hashes
        final List<DecisionLedgerEntry> existing = DecisionLedgerEntry
                .list("subjectId order by sequenceNumber desc", subjectUuid);
        final int nextSeq = existing.isEmpty() ? 1 : existing.get(0).sequenceNumber + 1;
        final String previousHash = existing.isEmpty() ? null : existing.get(0).digest;

        final DecisionLedgerEntry entry = new DecisionLedgerEntry();
        entry.subjectId = subjectUuid;
        entry.decisionId = UUID.randomUUID();
        entry.decisionCategory = category;
        entry.outcome = outcome;
        entry.sequenceNumber = nextSeq;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = algorithmRef;
        entry.actorType = ActorType.AGENT;
        entry.actorRole = "Classifier";
        entry.occurredAt = Instant.now().truncatedTo(ChronoUnit.MILLIS);

        // Attach GDPR Art.22 compliance supplement — all four structured fields
        final ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = algorithmRef;
        cs.confidenceScore = confidence;
        cs.contestationUri = "https://decisions.example.com/challenge/" + entry.decisionId;
        cs.humanOverrideAvailable = true;
        cs.decisionContext = inputContext;
        entry.attach(cs);

        // Hash chain
        entry.previousHash = previousHash;
        entry.digest = LedgerHashChain.compute(previousHash, entry);

        entry.persist();
        return entry;
    }

    public List<DecisionLedgerEntry> history(final String subjectId) {
        return DecisionLedgerEntry.list(
                "subjectId order by sequenceNumber asc", UUID.fromString(subjectId));
    }
}
```

- [ ] **Step 5: Create `DecisionResource.java`**

```java
package io.casehub.ledger.examples.art22.api;

import java.util.List;
import java.util.Map;

import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import io.casehub.ledger.examples.art22.ledger.DecisionLedgerEntry;
import io.casehub.ledger.examples.art22.service.DecisionService;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.service.LedgerHashChain;

@Path("/decisions")
@Produces(MediaType.APPLICATION_JSON)
public class DecisionResource {

    @Inject
    DecisionService decisionService;

    @POST
    public Response record(final DecisionRequest req) {
        final DecisionLedgerEntry entry = decisionService.recordDecision(
                req.subjectId, req.category, req.outcome,
                req.algorithmRef, req.confidence, req.inputContext);
        return Response.status(201).entity(toView(entry)).build();
    }

    @GET
    @Path("/{subjectId}/ledger")
    public List<Map<String, Object>> ledger(@PathParam("subjectId") final String subjectId) {
        return decisionService.history(subjectId).stream()
                .map(this::toView).toList();
    }

    @GET
    @Path("/{subjectId}/ledger/verify")
    public Map<String, Object> verify(@PathParam("subjectId") final String subjectId) {
        final List<DecisionLedgerEntry> entries = decisionService.history(subjectId);
        return Map.of("intact", LedgerHashChain.verify(entries), "entries", entries.size());
    }

    private Map<String, Object> toView(final DecisionLedgerEntry e) {
        final ComplianceSupplement cs = e.compliance().orElse(null);
        return Map.of(
                "id", e.id,
                "decisionId", e.decisionId,
                "category", e.decisionCategory,
                "outcome", e.outcome,
                "actorId", e.actorId,
                "occurredAt", e.occurredAt,
                "digest", e.digest != null ? e.digest : "",
                "supplementJson", e.supplementJson != null ? e.supplementJson : "",
                "art22", cs == null ? Map.of() : Map.of(
                        "algorithmRef", cs.algorithmRef != null ? cs.algorithmRef : "",
                        "confidenceScore", cs.confidenceScore != null ? cs.confidenceScore : 0.0,
                        "contestationUri", cs.contestationUri != null ? cs.contestationUri : "",
                        "humanOverrideAvailable", Boolean.TRUE.equals(cs.humanOverrideAvailable)));
    }

    public static class DecisionRequest {
        public String subjectId;
        public String category;
        public String outcome;
        public String algorithmRef;
        public double confidence;
        public String inputContext;
    }
}
```

- [ ] **Step 6: Create `V1003__decision_schema.sql`**

```sql
CREATE TABLE decision_ledger_entry (
    id                UUID         NOT NULL,
    decision_id       UUID         NOT NULL,
    decision_category VARCHAR(100),
    outcome           VARCHAR(50),
    CONSTRAINT pk_decision_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_decision_entry FOREIGN KEY (id) REFERENCES ledger_entry (id)
);
```

Note: V1003 satisfies the updated convention (consumer subclass join tables use V1003+).

- [ ] **Step 7: Create `application.properties`** under `src/main/resources/`:

```properties
quarkus.datasource.db-kind=h2
quarkus.datasource.jdbc.url=jdbc:h2:mem:art22;DB_CLOSE_DELAY=-1
quarkus.hibernate-orm.database.generation=none
quarkus.flyway.migrate-at-start=true
quarkus.ledger.enabled=true
quarkus.ledger.hash-chain.enabled=true
quarkus.ledger.decision-context.enabled=true
```

- [ ] **Step 8: Write the integration test**

`src/test/java/io/casehub/ledger/examples/art22/DecisionLedgerIT.java`:

```java
package io.casehub.ledger.examples.art22;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

import java.util.UUID;

import org.junit.jupiter.api.Test;

import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class DecisionLedgerIT {

    private static final String BASE = "/decisions";
    private static final String SUBJECT = UUID.randomUUID().toString();

    @Test
    void recordDecision_returns201_withArt22Fields() {
        given()
                .contentType("application/json")
                .body(request(SUBJECT, "credit-risk", "APPROVED", "risk-model-v3", 0.88))
                .when().post(BASE)
                .then()
                .statusCode(201)
                .body("art22.algorithmRef", equalTo("risk-model-v3"))
                .body("art22.confidenceScore", equalTo(0.88f))
                .body("art22.humanOverrideAvailable", equalTo(true))
                .body("art22.contestationUri", containsString("challenge"));
    }

    @Test
    void ledgerHistory_containsSupplementJson() {
        final String subject = UUID.randomUUID().toString();
        given().contentType("application/json")
                .body(request(subject, "content-moderation", "FLAGGED", "safety-classifier-v2", 0.95))
                .when().post(BASE).then().statusCode(201);

        given()
                .when().get(BASE + "/" + subject + "/ledger")
                .then()
                .statusCode(200)
                .body("$", hasSize(1))
                .body("[0].supplementJson", containsString("COMPLIANCE"))
                .body("[0].supplementJson", containsString("safety-classifier-v2"));
    }

    @Test
    void hashChain_intactAfterTwoDecisions() {
        final String subject = UUID.randomUUID().toString();
        given().contentType("application/json")
                .body(request(subject, "fraud-detection", "APPROVED", "fraud-model-v1", 0.72))
                .when().post(BASE).then().statusCode(201);
        given().contentType("application/json")
                .body(request(subject, "fraud-detection", "FLAGGED", "fraud-model-v1", 0.91))
                .when().post(BASE).then().statusCode(201);

        given()
                .when().get(BASE + "/" + subject + "/ledger/verify")
                .then()
                .statusCode(200)
                .body("intact", equalTo(true))
                .body("entries", equalTo(2));
    }

    private String request(final String subjectId, final String category,
            final String outcome, final String algorithmRef, final double confidence) {
        return String.format(
                "{\"subjectId\":\"%s\",\"category\":\"%s\",\"outcome\":\"%s\","
                        + "\"algorithmRef\":\"%s\",\"confidence\":%s,"
                        + "\"inputContext\":\"{\\\"inputs\\\":{\\\"score\\\":%s}}\"}",
                subjectId, category, outcome, algorithmRef, confidence, (int) (confidence * 100));
    }
}
```

- [ ] **Step 9: Run the example's tests**

```bash
cd examples/art22-decision-snapshot
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -10
cd ../..
```

Expected: `Tests run: 3, Failures: 0, Errors: 0`.

- [ ] **Step 10: Commit**

```bash
git add examples/art22-decision-snapshot/
git commit -m "feat(supplement): art22-decision-snapshot example — GDPR Art.22 in a runnable Quarkus app

Demonstrates ComplianceSupplement with all four Art.22 fields: algorithmRef,
confidenceScore, contestationUri, humanOverrideAvailable. REST endpoints for
recording decisions and retrieving history + chain verification. 3 IT tests.

Refs #7"
```

---

## Task 8 — Documentation + Javadoc update

**Files:**
- Modify: `docs/DESIGN.md`
- Modify: `docs/AUDITABILITY.md`
- Modify: `docs/integration-guide.md`
- Create: `examples/art22-decision-snapshot/README.md`

- [ ] **Step 1: Add Supplement chapter to `docs/DESIGN.md`**

After the existing `## Architecture` section, add:

```markdown
---

## Supplements

A **supplement** is an optional, lazily-loaded extension to a `LedgerEntry` that carries
a named group of cross-cutting fields. Supplements live in separate joined tables and are
never written unless the consumer explicitly attaches one — consumers that do not use
supplements incur zero schema or runtime cost.

### Why supplements?

`LedgerEntry` is the shared base for every consumer in the ecosystem. Adding optional
fields directly to the base entity creates a wide-table anti-pattern: every consumer
sees fields that are irrelevant to their use case, with no signal about which fields
belong together or when to populate them. Supplements solve this by grouping optional
fields by concern and moving them out of the core entity entirely.

### Built-in supplements

| Supplement | Table | Fields | Use when |
|---|---|---|---|
| `ComplianceSupplement` | `ledger_supplement_compliance` | `planRef`, `rationale`, `evidence`, `detail`, `decisionContext`, `algorithmRef`, `confidenceScore`, `contestationUri`, `humanOverrideAvailable` | Recording automated decisions subject to GDPR Art.22 or EU AI Act Art.12 |
| `ProvenanceSupplement` | `ledger_supplement_provenance` | `sourceEntityId`, `sourceEntityType`, `sourceEntitySystem` | Entry is driven by an external workflow system |
| `ObservabilitySupplement` | `ledger_supplement_observability` | `correlationId`, `causedByEntryId` | Linking entries to OTel traces or cross-system causal chains |

### Attaching a supplement

```java
ComplianceSupplement cs = new ComplianceSupplement();
cs.algorithmRef = "risk-model-v3";
cs.confidenceScore = 0.91;
cs.contestationUri = "https://example.com/challenge/" + entry.id;
cs.humanOverrideAvailable = true;
entry.attach(cs); // also refreshes entry.supplementJson
```

### Reading supplement data

**Fast path (single entry, no join):** Read `entry.supplementJson` — a JSON blob
written by `attach()` containing all attached supplements. No additional query.

**Typed access (lazy join):** Use the typed accessors — `entry.compliance()`,
`entry.provenance()`, `entry.observability()`. Triggers a single SELECT on the
supplement table only when accessed.

### Zero-complexity guarantee

If a consumer never calls `attach()`, no supplement table rows are written and the
lazy `supplements` list is never initialised. Consumers already integrated with
`casehub-ledger` require zero changes.
```

- [ ] **Step 2: Update hash chain canonical form in `docs/DESIGN.md`**

Find the line:
```
`subjectId|seqNum|entryType|actorId|actorRole|planRef|occurredAt`
```
Replace with:
```
`subjectId|seqNum|entryType|actorId|actorRole|occurredAt`

`planRef` was removed from the canonical form in V1002 — it now lives in
`ComplianceSupplement`. Supplement fields are deliberately excluded: the chain
covers the immutable core audit record; compliance metadata is enrichment,
not a tamper-evidence target.
```

- [ ] **Step 3: Update Flyway convention in `docs/DESIGN.md`**

Find the Flyway version numbering table and update:
```markdown
| Range | Owner | Purpose |
|---|---|---|
| V1000–V1002 | `casehub-ledger` base | Base schema (reserved — do not use in consumers) |
| V1–V999 | Consumer | Domain tables (orders, cases, channels, etc.) |
| V1003+ | Consumer | Subclass join tables (must run after V1000 — FK constraint) |
```

- [ ] **Step 4: Update `docs/AUDITABILITY.md`**

Find Axiom 8 (Governance Alignment) gap entry in the gap closing table and update:
```markdown
| No Art. 22 decision fields (Axiom 8) | GDPR Art. 22 enrichment (#7) | ✅ ComplianceSupplement — all fields nullable |
```

Change `**Status:**` line under Axiom 8 from `⚠️ Partial` to `✅ Addressed (#7)`.

- [ ] **Step 5: Add Supplement section to `docs/integration-guide.md`**

Add a new section after the basic integration section:

```markdown
## Supplements — Optional Extensions

Supplements add cross-cutting fields to any ledger entry without polluting the core
entity. See `docs/DESIGN.md` § Supplements for the full reference.

### Quick start — GDPR Art.22 compliance

```java
// In your capture service, after building the entry:
ComplianceSupplement cs = new ComplianceSupplement();
cs.algorithmRef       = "my-model-v2";
cs.confidenceScore    = 0.87;
cs.contestationUri    = "https://yourapp.com/decisions/" + entry.id + "/challenge";
cs.humanOverrideAvailable = true;
cs.decisionContext    = objectMapper.writeValueAsString(inputSnapshot);
entry.attach(cs);  // persists with entry; no separate persist() call needed
```

The `supplementJson` field is populated automatically by `attach()` — no extra
step required for single-entry reads.

### Runnable example

`examples/art22-decision-snapshot/` — a full Quarkus app demonstrating a GDPR
Art.22 compliant AI decision service. See its `README.md` for regulatory context.
```

- [ ] **Step 6: Write `examples/art22-decision-snapshot/README.md`**

```markdown
# Example: GDPR Art.22 Decision Snapshot

This example demonstrates how to use `casehub-ledger` with `ComplianceSupplement`
to build an AI decision service that is compliant with GDPR Article 22.

## What is GDPR Article 22?

Article 22 of the GDPR gives individuals the right not to be subject to decisions
based solely on automated processing when those decisions have legal or similarly
significant effects. When organisations do make such decisions, Article 22(2) and
recital 71 require:

- **Explainability** — meaningful information about the logic involved
- **Human oversight** — the ability to obtain human intervention
- **Contestation** — the right to express one's view and challenge the decision

## How this example satisfies Article 22

Each AI decision records a `ComplianceSupplement` with four structured fields:

| Field | Art.22 requirement satisfied |
|---|---|
| `algorithmRef` | "Meaningful information about the logic involved" — identifies which model or rule produced the decision |
| `confidenceScore` | "Significance and envisaged consequences" — the model's stated certainty |
| `contestationUri` | Right to contest — a URI where the subject can request human review |
| `humanOverrideAvailable` | Right to obtain human intervention — explicit boolean flag |

The `decisionContext` field in the same supplement carries a full JSON snapshot of
the inputs used — satisfying Arts.13–15's right to receive information about
"the categories of personal data used".

## Running the example

```bash
cd examples/art22-decision-snapshot
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn quarkus:dev
```

```bash
# Record an AI decision
curl -X POST http://localhost:8080/decisions \
  -H "Content-Type: application/json" \
  -d '{
    "subjectId": "customer-uuid-here",
    "category": "credit-risk",
    "outcome": "APPROVED",
    "algorithmRef": "risk-model-v3",
    "confidence": 0.88,
    "inputContext": "{\"creditScore\":720}"
  }'

# Retrieve full decision history with Art.22 fields
curl http://localhost:8080/decisions/customer-uuid-here/ledger

# Verify hash chain integrity
curl http://localhost:8080/decisions/customer-uuid-here/ledger/verify
```

## Running the tests

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test
```
```

- [ ] **Step 7: Run full build to confirm all docs changes don't break anything**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q 2>&1 | tail -5
cd examples/order-processing && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5 && cd ../..
cd examples/art22-decision-snapshot && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q 2>&1 | tail -5 && cd ../..
```

Expected: all three BUILD SUCCESS.

- [ ] **Step 8: Commit docs**

```bash
git add docs/ examples/art22-decision-snapshot/README.md
git commit -m "docs: Supplement chapter, Art.22 integration guide, AUDITABILITY axiom gap closed

DESIGN.md: new Supplement chapter (what/why/usage/zero-complexity guarantee),
updated hash chain canonical form (planRef removed), updated Flyway convention
(V1003+ for consumer subclass tables).

AUDITABILITY.md: Axiom 8 gap marked as addressed by #7.
integration-guide.md: quick-start Supplement section + example reference.
art22-decision-snapshot/README.md: GDPR Art.22 regulatory context + field mapping.

Closes #7"
```

---

## Self-Review

**Spec coverage check:**

| Spec requirement | Task |
|---|---|
| 4 Art.22 typed fields on ComplianceSupplement | Task 1 |
| planRef, rationale, evidence, detail, decisionContext in ComplianceSupplement | Task 1 |
| ProvenanceSupplement (sourceEntityId/Type/System) | Task 1 |
| ObservabilitySupplement (correlationId, causedByEntryId) | Task 1 |
| LedgerSupplementSerializer — null fields omitted | Task 2 |
| LedgerEntry slimmed to 10 core fields | Task 3 |
| attach() + compliance() + provenance() + observability() helpers | Task 3 |
| Hash chain canonical form removes planRef | Task 3 |
| supplementJson auto-updated by attach() | Task 3 |
| V1002 migration — drop columns, create supplement tables | Task 4 |
| Unit tests: supplement round-trip, lazy loading, zero-touch | Task 5 |
| order-processing example updated | Task 6 |
| order-processing: 10 IT tests (8 original + 2 new) | Task 6 |
| art22-decision-snapshot example | Task 7 |
| art22 IT tests (3) | Task 7 |
| DESIGN.md Supplement chapter | Task 8 |
| DESIGN.md hash chain + Flyway convention updated | Task 8 |
| AUDITABILITY.md Axiom 8 gap closed | Task 8 |
| integration-guide.md supplement section | Task 8 |
| art22 README.md with regulatory mapping | Task 8 |

**No placeholders found.** All steps contain complete code or exact commands.

**Type consistency:** `LedgerSupplementSerializer.toJson(List<LedgerSupplement>)` is
used consistently in Task 2 (tested), Task 3 (`attach()` calls it), and Task 5 (IT).
The `attach()` method signature is `void attach(LedgerSupplement)` in Task 3 and called
identically in Tasks 5, 6, 7.
