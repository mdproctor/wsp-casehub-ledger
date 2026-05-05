# W3C PROV-DM JSON-LD Export Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Export any subject's complete audit trail as a W3C PROV-DM JSON-LD document for interoperability with ML pipeline auditing tools and regulators.

**Architecture:** `LedgerProvSerializer` (pure static, like `LedgerMerkleTree`) builds JSON-LD from a list of entries using Jackson's `ObjectMapper`. `LedgerProvExportService` (CDI `@ApplicationScoped`) fetches entries by subjectId and delegates. `docs/prov-dm-mapping.md` documents the full field mapping. `examples/prov-dm-export/` demonstrates all supplement types.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson (already on classpath), JUnit 5, AssertJ. All tests: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`.

**Spec:** `docs/superpowers/specs/2026-04-18-prov-dm-export-design.md`

---

## File Map

**Created:**
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvSerializer.java`
- `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvExportService.java`
- `runtime/src/test/java/io/casehub/ledger/service/LedgerProvSerializerTest.java`
- `runtime/src/test/java/io/casehub/ledger/service/LedgerProvExportServiceIT.java`
- `docs/prov-dm-mapping.md`
- `examples/prov-dm-export/pom.xml`
- `examples/prov-dm-export/src/main/java/io/casehub/ledger/example/prov/ProvAuditEntry.java`
- `examples/prov-dm-export/src/main/java/io/casehub/ledger/example/prov/ProvAuditEntryRepository.java`
- `examples/prov-dm-export/src/main/java/io/casehub/ledger/example/prov/ProvDmExportExample.java`
- `examples/prov-dm-export/src/main/resources/application.properties`
- `examples/prov-dm-export/src/main/resources/META-INF/beans.xml`
- `examples/prov-dm-export/src/main/resources/db/migration/V2001__prov_example_entry.sql`
- `examples/prov-dm-export/src/test/java/io/casehub/ledger/example/prov/ProvDmExportIT.java`
- `examples/prov-dm-export/README.md`

---

## Task 1: Issue and Epic Setup

**Files:** GitHub issues only.

- [ ] **Step 1: Create the epic**

```bash
gh issue create --repo casehubio/ledger \
  --title "Epic: W3C PROV-DM JSON-LD export" \
  --label "enhancement" \
  --body "$(cat <<'EOF'
Implements RESEARCH.md item #7 — W3C PROV-DM JSON-LD serialiser for external audit interoperability.

## Child Issues
- [ ] TBD — created next step

## Acceptance Criteria
- LedgerProvSerializer.toProvJsonLd(subjectId, entries) produces valid PROV-JSON-LD
- LedgerProvExportService.exportSubject(UUID) CDI bean auto-activated
- All supplement types mapped and documented
- examples/prov-dm-export/ demonstrates full mapping
- docs/prov-dm-mapping.md field-by-field reference
EOF
)"
```

Note the epic number (e.g. #13). 

- [ ] **Step 2: Create the implementation issue**

```bash
gh issue create --repo casehubio/ledger \
  --title "feat: W3C PROV-DM JSON-LD export — LedgerProvSerializer + LedgerProvExportService" \
  --label "enhancement" \
  --body "$(cat <<'EOF'
Implement PROV-DM JSON-LD export per spec: docs/superpowers/specs/2026-04-18-prov-dm-export-design.md

## Acceptance Criteria
- [ ] LedgerProvSerializer.toProvJsonLd() produces valid PROV-JSON-LD
- [ ] All PROV-DM relations: wasGeneratedBy, wasAssociatedWith, wasDerivedFrom, hadPrimarySource
- [ ] ComplianceSupplement + ProvenanceSupplement fully mapped
- [ ] LedgerProvExportService CDI bean auto-activated
- [ ] Unit tests + IT tests + example passing
- [ ] docs/prov-dm-mapping.md written
EOF
)"
```

Note the implementation issue number (e.g. #14). All commits: `Refs #14, Refs #13`.

---

## Task 2: TDD — Write Failing LedgerProvSerializer Tests + Stub

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvSerializer.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerProvSerializerTest.java`

- [ ] **Step 1: Create stub LedgerProvSerializer**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;
import java.util.UUID;

import io.casehub.ledger.runtime.model.LedgerEntry;

public final class LedgerProvSerializer {
    private LedgerProvSerializer() {}

    public static String toProvJsonLd(UUID subjectId, List<LedgerEntry> entries) {
        throw new UnsupportedOperationException();
    }
}
```

- [ ] **Step 2: Create LedgerProvSerializerTest**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.UUID;

import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;
import io.casehub.ledger.runtime.service.LedgerProvSerializer;
import io.casehub.ledger.service.supplement.TestEntry;

class LedgerProvSerializerTest {

    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final Instant FIXED = Instant.parse("2026-04-18T10:00:00Z");

    private TestEntry entry(UUID subjectId, int seq, String actorId) {
        TestEntry e = new TestEntry();
        e.id = UUID.randomUUID();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.AGENT;
        e.actorRole = "Tester";
        e.occurredAt = FIXED;
        return e;
    }

    private Map<String, Object> parse(String json) throws Exception {
        return MAPPER.readValue(json, new TypeReference<>() {});
    }

    // ── @context ─────────────────────────────────────────────────────────────

    @Test
    void output_hasContextWithThreeKeys() throws Exception {
        UUID sub = UUID.randomUUID();
        String json = LedgerProvSerializer.toProvJsonLd(sub, List.of(entry(sub, 1, "a1")));
        Map<String, Object> doc = parse(json);
        Map<?, ?> ctx = (Map<?, ?>) doc.get("@context");
        assertThat(ctx).containsOnlyKeys("prov", "ledger", "xsd");
        assertThat(ctx.get("prov")).isEqualTo("http://www.w3.org/ns/prov#");
        assertThat(ctx.get("ledger")).isEqualTo("https://casehubio.github.io/ledger#");
        assertThat(ctx.get("xsd")).isEqualTo("http://www.w3.org/2001/XMLSchema#");
    }

    // ── entity ───────────────────────────────────────────────────────────────

    @Test
    void singleEntry_producesOneEntity() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "actor-a");
        String json = LedgerProvSerializer.toProvJsonLd(sub, List.of(e));
        Map<String, Object> doc = parse(json);
        Map<?, ?> entities = (Map<?, ?>) doc.get("entity");
        assertThat(entities).hasSize(1);
        assertThat(entities).containsKey("ledger:entry/" + e.id);
    }

    @Test
    void entity_hasRequiredProvDmFields() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "actor-a");
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        Map<?, ?> entity = (Map<?, ?>) ((Map<?, ?>) doc.get("entity")).get("ledger:entry/" + e.id);
        assertThat(entity.get("prov:type")).isEqualTo("ledger:LedgerEntry");
        assertThat(entity.get("ledger:subjectId")).isEqualTo(sub.toString());
        assertThat(entity.get("ledger:sequenceNumber")).isEqualTo(1);
        assertThat(entity.get("ledger:entryType")).isEqualTo("EVENT");
        assertThat(entity).containsKey("prov:generatedAtTime");
    }

    @Test
    void entity_nullFieldsOmitted() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "actor-a");
        e.correlationId = null;
        e.digest = null;
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        Map<?, ?> entity = (Map<?, ?>) ((Map<?, ?>) doc.get("entity")).get("ledger:entry/" + e.id);
        assertThat(entity).doesNotContainKey("ledger:correlationId");
        assertThat(entity).doesNotContainKey("ledger:digest");
    }

    // ── agent ─────────────────────────────────────────────────────────────────

    @Test
    void singleEntry_producesOneAgent() throws Exception {
        UUID sub = UUID.randomUUID();
        String json = LedgerProvSerializer.toProvJsonLd(sub, List.of(entry(sub, 1, "agent-007")));
        Map<?, ?> agents = (Map<?, ?>) parse(json).get("agent");
        assertThat(agents).hasSize(1);
        assertThat(agents).containsKey("ledger:actor/agent-007");
    }

    @Test
    void sameActorId_acrossEntries_deduplicatedToOneAgent() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e1 = entry(sub, 1, "shared-actor");
        TestEntry e2 = entry(sub, 2, "shared-actor");
        String json = LedgerProvSerializer.toProvJsonLd(sub, List.of(e1, e2));
        Map<?, ?> agents = (Map<?, ?>) parse(json).get("agent");
        assertThat(agents).hasSize(1);
    }

    @Test
    void nullActorId_noAgentNode_noWasAssociatedWith() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, null);
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        assertThat(doc).doesNotContainKey("agent");
        assertThat(doc).doesNotContainKey("wasAssociatedWith");
    }

    // ── relations ─────────────────────────────────────────────────────────────

    @Test
    void wasGeneratedBy_alwaysPresent() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "a");
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        Map<?, ?> wgb = (Map<?, ?>) doc.get("wasGeneratedBy");
        assertThat(wgb).hasSize(1);
        Map<?, ?> rel = (Map<?, ?>) wgb.values().iterator().next();
        assertThat(rel.get("prov:entity")).isEqualTo("ledger:entry/" + e.id);
        assertThat(rel.get("prov:activity")).isEqualTo("ledger:activity/" + e.id);
    }

    @Test
    void twoEntries_wasDerivedFrom_sequentialChain() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e1 = entry(sub, 1, "a");
        TestEntry e2 = entry(sub, 2, "a");
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e1, e2)));
        Map<?, ?> wdf = (Map<?, ?>) doc.get("wasDerivedFrom");
        assertThat(wdf).isNotEmpty();
        boolean hasChain = wdf.values().stream().anyMatch(v -> {
            Map<?, ?> rel = (Map<?, ?>) v;
            return rel.get("prov:generatedEntity").equals("ledger:entry/" + e2.id)
                && rel.get("prov:usedEntity").equals("ledger:entry/" + e1.id);
        });
        assertThat(hasChain).isTrue();
    }

    @Test
    void causedByEntryId_producesWasDerivedFrom() throws Exception {
        UUID sub = UUID.randomUUID();
        UUID externalEntryId = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "a");
        e.causedByEntryId = externalEntryId;
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        Map<?, ?> wdf = (Map<?, ?>) doc.get("wasDerivedFrom");
        assertThat(wdf).isNotEmpty();
        boolean hasCausal = wdf.values().stream().anyMatch(v -> {
            Map<?, ?> rel = (Map<?, ?>) v;
            return rel.get("prov:generatedEntity").equals("ledger:entry/" + e.id)
                && rel.get("prov:usedEntity").equals("ledger:entry/" + externalEntryId);
        });
        assertThat(hasCausal).isTrue();
    }

    // ── ComplianceSupplement ──────────────────────────────────────────────────

    @Test
    void complianceSupplement_allFieldsMappedToEntity() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "a");
        ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "gpt-4o";
        cs.confidenceScore = 0.92;
        cs.contestationUri = "https://example.com/challenge";
        cs.humanOverrideAvailable = true;
        cs.planRef = "policy-v9";
        cs.rationale = "threshold exceeded";
        cs.evidence = "{\"score\":0.92}";
        cs.decisionContext = "{\"input\":\"test\"}";
        e.attach(cs);
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        Map<?, ?> entity = (Map<?, ?>) ((Map<?, ?>) doc.get("entity")).get("ledger:entry/" + e.id);
        assertThat(entity.get("ledger:algorithmRef")).isEqualTo("gpt-4o");
        assertThat(entity.get("ledger:confidenceScore")).isEqualTo(0.92);
        assertThat(entity.get("ledger:contestationUri")).isEqualTo("https://example.com/challenge");
        assertThat(entity.get("ledger:humanOverrideAvailable")).isEqualTo(true);
        assertThat(entity.get("ledger:planRef")).isEqualTo("policy-v9");
        assertThat(entity.get("ledger:rationale")).isEqualTo("threshold exceeded");
        assertThat(entity.get("ledger:evidence")).isEqualTo("{\"score\":0.92}");
        assertThat(entity.get("ledger:decisionContext")).isEqualTo("{\"input\":\"test\"}");
    }

    @Test
    void complianceSupplement_nullFieldsOmitted() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "a");
        ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "gpt-4o";
        // all other fields null
        e.attach(cs);
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        Map<?, ?> entity = (Map<?, ?>) ((Map<?, ?>) doc.get("entity")).get("ledger:entry/" + e.id);
        assertThat(entity.get("ledger:algorithmRef")).isEqualTo("gpt-4o");
        assertThat(entity).doesNotContainKey("ledger:confidenceScore");
        assertThat(entity).doesNotContainKey("ledger:planRef");
    }

    // ── ProvenanceSupplement ──────────────────────────────────────────────────

    @Test
    void provenanceSupplement_producesHadPrimarySource() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = entry(sub, 1, "a");
        ProvenanceSupplement ps = new ProvenanceSupplement();
        ps.sourceEntityId = "wi-uuid-123";
        ps.sourceEntityType = "WorkItem";
        ps.sourceEntitySystem = "tarkus";
        e.attach(ps);
        Map<String, Object> doc = parse(LedgerProvSerializer.toProvJsonLd(sub, List.of(e)));
        Map<?, ?> hps = (Map<?, ?>) doc.get("hadPrimarySource");
        assertThat(hps).isNotEmpty();
        Map<?, ?> rel = (Map<?, ?>) hps.values().iterator().next();
        assertThat(rel.get("prov:entity")).isEqualTo("ledger:entry/" + e.id);
        assertThat(rel.get("prov:hadPrimarySource"))
            .isEqualTo("ledger:external/WorkItem/tarkus/wi-uuid-123");
    }
}
```

- [ ] **Step 3: Compile test sources**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime -q
```

Expected: BUILD SUCCESS (stub compiles, tests not yet run).

- [ ] **Step 4: Run tests — verify they fail**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerProvSerializerTest 2>&1 | tail -5
```

Expected: BUILD FAILURE — `UnsupportedOperationException`.

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvSerializer.java
git add runtime/src/test/java/io/casehub/ledger/service/LedgerProvSerializerTest.java
git commit -m "test(prov): add LedgerProvSerializerTest (TDD — all failing) + stub

Refs #14, Refs #13"
```

---

## Task 3: Implement LedgerProvSerializer

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvSerializer.java`

- [ ] **Step 1: Write the full implementation**

```java
package io.casehub.ledger.runtime.service;

import java.time.temporal.ChronoUnit;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.runtime.model.LedgerEntry;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;

/**
 * Serialises a subject's ledger history as a W3C PROV-DM JSON-LD document.
 *
 * <p>Mapping: {@code LedgerEntry} → {@code prov:Entity}, {@code actorId} → {@code prov:Agent}
 * (deduplicated), entry action → {@code prov:Activity}. See {@code docs/prov-dm-mapping.md}
 * for the full field-by-field reference.
 *
 * <p>Pure static utility — no CDI, no DB access. Call from within a {@code @Transactional}
 * context so supplement lazy-loading succeeds.
 */
public final class LedgerProvSerializer {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private static final Map<String, Object> CONTEXT = Map.of(
            "prov", "http://www.w3.org/ns/prov#",
            "ledger", "https://casehubio.github.io/ledger#",
            "xsd", "http://www.w3.org/2001/XMLSchema#");

    private LedgerProvSerializer() {}

    /**
     * Serialise the entries for a subject as a PROV-JSON-LD document.
     *
     * @param subjectId the aggregate identifier scoping this bundle
     * @param entries   ordered list (ascending sequenceNumber) for this subject
     * @return pretty-printed PROV-JSON-LD string
     */
    public static String toProvJsonLd(final UUID subjectId, final List<LedgerEntry> entries) {
        final Map<String, Object> doc = new LinkedHashMap<>();
        doc.put("@context", CONTEXT);

        final Map<String, Object> entities = new LinkedHashMap<>();
        final Map<String, Object> agents = new LinkedHashMap<>();
        final Map<String, Object> activities = new LinkedHashMap<>();
        final Map<String, Object> wasGeneratedBy = new LinkedHashMap<>();
        final Map<String, Object> wasAssociatedWith = new LinkedHashMap<>();
        final Map<String, Object> wasDerivedFrom = new LinkedHashMap<>();
        final Map<String, Object> hadPrimarySource = new LinkedHashMap<>();

        // Index by sequenceNumber for O(1) predecessor lookup
        final Map<Integer, LedgerEntry> bySeq = entries.stream()
                .collect(Collectors.toMap(e -> e.sequenceNumber, e -> e));

        for (final LedgerEntry entry : entries) {
            final String entryIri = entryIri(entry.id);
            final String activityIri = activityIri(entry.id);

            // ── Entity ───────────────────────────────────────────────────────
            final Map<String, Object> entity = new LinkedHashMap<>();
            entity.put("prov:type", "ledger:LedgerEntry");
            entity.put("ledger:subjectId", subjectId.toString());
            entity.put("ledger:sequenceNumber", entry.sequenceNumber);
            entity.put("ledger:entryType", entry.entryType != null ? entry.entryType.name() : null);
            putIfNotNull(entity, "ledger:digest", entry.digest);
            entity.put("prov:generatedAtTime", formatInstant(entry.occurredAt));
            putIfNotNull(entity, "ledger:correlationId", entry.correlationId);

            // ComplianceSupplement fields
            entry.compliance().ifPresent(cs -> {
                putIfNotNull(entity, "ledger:algorithmRef", cs.algorithmRef);
                putIfNotNull(entity, "ledger:confidenceScore", cs.confidenceScore);
                putIfNotNull(entity, "ledger:contestationUri", cs.contestationUri);
                putIfNotNull(entity, "ledger:humanOverrideAvailable", cs.humanOverrideAvailable);
                putIfNotNull(entity, "ledger:planRef", cs.planRef);
                putIfNotNull(entity, "ledger:rationale", cs.rationale);
                putIfNotNull(entity, "ledger:evidence", cs.evidence);
                putIfNotNull(entity, "ledger:decisionContext", cs.decisionContext);
            });
            entities.put(entryIri, entity);

            // ── Activity ─────────────────────────────────────────────────────
            final Map<String, Object> activity = new LinkedHashMap<>();
            activity.put("prov:type", "ledger:Activity");
            activity.put("ledger:entryType", entry.entryType != null ? entry.entryType.name() : null);
            activity.put("prov:startedAtTime", formatInstant(entry.occurredAt));
            activities.put(activityIri, activity);

            // ── Agent (deduplicated) ─────────────────────────────────────────
            if (entry.actorId != null && !agents.containsKey(agentIri(entry.actorId))) {
                final Map<String, Object> agent = new LinkedHashMap<>();
                agent.put("prov:type", "ledger:Actor");
                putIfNotNull(agent, "ledger:actorType",
                        entry.actorType != null ? entry.actorType.name() : null);
                putIfNotNull(agent, "ledger:actorRole", entry.actorRole);
                agents.put(agentIri(entry.actorId), agent);
            }

            // ── wasGeneratedBy ───────────────────────────────────────────────
            final Map<String, Object> wgb = new LinkedHashMap<>();
            wgb.put("prov:entity", entryIri);
            wgb.put("prov:activity", activityIri);
            wasGeneratedBy.put("_:wgb-" + entry.id, wgb);

            // ── wasAssociatedWith ────────────────────────────────────────────
            if (entry.actorId != null) {
                final Map<String, Object> waw = new LinkedHashMap<>();
                waw.put("prov:activity", activityIri);
                waw.put("prov:agent", agentIri(entry.actorId));
                wasAssociatedWith.put("_:waw-" + entry.id, waw);
            }

            // ── wasDerivedFrom — sequential chain ────────────────────────────
            if (entry.sequenceNumber > 1) {
                final LedgerEntry prev = bySeq.get(entry.sequenceNumber - 1);
                if (prev != null) {
                    final Map<String, Object> wdf = new LinkedHashMap<>();
                    wdf.put("prov:generatedEntity", entryIri);
                    wdf.put("prov:usedEntity", entryIri(prev.id));
                    wasDerivedFrom.put("_:wdf-" + entry.id + "-" + prev.id, wdf);
                }
            }

            // ── wasDerivedFrom — cross-subject causality ─────────────────────
            if (entry.causedByEntryId != null) {
                final Map<String, Object> wdf = new LinkedHashMap<>();
                wdf.put("prov:generatedEntity", entryIri);
                wdf.put("prov:usedEntity", entryIri(entry.causedByEntryId));
                wasDerivedFrom.put("_:wdf-caused-" + entry.id + "-" + entry.causedByEntryId, wdf);
            }

            // ── hadPrimarySource — ProvenanceSupplement ──────────────────────
            entry.provenance().ifPresent(ps -> {
                if (ps.sourceEntityId != null) {
                    final Map<String, Object> hps = new LinkedHashMap<>();
                    hps.put("prov:entity", entryIri);
                    hps.put("prov:hadPrimarySource",
                            externalIri(ps.sourceEntityType, ps.sourceEntitySystem, ps.sourceEntityId));
                    hadPrimarySource.put("_:hps-" + entry.id, hps);
                }
            });
        }

        doc.put("entity", entities);
        if (!agents.isEmpty()) doc.put("agent", agents);
        doc.put("activity", activities);
        doc.put("wasGeneratedBy", wasGeneratedBy);
        if (!wasAssociatedWith.isEmpty()) doc.put("wasAssociatedWith", wasAssociatedWith);
        if (!wasDerivedFrom.isEmpty()) doc.put("wasDerivedFrom", wasDerivedFrom);
        if (!hadPrimarySource.isEmpty()) doc.put("hadPrimarySource", hadPrimarySource);

        try {
            return MAPPER.writerWithDefaultPrettyPrinter().writeValueAsString(doc);
        } catch (final JsonProcessingException e) {
            throw new IllegalStateException("Failed to serialise PROV-JSON-LD", e);
        }
    }

    // ── IRI helpers ───────────────────────────────────────────────────────────

    private static String entryIri(final UUID id) {
        return "ledger:entry/" + id;
    }

    private static String agentIri(final String actorId) {
        return "ledger:actor/" + actorId;
    }

    private static String activityIri(final UUID id) {
        return "ledger:activity/" + id;
    }

    private static String externalIri(final String type, final String system, final String id) {
        return "ledger:external/" + type + "/" + system + "/" + id;
    }

    private static String formatInstant(final java.time.Instant instant) {
        if (instant == null) return null;
        return instant.truncatedTo(ChronoUnit.MILLIS).toString();
    }

    private static void putIfNotNull(final Map<String, Object> map,
            final String key, final Object value) {
        if (value != null) {
            map.put(key, value);
        }
    }
}
```

- [ ] **Step 2: Run LedgerProvSerializerTest**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerProvSerializerTest 2>&1 | tail -10
```

Expected: BUILD SUCCESS, all tests pass.

- [ ] **Step 3: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: BUILD SUCCESS, 110 existing tests still pass + new serialiser tests.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvSerializer.java
git commit -m "feat(prov): implement LedgerProvSerializer — PROV-JSON-LD serialiser

Maps LedgerEntry → prov:Entity, actorId → prov:Agent (deduplicated),
entry action → prov:Activity. Supplements mapped as entity properties.
wasDerivedFrom covers sequential chain + cross-subject causedByEntryId.

Refs #14, Refs #13"
```

---

## Task 4: TDD — Write Failing LedgerProvExportService IT + Stub

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvExportService.java`
- Create: `runtime/src/test/java/io/casehub/ledger/service/LedgerProvExportServiceIT.java`

- [ ] **Step 1: Create stub LedgerProvExportService**

```java
package io.casehub.ledger.runtime.service;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class LedgerProvExportService {

    /** Export the complete provenance graph for a subject as PROV-JSON-LD. */
    public String exportSubject(final UUID subjectId) {
        throw new UnsupportedOperationException();
    }
}
```

- [ ] **Step 2: Create LedgerProvExportServiceIT**

```java
package io.casehub.ledger.service;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerProvExportService;
import io.casehub.ledger.service.supplement.TestEntry;
import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class LedgerProvExportServiceIT {

    @Inject LedgerProvExportService exportService;
    @Inject LedgerEntryRepository repo;

    private static final ObjectMapper MAPPER = new ObjectMapper();

    private TestEntry seed(UUID subjectId, int seq, String actorId) {
        TestEntry e = new TestEntry();
        e.subjectId = subjectId;
        e.sequenceNumber = seq;
        e.entryType = LedgerEntryType.EVENT;
        e.actorId = actorId;
        e.actorType = ActorType.SYSTEM;
        e.actorRole = "Tester";
        return (TestEntry) repo.save(e);
    }

    // ── happy path ────────────────────────────────────────────────────────────

    @Test
    @Transactional
    void exportSubject_threeEntries_validJsonLd() throws Exception {
        UUID sub = UUID.randomUUID();
        seed(sub, 1, "actor-a");
        seed(sub, 2, "actor-a");
        seed(sub, 3, "actor-b");

        String json = exportService.exportSubject(sub);

        assertThat(json).isNotBlank();
        Map<String, Object> doc = MAPPER.readValue(json, new TypeReference<>() {});
        assertThat(doc).containsKey("@context");
        assertThat(doc).containsKey("entity");
        assertThat(doc).containsKey("agent");
        assertThat(doc).containsKey("activity");
        assertThat(doc).containsKey("wasGeneratedBy");
        Map<?, ?> entities = (Map<?, ?>) doc.get("entity");
        assertThat(entities).hasSize(3);
    }

    @Test
    @Transactional
    void exportSubject_complianceSupplement_fieldsInOutput() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = seed(sub, 1, "actor-a");
        ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "gpt-4o";
        cs.confidenceScore = 0.88;
        e.attach(cs);
        e.persist();

        String json = exportService.exportSubject(sub);

        Map<String, Object> doc = MAPPER.readValue(json, new TypeReference<>() {});
        Map<?, ?> entity = (Map<?, ?>) ((Map<?, ?>) doc.get("entity"))
                .get("ledger:entry/" + e.id);
        assertThat(entity.get("ledger:algorithmRef")).isEqualTo("gpt-4o");
        assertThat(entity.get("ledger:confidenceScore")).isEqualTo(0.88);
    }

    @Test
    @Transactional
    void exportSubject_provenanceSupplement_hadPrimarySourcePresent() throws Exception {
        UUID sub = UUID.randomUUID();
        TestEntry e = seed(sub, 1, "actor-a");
        ProvenanceSupplement ps = new ProvenanceSupplement();
        ps.sourceEntityId = "wi-abc";
        ps.sourceEntityType = "WorkItem";
        ps.sourceEntitySystem = "tarkus";
        e.attach(ps);
        e.persist();

        String json = exportService.exportSubject(sub);

        Map<String, Object> doc = MAPPER.readValue(json, new TypeReference<>() {});
        assertThat(doc).containsKey("hadPrimarySource");
    }

    // ── end to end ────────────────────────────────────────────────────────────

    @Test
    @Transactional
    void e2e_allRelationsPresent_forThreeEntrySubject() throws Exception {
        UUID sub = UUID.randomUUID();
        seed(sub, 1, "actor-a");
        seed(sub, 2, "actor-b");
        seed(sub, 3, "actor-a");

        String json = exportService.exportSubject(sub);

        Map<String, Object> doc = MAPPER.readValue(json, new TypeReference<>() {});
        // 2 agents (actor-a deduplicated), 3 activities, 2 sequential wasDerivedFrom
        Map<?, ?> agents = (Map<?, ?>) doc.get("agent");
        assertThat(agents).hasSize(2);
        Map<?, ?> wdf = (Map<?, ?>) doc.get("wasDerivedFrom");
        assertThat(wdf).hasSize(2); // seq 2→1, seq 3→2
    }
}
```

- [ ] **Step 3: Compile test sources**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test-compile -pl runtime -q
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit stub + tests**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvExportService.java
git add runtime/src/test/java/io/casehub/ledger/service/LedgerProvExportServiceIT.java
git commit -m "test(prov): add LedgerProvExportServiceIT (TDD — all failing) + stub bean

Refs #14, Refs #13"
```

---

## Task 5: Implement LedgerProvExportService

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvExportService.java`

- [ ] **Step 1: Write the implementation**

```java
package io.casehub.ledger.runtime.service;

import java.util.List;
import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.LedgerEntry;

/**
 * CDI bean exporting a subject's audit history as W3C PROV-DM JSON-LD.
 *
 * <p>Auto-activated — no consumer configuration required. Consumers may call
 * {@link #exportSubject(UUID)} directly or expose it via their own REST endpoint.
 *
 * <p>See {@code docs/prov-dm-mapping.md} for the full field-by-field mapping.
 */
@ApplicationScoped
public class LedgerProvExportService {

    /**
     * Export the complete provenance graph for the given subject as PROV-JSON-LD.
     *
     * <p>Fetches all entries ordered by sequence number. Supplement lazy-loading
     * succeeds within this transaction boundary.
     *
     * @param subjectId the aggregate identifier
     * @return pretty-printed PROV-JSON-LD string
     * @throws IllegalArgumentException if no entries exist for the subject
     */
    @Transactional
    public String exportSubject(final UUID subjectId) {
        final List<LedgerEntry> entries = LedgerEntry.<LedgerEntry>list(
                "subjectId = ?1 ORDER BY sequenceNumber ASC", subjectId);
        if (entries.isEmpty()) {
            throw new IllegalArgumentException("No entries found for subject: " + subjectId);
        }
        // Access supplements within the transaction to trigger lazy loading
        entries.forEach(e -> e.supplements.size());
        return LedgerProvSerializer.toProvJsonLd(subjectId, entries);
    }
}
```

- [ ] **Step 2: Run the IT tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime \
  -Dtest=LedgerProvExportServiceIT 2>&1 | tail -10
```

Expected: BUILD SUCCESS, all IT tests pass.

- [ ] **Step 3: Run full suite**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime 2>&1 | grep -E "Tests run:|BUILD"
```

Expected: BUILD SUCCESS.

- [ ] **Step 4: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/LedgerProvExportService.java
git commit -m "feat(prov): implement LedgerProvExportService — CDI export bean

Fetches entries by subject, initialises lazy supplements within @Transactional,
delegates to LedgerProvSerializer. Auto-activated, no consumer config required.

Refs #14, Refs #13"
```

---

## Task 6: Write docs/prov-dm-mapping.md

**Files:**
- Create: `docs/prov-dm-mapping.md`

- [ ] **Step 1: Write the mapping document**

```markdown
# W3C PROV-DM Mapping Reference

Documents how `casehub-ledger` fields map to W3C PROV-DM concepts in the JSON-LD
export produced by `LedgerProvExportService.exportSubject(UUID)`.

## PROV-DM Primer

PROV-DM has three core types:

| PROV type | Meaning | Ledger mapping |
|---|---|---|
| `prov:Entity` | A thing with a provenance story | Each `LedgerEntry` |
| `prov:Agent` | An actor who caused something | Each distinct `actorId` |
| `prov:Activity` | An action that occurred | One per `LedgerEntry` |

And three core relations:

| Relation | Meaning | Emitted when |
|---|---|---|
| `wasGeneratedBy` | An entity was produced by an activity | Every entry |
| `wasAssociatedWith` | An activity was performed by an agent | `actorId` is non-null |
| `wasDerivedFrom` | An entity derives from another entity | Sequential chain + `causedByEntryId` |

## IRI Conventions

All IRIs use the `ledger:` prefix (`https://casehubio.github.io/ledger#`):

| Element | IRI pattern | Example |
|---|---|---|
| Entry entity | `ledger:entry/<uuid>` | `ledger:entry/e1a2b3...` |
| Agent | `ledger:actor/<actorId>` | `ledger:actor/agent-007` |
| Activity | `ledger:activity/<uuid>` | `ledger:activity/e1a2b3...` |
| External source | `ledger:external/<type>/<system>/<id>` | `ledger:external/WorkItem/tarkus/wi-abc` |

Agents are **deduplicated** — the same `actorId` appearing across multiple entries
produces a single `agent` node in the document.

## Core LedgerEntry Fields

| `LedgerEntry` field | PROV-DM location | Property |
|---|---|---|
| `id` | Entity IRI | `ledger:entry/<id>` |
| `subjectId` | Entity property | `ledger:subjectId` |
| `sequenceNumber` | Entity property | `ledger:sequenceNumber` |
| `entryType` | Entity + Activity property | `ledger:entryType` |
| `digest` | Entity property | `ledger:digest` |
| `occurredAt` | Entity + Activity property | `prov:generatedAtTime`, `prov:startedAtTime` |
| `correlationId` | Entity property | `ledger:correlationId` |
| `actorId` | Agent IRI | `ledger:actor/<actorId>` |
| `actorType` | Agent property | `ledger:actorType` |
| `actorRole` | Agent property | `ledger:actorRole` |
| `causedByEntryId` | `wasDerivedFrom` relation | Cross-subject causal link |

## ComplianceSupplement Fields

Compliance fields appear as additional properties on the `prov:Entity` when a
`ComplianceSupplement` is attached to the entry. Null fields are omitted.

| `ComplianceSupplement` field | Entity property | Notes |
|---|---|---|
| `algorithmRef` | `ledger:algorithmRef` | Model or rule engine version (GDPR Art.22) |
| `confidenceScore` | `ledger:confidenceScore` | 0.0–1.0 producing system confidence |
| `contestationUri` | `ledger:contestationUri` | URI for challenging the decision |
| `humanOverrideAvailable` | `ledger:humanOverrideAvailable` | Boolean |
| `planRef` | `ledger:planRef` | Policy / procedure version reference |
| `rationale` | `ledger:rationale` | Actor's stated basis for the decision |
| `evidence` | `ledger:evidence` | Structured evidence (JSON string) |
| `decisionContext` | `ledger:decisionContext` | JSON snapshot of decision-time state |

## ProvenanceSupplement Fields

A `ProvenanceSupplement` produces a `hadPrimarySource` relation from the entry entity
to an external IRI constructed from the three source fields.

| `ProvenanceSupplement` field | Role in IRI |
|---|---|
| `sourceEntityType` | Type segment: `ledger:external/<type>/...` |
| `sourceEntitySystem` | System segment: `.../tarkus/...` |
| `sourceEntityId` | ID segment: `.../wi-abc` |

Example: `sourceEntityType=WorkItem`, `sourceEntitySystem=tarkus`, `sourceEntityId=wi-abc`
→ `ledger:external/WorkItem/tarkus/wi-abc`

## wasDerivedFrom Relations

Two situations produce a `wasDerivedFrom` edge:

1. **Sequential chain** — every entry with `sequenceNumber > 1` derives from its predecessor.
   Key: `_:wdf-<entry-uuid>-<prev-uuid>`

2. **Cross-subject causality** — when `causedByEntryId` is set, the entry derives from
   the causal predecessor (which may be in a different subject's history).
   Key: `_:wdf-caused-<entry-uuid>-<caused-by-uuid>`

## @context

The JSON-LD `@context` is always exactly:

```json
{
  "@context": {
    "prov": "http://www.w3.org/ns/prov#",
    "ledger": "https://casehubio.github.io/ledger#",
    "xsd": "http://www.w3.org/2001/XMLSchema#"
  }
}
```
```

- [ ] **Step 2: Commit**

```bash
git add docs/prov-dm-mapping.md
git commit -m "docs(prov): add prov-dm-mapping.md — field-by-field PROV-DM reference

Refs #14, Refs #13"
```

---

## Task 7: Create examples/prov-dm-export/

**Files:** All files under `examples/prov-dm-export/`.

- [ ] **Step 1: Copy pom.xml from merkle-verification and adapt**

```bash
cp examples/merkle-verification/pom.xml examples/prov-dm-export/pom.xml
```

Edit `examples/prov-dm-export/pom.xml`:
- `<artifactId>casehub-ledger-example-prov-dm-export</artifactId>`
- `<name>CaseHub Ledger :: Example :: PROV-DM Export</name>`

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p examples/prov-dm-export/src/main/java/io/casehub/ledger/example/prov
mkdir -p examples/prov-dm-export/src/main/resources/db/migration
mkdir -p examples/prov-dm-export/src/main/resources/META-INF
mkdir -p examples/prov-dm-export/src/test/java/io/casehub/ledger/example/prov
```

- [ ] **Step 3: Create ProvAuditEntry.java**

```java
package io.casehub.ledger.example.prov;

import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;

import io.casehub.ledger.runtime.model.LedgerEntry;

@Entity
@Table(name = "prov_example_entry")
@DiscriminatorValue("PROV_EXAMPLE")
public class ProvAuditEntry extends LedgerEntry {}
```

- [ ] **Step 4: Create ProvAuditEntryRepository.java**

```java
package io.casehub.ledger.example.prov;

import jakarta.enterprise.context.ApplicationScoped;

import io.casehub.ledger.runtime.repository.jpa.JpaLedgerEntryRepository;

@ApplicationScoped
public class ProvAuditEntryRepository extends JpaLedgerEntryRepository {}
```

- [ ] **Step 5: Create V2001__prov_example_entry.sql**

```sql
CREATE TABLE prov_example_entry (
    id UUID NOT NULL,
    CONSTRAINT pk_prov_example_entry PRIMARY KEY (id),
    CONSTRAINT fk_prov_example_entry FOREIGN KEY (id) REFERENCES ledger_entry (id)
);
```

- [ ] **Step 6: Copy application.properties from merkle-verification and adapt**

```bash
cp examples/merkle-verification/src/main/resources/application.properties \
   examples/prov-dm-export/src/main/resources/application.properties
```

Edit — change `quarkus.arc.selected-alternatives` to point to `ProvAuditEntryRepository`.

- [ ] **Step 7: Copy beans.xml**

```bash
cp examples/merkle-verification/src/main/resources/META-INF/beans.xml \
   examples/prov-dm-export/src/main/resources/META-INF/beans.xml
```

- [ ] **Step 8: Create ProvDmExportExample.java**

```java
package io.casehub.ledger.example.prov;

import java.util.UUID;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.transaction.Transactional;

import io.casehub.ledger.runtime.model.ActorType;
import io.casehub.ledger.runtime.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.supplement.ComplianceSupplement;
import io.casehub.ledger.runtime.model.supplement.ProvenanceSupplement;
import io.casehub.ledger.runtime.repository.LedgerEntryRepository;
import io.casehub.ledger.runtime.service.LedgerProvExportService;

/**
 * Demonstrates PROV-DM JSON-LD export covering all supplement types.
 *
 * <p>Happy path: 3 entries with ComplianceSupplement, ProvenanceSupplement,
 * and cross-subject causedByEntryId → export produces full PROV-JSON-LD document.
 */
@ApplicationScoped
public class ProvDmExportExample {

    @Inject LedgerEntryRepository repo;
    @Inject LedgerProvExportService exportService;

    @Transactional
    public String runHappyPath(final UUID subjectId) {
        // Entry 1: AI decision with ComplianceSupplement
        ProvAuditEntry e1 = new ProvAuditEntry();
        e1.subjectId = subjectId;
        e1.sequenceNumber = 1;
        e1.entryType = LedgerEntryType.COMMAND;
        e1.actorId = "classifier-agent-v2";
        e1.actorType = ActorType.AGENT;
        e1.actorRole = "Classifier";
        e1 = (ProvAuditEntry) repo.save(e1);
        ComplianceSupplement cs = new ComplianceSupplement();
        cs.algorithmRef = "gpt-4o";
        cs.confidenceScore = 0.94;
        cs.contestationUri = "https://example.com/challenge/" + subjectId;
        cs.humanOverrideAvailable = true;
        cs.planRef = "classification-policy-v3";
        cs.rationale = "Threshold exceeded; classified as high-priority";
        cs.decisionContext = "{\"inputScore\":0.94,\"threshold\":0.80}";
        e1.attach(cs);
        e1.persist();

        // Entry 2: event triggered by external WorkItem (ProvenanceSupplement)
        ProvAuditEntry e2 = new ProvAuditEntry();
        e2.subjectId = subjectId;
        e2.sequenceNumber = 2;
        e2.entryType = LedgerEntryType.EVENT;
        e2.actorId = "orchestrator-system";
        e2.actorType = ActorType.SYSTEM;
        e2.actorRole = "Orchestrator";
        e2 = (ProvAuditEntry) repo.save(e2);
        ProvenanceSupplement ps = new ProvenanceSupplement();
        ps.sourceEntityId = "wi-" + UUID.randomUUID();
        ps.sourceEntityType = "WorkItem";
        ps.sourceEntitySystem = "tarkus";
        e2.attach(ps);
        e2.persist();

        // Entry 3: caused by entry 1 in a cross-subject chain
        ProvAuditEntry e3 = new ProvAuditEntry();
        e3.subjectId = subjectId;
        e3.sequenceNumber = 3;
        e3.entryType = LedgerEntryType.EVENT;
        e3.actorId = "classifier-agent-v2";
        e3.actorType = ActorType.AGENT;
        e3.actorRole = "Classifier";
        e3.causedByEntryId = e1.id;
        repo.save(e3);

        return exportService.exportSubject(subjectId);
    }
}
```

- [ ] **Step 9: Create ProvDmExportIT.java**

```java
package io.casehub.ledger.example.prov;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;
import java.util.UUID;

import jakarta.inject.Inject;

import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;

import io.quarkus.test.junit.QuarkusTest;

@QuarkusTest
class ProvDmExportIT {

    @Inject ProvDmExportExample example;

    private static final ObjectMapper MAPPER = new ObjectMapper();

    @Test
    void happyPath_allSupplementTypes_validProvJsonLd() throws Exception {
        String json = example.runHappyPath(UUID.randomUUID());

        assertThat(json).isNotBlank();
        Map<String, Object> doc = MAPPER.readValue(json, new TypeReference<>() {});

        // @context always present with exactly 3 keys
        Map<?, ?> ctx = (Map<?, ?>) doc.get("@context");
        assertThat(ctx).containsOnlyKeys("prov", "ledger", "xsd");

        // 3 entities (one per entry)
        Map<?, ?> entities = (Map<?, ?>) doc.get("entity");
        assertThat(entities).hasSize(3);

        // 2 agents — classifier-agent-v2 (entries 1+3, deduplicated) + orchestrator-system
        Map<?, ?> agents = (Map<?, ?>) doc.get("agent");
        assertThat(agents).hasSize(2);
        assertThat(agents).containsKey("ledger:actor/classifier-agent-v2");
        assertThat(agents).containsKey("ledger:actor/orchestrator-system");

        // 3 activities
        assertThat((Map<?, ?>) doc.get("activity")).hasSize(3);

        // 3 wasGeneratedBy (one per entry)
        assertThat((Map<?, ?>) doc.get("wasGeneratedBy")).hasSize(3);

        // 3 wasAssociatedWith (all have actorId)
        assertThat((Map<?, ?>) doc.get("wasAssociatedWith")).hasSize(3);

        // wasDerivedFrom: 2 sequential (2→1, 3→2) + 1 causal (3→1) = 3 total
        assertThat((Map<?, ?>) doc.get("wasDerivedFrom")).hasSize(3);

        // hadPrimarySource: from entry 2's ProvenanceSupplement
        assertThat(doc).containsKey("hadPrimarySource");

        // ComplianceSupplement fields on entry 1
        Map<?, ?> entry1 = entities.values().stream()
                .map(v -> (Map<?, ?>) v)
                .filter(e -> "gpt-4o".equals(e.get("ledger:algorithmRef")))
                .findFirst().orElseThrow(() -> new AssertionError("Entry with algorithmRef not found"));
        assertThat(entry1.get("ledger:confidenceScore")).isEqualTo(0.94);
        assertThat(entry1.get("ledger:humanOverrideAvailable")).isEqualTo(true);
        assertThat(entry1.get("ledger:planRef")).isEqualTo("classification-policy-v3");
    }

    @Test
    void happyPath_contextKeys_alwaysExactlyProvLedgerXsd() throws Exception {
        String json = example.runHappyPath(UUID.randomUUID());
        Map<String, Object> doc = MAPPER.readValue(json, new TypeReference<>() {});
        Map<?, ?> ctx = (Map<?, ?>) doc.get("@context");
        assertThat(ctx.get("prov")).isEqualTo("http://www.w3.org/ns/prov#");
        assertThat(ctx.get("ledger")).isEqualTo("https://casehubio.github.io/ledger#");
        assertThat(ctx.get("xsd")).isEqualTo("http://www.w3.org/2001/XMLSchema#");
    }
}
```

- [ ] **Step 10: Create README.md**

```markdown
# Example: PROV-DM Export

Demonstrates W3C PROV-DM JSON-LD export using `casehub-ledger`.

## What This Shows

- All three supplement types (ComplianceSupplement, ProvenanceSupplement)
- Cross-subject causal chain (`causedByEntryId`)
- Agent deduplication across entries
- Sequential `wasDerivedFrom` chain + causal `wasDerivedFrom` edge
- `hadPrimarySource` from ProvenanceSupplement

## Run

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/prov-dm-export
```

## Field Mapping Reference

See `docs/prov-dm-mapping.md` for the full field-by-field PROV-DM mapping.
```

- [ ] **Step 11: Build and test the example**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) \
  mvn -f examples/prov-dm-export/pom.xml test 2>&1 | tail -10
```

Expected: BUILD SUCCESS, 2 tests pass.

If `@Alternative` activation fails, check `application.properties`:
```
quarkus.arc.selected-alternatives=io.casehub.ledger.example.prov.ProvAuditEntryRepository
```

- [ ] **Step 12: Close issues and commit**

```bash
git add examples/prov-dm-export/
git commit -m "feat(prov): add prov-dm-export example — all supplement types + e2e assertions

Refs #14, Refs #13"
```

```bash
gh issue close 14 --repo casehubio/ledger \
  --comment "Implemented: LedgerProvSerializer, LedgerProvExportService, prov-dm-mapping.md, example."
gh issue close 13 --repo casehubio/ledger \
  --comment "All child issues complete."
```

---

## Final Verification

- [ ] **Run complete build**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install 2>&1 | \
  grep -E "^(\[INFO\] Tests run|\[ERROR\] Tests run|\[INFO\] BUILD|\[ERROR\] BUILD)"
```

Expected: BUILD SUCCESS across all modules. Runtime: 110 + new serialiser + IT tests. Example: 2 tests.
