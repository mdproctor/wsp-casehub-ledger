# Unified API Generation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/ledger#207 — unified API generation — migrate to @McpDomain SPI
**Issue group:** casehubio/ledger#207

**Goal:** Replace hand-written REST endpoints and GraphQL resolvers with generated code from the platform `graphql-generator` APT via @McpDomain SPI interfaces.

**Architecture:** Four SPI interfaces in `api/` define the ledger's public API using platform annotations (`@PlatformQuery`, `@PlatformMutation`, `@PathParam`). Four `@DefaultBean` service implementations in `runtime/` delegate to existing repository/service beans. The `graphql-generator` APT runs in `rest/` (REST only) and `graphql/` (GraphQL only), generating JAX-RS resources and SmallRye GraphQL resolvers from the SPI interfaces.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-platform-api 0.2-SNAPSHOT (provides `@McpDomain`, `@PlatformQuery`, `@PlatformMutation`, `@PathParam`, `@RestMethod`, `HttpMethod`), casehub-platform-graphql-generator 0.2-SNAPSHOT (APT)

## Global Constraints

- All SPI annotations from `io.casehub.platform.api.mcp` — never JAX-RS on SPI interfaces
- `tenancyId` is always a separate method parameter, never inside request records
- Generated REST paths use kebab-cased method names (no `@RestPath` annotation exists yet)
- Domain filtering required: `-AdomainFilter=ledger/entries,ledger/attestations,ledger/verification,ledger/trust`
- `rest/` and `graphql/` depend on `casehub-ledger-api` only (compile scope); `casehub-ledger` moves to test scope
- IntelliJ MCP required for all .java file operations — no bash grep/Edit on source files
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
- Test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`

---

## Batch 1: API types — SPI interfaces and view records

### Task 1: Create view/request records and SPI interfaces in api/

**Files:**
- Create: `api/src/main/java/io/casehub/ledger/api/view/LedgerEntryView.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/LedgerEntryPage.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/AttestationView.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/VerificationView.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/InclusionProofView.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/TrustScoreView.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/CapabilityScoreView.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/TrustRoutingProfileView.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/AppendEntryRequest.java`
- Create: `api/src/main/java/io/casehub/ledger/api/view/CreateAttestationRequest.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryApi.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/LedgerAttestationApi.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/LedgerVerificationApi.java`
- Create: `api/src/main/java/io/casehub/ledger/api/spi/LedgerTrustApi.java`
- Test: `api/src/test/java/io/casehub/ledger/api/view/ViewRecordTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.mcp.McpDomain`, `PlatformQuery`, `PlatformMutation`, `PathParam` (from casehub-platform-api)
- Produces: SPI interfaces and view records used by Batch 2 (service impls) and Batch 3 (APT generation)

- [ ] **Step 1: Write a test that verifies the view records exist and have expected fields**

```java
package io.casehub.ledger.api.view;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import java.util.OptionalDouble;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;

class ViewRecordTest {

    @Test
    void ledgerEntryViewHasAllFields() {
        var view = new LedgerEntryView(
                UUID.randomUUID(), UUID.randomUUID(), "tenant-1", 1,
                "EVENT", "actor-1", "AGENT", "reviewer",
                Instant.now(), "abc123", "trace-1",
                UUID.randomUUID(), "{}", Map.of("key", "value"));
        assertNotNull(view.id());
        assertEquals("EVENT", view.entryType());
        assertEquals("reviewer", view.actorRole());
    }

    @Test
    void attestationViewHasAllFields() {
        var view = new AttestationView(
                UUID.randomUUID(), UUID.randomUUID(), UUID.randomUUID(),
                "attestor-1", "AGENT", "reviewer",
                "SOUND", "evidence", 0.95, "capability-1",
                "accuracy", 0.9, Instant.now());
        assertEquals("SOUND", view.verdict());
        assertEquals(0.95, view.confidence());
    }

    @Test
    void trustScoreViewHasAllFields() {
        var view = new TrustScoreView(
                "actor-1", OptionalDouble.of(0.85),
                Map.of("cap1", 0.9), Map.of("dim1", 0.8));
        assertEquals("actor-1", view.actorId());
        assertTrue(view.globalScore().isPresent());
    }

    @Test
    void appendEntryRequestHasNoTenancyId() {
        var req = new AppendEntryRequest(
                UUID.randomUUID(), "actor-1", "AGENT",
                "EVENT", "role", "{}", Map.of());
        assertNull(req.getClass().getRecordComponents()[0].getName().equals("tenancyId")
                ? "found" : null);
    }
}
```

- [ ] **Step 2: Run the test to verify it fails (types don't exist yet)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ViewRecordTest`
Expected: compilation failure — classes not found

- [ ] **Step 3: Create all view and request records**

Create `api/src/main/java/io/casehub/ledger/api/view/LedgerEntryView.java`:
```java
package io.casehub.ledger.api.view;

import java.time.Instant;
import java.util.Map;
import java.util.UUID;

public record LedgerEntryView(
        UUID id,
        UUID subjectId,
        String tenancyId,
        int sequenceNumber,
        String entryType,
        String actorId,
        String actorType,
        String actorRole,
        Instant occurredAt,
        String digest,
        String traceId,
        UUID causedByEntryId,
        String metadata,
        Map<String, Object> domainData) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/LedgerEntryPage.java`:
```java
package io.casehub.ledger.api.view;

import java.util.List;

public record LedgerEntryPage(
        List<LedgerEntryView> entries,
        int totalCount,
        boolean hasMore) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/AttestationView.java`:
```java
package io.casehub.ledger.api.view;

import java.time.Instant;
import java.util.UUID;

public record AttestationView(
        UUID id,
        UUID ledgerEntryId,
        UUID subjectId,
        String attestorId,
        String attestorType,
        String attestorRole,
        String verdict,
        String evidence,
        double confidence,
        String capabilityTag,
        String trustDimension,
        Double dimensionScore,
        Instant occurredAt) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/VerificationView.java`:
```java
package io.casehub.ledger.api.view;

import java.util.UUID;

public record VerificationView(
        UUID subjectId,
        String treeRoot,
        boolean verified) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/InclusionProofView.java`:
```java
package io.casehub.ledger.api.view;

import java.util.List;
import java.util.UUID;

public record InclusionProofView(
        UUID entryId,
        int entryIndex,
        int treeSize,
        String leafHash,
        List<ProofStepView> siblings,
        String treeRoot) {

    public record ProofStepView(String hash, String side) {
    }
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/TrustScoreView.java`:
```java
package io.casehub.ledger.api.view;

import java.util.Map;
import java.util.OptionalDouble;

public record TrustScoreView(
        String actorId,
        OptionalDouble globalScore,
        Map<String, Double> capabilityScores,
        Map<String, Double> dimensionScores) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/CapabilityScoreView.java`:
```java
package io.casehub.ledger.api.view;

import java.util.Map;
import java.util.OptionalDouble;

public record CapabilityScoreView(
        String actorId,
        String capabilityTag,
        OptionalDouble score,
        int decisionCount,
        Map<String, Double> qualityScores) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/TrustRoutingProfileView.java`:
```java
package io.casehub.ledger.api.view;

import java.util.Map;
import java.util.OptionalDouble;

public record TrustRoutingProfileView(
        String actorId,
        String capabilityTag,
        OptionalDouble globalScore,
        OptionalDouble capabilityScore,
        int decisionCount,
        Map<String, Double> qualityScores) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/AppendEntryRequest.java`:
```java
package io.casehub.ledger.api.view;

import java.util.Map;
import java.util.UUID;

public record AppendEntryRequest(
        UUID subjectId,
        String actorId,
        String actorType,
        String entryType,
        String actorRole,
        String metadata,
        Map<String, Object> domainData) {
}
```

Create `api/src/main/java/io/casehub/ledger/api/view/CreateAttestationRequest.java`:
```java
package io.casehub.ledger.api.view;

import java.util.UUID;

public record CreateAttestationRequest(
        UUID entryId,
        String attestorId,
        String attestorType,
        String attestorRole,
        String verdict,
        String evidence,
        double confidence,
        String capabilityTag,
        String trustDimension,
        Double dimensionScore) {
}
```

- [ ] **Step 4: Create SPI interfaces**

Create `api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryApi.java`:
```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.view.AppendEntryRequest;
import io.casehub.ledger.api.view.LedgerEntryPage;
import io.casehub.ledger.api.view.LedgerEntryView;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import java.time.Instant;
import java.util.UUID;

@McpDomain("ledger/entries")
public interface LedgerEntryApi {

    @PlatformQuery("List ledger entries by subject or actor with optional time range")
    LedgerEntryPage listEntries(UUID subjectId, String actorId,
                                 String tenancyId, Instant from, Instant to,
                                 Integer offset, Integer limit);

    @PlatformQuery("Get a single ledger entry by ID")
    LedgerEntryView getEntry(@PathParam UUID id, String tenancyId);

    @PlatformQuery("Get entries causally triggered by this entry")
    java.util.List<LedgerEntryView> getCausedBy(@PathParam UUID id, String tenancyId);

    @PlatformMutation("Append a new audit entry to the ledger")
    LedgerEntryView appendEntry(AppendEntryRequest request, String tenancyId);
}
```

Create `api/src/main/java/io/casehub/ledger/api/spi/LedgerAttestationApi.java`:
```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.view.AttestationView;
import io.casehub.ledger.api.view.CreateAttestationRequest;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformMutation;
import io.casehub.platform.api.mcp.PlatformQuery;
import java.util.List;
import java.util.UUID;

@McpDomain("ledger/attestations")
public interface LedgerAttestationApi {

    @PlatformQuery("List attestations for a ledger entry, optionally filtered by capability tag")
    List<AttestationView> listAttestations(@PathParam UUID entryId,
                                            String tenancyId, String capabilityTag);

    @PlatformMutation("Create an attestation on a ledger entry")
    AttestationView createAttestation(CreateAttestationRequest request, String tenancyId);
}
```

Create `api/src/main/java/io/casehub/ledger/api/spi/LedgerVerificationApi.java`:
```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.view.InclusionProofView;
import io.casehub.ledger.api.view.VerificationView;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformQuery;
import java.util.UUID;

@McpDomain("ledger/verification")
public interface LedgerVerificationApi {

    @PlatformQuery("Verify Merkle tree integrity for all entries of a subject")
    VerificationView verify(UUID subjectId, String tenancyId);

    @PlatformQuery("Get Merkle inclusion proof for a single entry")
    InclusionProofView inclusionProof(@PathParam UUID entryId, String tenancyId);
}
```

Create `api/src/main/java/io/casehub/ledger/api/spi/LedgerTrustApi.java`:
```java
package io.casehub.ledger.api.spi;

import io.casehub.ledger.api.view.CapabilityScoreView;
import io.casehub.ledger.api.view.TrustRoutingProfileView;
import io.casehub.ledger.api.view.TrustScoreView;
import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.PathParam;
import io.casehub.platform.api.mcp.PlatformQuery;

@McpDomain("ledger/trust")
public interface LedgerTrustApi {

    @PlatformQuery("Global trust score for an actor — aggregate across all capabilities")
    TrustScoreView trustScore(@PathParam String actorId);

    @PlatformQuery("Capability-scoped trust score with quality dimensions")
    CapabilityScoreView capabilityScore(@PathParam String actorId,
                                         @PathParam String capabilityTag);

    @PlatformQuery("Composite trust routing profile — global + capability in one call")
    TrustRoutingProfileView routingProfile(@PathParam String actorId,
                                            @PathParam String capabilityTag);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest=ViewRecordTest`
Expected: PASS

- [ ] **Step 6: Run full api module build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api`
Expected: BUILD SUCCESS (SPI interfaces compile against platform-api annotations)

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/ledger/api/view/ api/src/main/java/io/casehub/ledger/api/spi/LedgerEntryApi.java api/src/main/java/io/casehub/ledger/api/spi/LedgerAttestationApi.java api/src/main/java/io/casehub/ledger/api/spi/LedgerVerificationApi.java api/src/main/java/io/casehub/ledger/api/spi/LedgerTrustApi.java api/src/test/java/io/casehub/ledger/api/view/ViewRecordTest.java
git commit -m "feat(#207): add SPI interfaces and view records for unified API generation

Four @McpDomain SPI interfaces (entries, attestations, verification,
trust) with 10 view/request records. Platform annotations only — no
JAX-RS on SPI interfaces.

Refs casehubio/ledger#207"
```

---

## Batch 2: Service implementations in runtime/

### Task 2: Create DefaultXxxApi service beans with tests

**Files:**
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerEntryApi.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerAttestationApi.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerVerificationApi.java`
- Create: `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerTrustApi.java`
- Test: `runtime/src/test/java/io/casehub/ledger/runtime/service/api/DefaultLedgerEntryApiTest.java`
- Test: `runtime/src/test/java/io/casehub/ledger/runtime/service/api/DefaultLedgerTrustApiTest.java`

**Interfaces:**
- Consumes: `LedgerEntryApi`, `LedgerAttestationApi`, `LedgerVerificationApi`, `LedgerTrustApi` (from Task 1); `LedgerEntryRepository`, `LedgerAppender`, `OutcomeRecorder`, `LedgerVerificationService`, `TrustScoreSource` (existing runtime beans)
- Produces: `@DefaultBean` CDI beans that are injected by generated REST/GraphQL code in Batch 3

- [ ] **Step 1: Write failing test for DefaultLedgerEntryApi**

```java
package io.casehub.ledger.runtime.service.api;

import io.casehub.ledger.api.view.LedgerEntryPage;
import io.casehub.ledger.api.view.LedgerEntryView;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class DefaultLedgerEntryApiTest {

    @Inject
    DefaultLedgerEntryApi api;

    @Test
    void listEntriesReturnsEmptyPageWhenNoEntries() {
        UUID subjectId = UUID.randomUUID();
        LedgerEntryPage page = api.listEntries(subjectId, null, null, null, null, null, null);
        assertThat(page.entries()).isEmpty();
        assertThat(page.totalCount()).isZero();
    }

    @Test
    void getEntryReturnsNullForMissingEntry() {
        LedgerEntryView entry = api.getEntry(UUID.randomUUID(), null);
        assertThat(entry).isNull();
    }

    @Test
    void getCausedByReturnsEmptyForMissingEntry() {
        var result = api.getCausedBy(UUID.randomUUID(), null);
        assertThat(result).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl api -q && JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultLedgerEntryApiTest`
Expected: compilation failure — `DefaultLedgerEntryApi` not found

- [ ] **Step 3: Implement DefaultLedgerEntryApi**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerEntryApi.java`:
```java
package io.casehub.ledger.runtime.service.api;

import io.casehub.ledger.api.model.AuditRecord;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.spi.LedgerAppender;
import io.casehub.ledger.api.spi.LedgerEntryApi;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.api.view.AppendEntryRequest;
import io.casehub.ledger.api.view.LedgerEntryPage;
import io.casehub.ledger.api.view.LedgerEntryView;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@DefaultBean
@ApplicationScoped
public class DefaultLedgerEntryApi implements LedgerEntryApi {

    @Inject LedgerEntryRepository repository;
    @Inject LedgerAppender appender;

    @Override
    public LedgerEntryPage listEntries(UUID subjectId, String actorId,
                                        String tenancyId, Instant from, Instant to,
                                        Integer offset, Integer limit) {
        String tid = defaultTenancyId(tenancyId);
        int off = offset != null ? offset : 0;
        int lim = limit != null ? limit : 20;

        List<? extends LedgerEntry> entries;
        if (subjectId != null) {
            entries = (from != null && to != null)
                    ? repository.findBySubjectIdAndTimeRange(subjectId, from, to, tid)
                    : repository.findBySubjectId(subjectId, tid);
        } else if (actorId != null) {
            Instant start = from != null ? from : Instant.EPOCH;
            Instant end = to != null ? to : Instant.now();
            entries = repository.findByActorId(actorId, start, end, tid);
        } else {
            entries = List.of();
        }

        List<LedgerEntryView> all = entries.stream().map(DefaultLedgerEntryApi::toView).toList();
        int total = all.size();
        int end = Math.min(off + lim, total);
        List<LedgerEntryView> items = off < total ? all.subList(off, end) : List.of();
        return new LedgerEntryPage(items, total, end < total);
    }

    @Override
    public LedgerEntryView getEntry(UUID id, String tenancyId) {
        return repository.findEntryById(id, defaultTenancyId(tenancyId))
                .map(DefaultLedgerEntryApi::toView)
                .orElse(null);
    }

    @Override
    public List<LedgerEntryView> getCausedBy(UUID id, String tenancyId) {
        return repository.findCausedBy(id, defaultTenancyId(tenancyId))
                .stream().map(DefaultLedgerEntryApi::toView).toList();
    }

    @Override
    public LedgerEntryView appendEntry(AppendEntryRequest request, String tenancyId) {
        String tid = defaultTenancyId(tenancyId);
        AuditRecord record = AuditRecord.event(request.actorId(), request.subjectId());
        if (request.actorRole() != null) {
            record = record.withActorRole(request.actorRole());
        }
        if (request.metadata() != null) {
            record = record.withMetadata(request.metadata());
        }
        if (request.domainData() != null) {
            record = record.withDomainData(request.domainData());
        }
        UUID entryId = appender.append(record, tid);
        return repository.findEntryById(entryId, tid)
                .map(DefaultLedgerEntryApi::toView)
                .orElseThrow();
    }

    static LedgerEntryView toView(LedgerEntry e) {
        return new LedgerEntryView(
                e.id, e.subjectId, e.tenancyId, e.sequenceNumber,
                e.entryType != null ? e.entryType.name() : null,
                e.actorId,
                e.actorType != null ? e.actorType.name() : null,
                e.actorRole, e.occurredAt, e.digest, e.traceId,
                e.causedByEntryId, e.metadata, e.domainData);
    }

    static String defaultTenancyId(String tenancyId) {
        return tenancyId != null ? tenancyId : TenancyConstants.DEFAULT_TENANT_ID;
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultLedgerEntryApiTest`
Expected: PASS

- [ ] **Step 5: Implement DefaultLedgerAttestationApi, DefaultLedgerVerificationApi, DefaultLedgerTrustApi**

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerAttestationApi.java`:
```java
package io.casehub.ledger.runtime.service.api;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.spi.LedgerAttestationApi;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.api.view.AttestationView;
import io.casehub.ledger.api.view.CreateAttestationRequest;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@DefaultBean
@ApplicationScoped
public class DefaultLedgerAttestationApi implements LedgerAttestationApi {

    @Inject LedgerEntryRepository repository;

    @Override
    public List<AttestationView> listAttestations(UUID entryId, String tenancyId,
                                                    String capabilityTag) {
        String tid = DefaultLedgerEntryApi.defaultTenancyId(tenancyId);
        List<LedgerAttestation> attestations = capabilityTag != null
                ? repository.findAttestationsByEntryIdAndCapabilityTag(entryId, capabilityTag, tid)
                : repository.findAttestationsByEntryId(entryId, tid);
        return attestations.stream().map(DefaultLedgerAttestationApi::toView).toList();
    }

    @Override
    public AttestationView createAttestation(CreateAttestationRequest request,
                                              String tenancyId) {
        String tid = DefaultLedgerEntryApi.defaultTenancyId(tenancyId);
        var entry = repository.findEntryById(request.entryId(), tid)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Entry not found: " + request.entryId()));

        var attestation = new LedgerAttestation();
        attestation.id = UUID.randomUUID();
        attestation.ledgerEntryId = request.entryId();
        attestation.subjectId = entry.subjectId;
        attestation.attestorId = request.attestorId();
        attestation.attestorType = ActorType.valueOf(request.attestorType());
        attestation.attestorRole = request.attestorRole();
        attestation.verdict = AttestationVerdict.valueOf(request.verdict());
        attestation.evidence = request.evidence();
        attestation.confidence = request.confidence();
        attestation.capabilityTag = request.capabilityTag();
        attestation.trustDimension = request.trustDimension();
        attestation.dimensionScore = request.dimensionScore();
        attestation.occurredAt = Instant.now();

        LedgerAttestation saved = repository.saveAttestation(attestation, tid);
        return toView(saved);
    }

    static AttestationView toView(LedgerAttestation a) {
        return new AttestationView(
                a.id, a.ledgerEntryId, a.subjectId,
                a.attestorId,
                a.attestorType != null ? a.attestorType.name() : null,
                a.attestorRole,
                a.verdict != null ? a.verdict.name() : null,
                a.evidence, a.confidence, a.capabilityTag,
                a.trustDimension, a.dimensionScore, a.occurredAt);
    }
}
```

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerVerificationApi.java`:
```java
package io.casehub.ledger.runtime.service.api;

import io.casehub.ledger.api.spi.LedgerVerificationApi;
import io.casehub.ledger.api.view.InclusionProofView;
import io.casehub.ledger.api.view.VerificationView;
import io.casehub.ledger.core.merkle.InclusionProof;
import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.platform.api.identity.TenancyConstants;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.UUID;

@DefaultBean
@ApplicationScoped
public class DefaultLedgerVerificationApi implements LedgerVerificationApi {

    @Inject LedgerVerificationService verificationService;

    @Override
    public VerificationView verify(UUID subjectId, String tenancyId) {
        String tid = DefaultLedgerEntryApi.defaultTenancyId(tenancyId);
        boolean verified = verificationService.verify(subjectId, tid);
        String treeRoot = verified ? verificationService.treeRoot(subjectId, tid) : null;
        return new VerificationView(subjectId, treeRoot, verified);
    }

    @Override
    public InclusionProofView inclusionProof(UUID entryId, String tenancyId) {
        String tid = DefaultLedgerEntryApi.defaultTenancyId(tenancyId);
        InclusionProof proof = verificationService.inclusionProof(entryId, tid);
        var steps = proof.siblings().stream()
                .map(s -> new InclusionProofView.ProofStepView(s.hash(), s.side().name()))
                .toList();
        return new InclusionProofView(
                proof.entryId(), proof.entryIndex(), proof.treeSize(),
                proof.leafHash(), steps, proof.treeRoot());
    }
}
```

Create `runtime/src/main/java/io/casehub/ledger/runtime/service/api/DefaultLedgerTrustApi.java`:
```java
package io.casehub.ledger.runtime.service.api;

import io.casehub.ledger.api.spi.LedgerTrustApi;
import io.casehub.ledger.api.spi.TrustScoreSource;
import io.casehub.ledger.api.view.CapabilityScoreView;
import io.casehub.ledger.api.view.TrustRoutingProfileView;
import io.casehub.ledger.api.view.TrustScoreView;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@DefaultBean
@ApplicationScoped
public class DefaultLedgerTrustApi implements LedgerTrustApi {

    @Inject TrustScoreSource trustScoreSource;

    @Override
    public TrustScoreView trustScore(String actorId) {
        return new TrustScoreView(
                actorId,
                trustScoreSource.globalScore(actorId),
                trustScoreSource.allCapabilityScores(actorId),
                trustScoreSource.allDimensionScores(actorId));
    }

    @Override
    public CapabilityScoreView capabilityScore(String actorId, String capabilityTag) {
        return new CapabilityScoreView(
                actorId, capabilityTag,
                trustScoreSource.capabilityScore(actorId, capabilityTag),
                trustScoreSource.decisionCount(actorId, capabilityTag),
                trustScoreSource.qualityScores(actorId, capabilityTag));
    }

    @Override
    public TrustRoutingProfileView routingProfile(String actorId, String capabilityTag) {
        return new TrustRoutingProfileView(
                actorId, capabilityTag,
                trustScoreSource.globalScore(actorId),
                trustScoreSource.capabilityScore(actorId, capabilityTag),
                trustScoreSource.decisionCount(actorId, capabilityTag),
                trustScoreSource.qualityScores(actorId, capabilityTag));
    }
}
```

- [ ] **Step 6: Write and run trust API test**

```java
package io.casehub.ledger.runtime.service.api;

import io.casehub.ledger.api.view.TrustScoreView;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class DefaultLedgerTrustApiTest {

    @Inject
    DefaultLedgerTrustApi api;

    @Test
    void trustScoreReturnsEmptyForUnknownActor() {
        TrustScoreView view = api.trustScore("unknown-actor");
        assertThat(view.actorId()).isEqualTo("unknown-actor");
        assertThat(view.globalScore()).isEmpty();
        assertThat(view.capabilityScores()).isEmpty();
    }
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=DefaultLedgerTrustApiTest`
Expected: PASS

- [ ] **Step 7: Run full runtime test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: all existing tests pass + new tests pass

- [ ] **Step 8: Commit**

```bash
git add runtime/src/main/java/io/casehub/ledger/runtime/service/api/ runtime/src/test/java/io/casehub/ledger/runtime/service/api/
git commit -m "feat(#207): add DefaultXxxApi service implementations

Four @DefaultBean service beans implementing the SPI interfaces.
Each delegates to existing repositories/services with entity→view
mapping. No new business logic.

Refs casehubio/ledger#207"
```

---

## Batch 3: APT wiring, migration, and cleanup

### Task 3: Configure APT, update dependencies, delete hand-written endpoints, update tests

**Files:**
- Modify: `rest/pom.xml` — change compile dep from runtime to api, add APT config
- Modify: `graphql/pom.xml` — change compile dep from runtime to api, add APT config
- Delete: `rest/src/main/java/io/casehub/ledger/rest/LedgerEntryResource.java` (use `ide_refactor_safe_delete`)
- Delete: `rest/src/main/java/io/casehub/ledger/rest/AttestationResource.java`
- Delete: `rest/src/main/java/io/casehub/ledger/rest/MerkleVerificationResource.java`
- Delete: `rest/src/main/java/io/casehub/ledger/rest/TrustScoreResource.java`
- Delete: `rest/src/main/java/io/casehub/ledger/rest/dto/` (all files)
- Delete: `graphql/src/main/java/io/casehub/ledger/graphql/LedgerQueryResolver.java`
- Delete: `graphql/src/main/java/io/casehub/ledger/graphql/LedgerMutationResolver.java`
- Delete: `graphql/src/main/java/io/casehub/ledger/graphql/dto/` (all files except LedgerModelEnricher-related)
- Modify: existing REST tests — update paths to match generated kebab-case paths
- Modify: existing GraphQL tests — update operation names if changed

**Interfaces:**
- Consumes: SPI interfaces (Task 1), service implementations (Task 2), `graphql-generator` APT (casehub-platform-graphql-generator)
- Produces: generated `GeneratedLedgerEntriesResource`, `GeneratedLedgerAttestationsResource`, `GeneratedLedgerVerificationResource`, `GeneratedLedgerTrustResource` (REST); generated `GeneratedLedgerEntriesResolver`, `GeneratedLedgerAttestationsResolver`, `GeneratedLedgerVerificationResolver`, `GeneratedLedgerTrustResolver` (GraphQL)

- [ ] **Step 1: Update rest/pom.xml — change compile dep and add APT**

Change `casehub-ledger` compile dep to `casehub-ledger-api` compile + `casehub-ledger` test. Add APT config and `jakarta.validation-api` for `@Valid` on generated body params.

```xml
<!-- Replace existing casehub-ledger dependency -->
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-api</artifactId>
  <version>${project.version}</version>
</dependency>
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger</artifactId>
  <version>${project.version}</version>
  <scope>test</scope>
</dependency>

<!-- Add jakarta.validation for @Valid on generated body params -->
<dependency>
  <groupId>jakarta.validation</groupId>
  <artifactId>jakarta.validation-api</artifactId>
  <scope>provided</scope>
</dependency>
```

Add APT config to `maven-compiler-plugin`:
```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <annotationProcessorPaths>
      <path>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-graphql-generator</artifactId>
        <version>${casehub-platform.version}</version>
      </path>
    </annotationProcessorPaths>
    <compilerArgs>
      <arg>-AdomainFilter=ledger/entries,ledger/attestations,ledger/verification,ledger/trust</arg>
      <arg>-AgenerateGraphQL=false</arg>
    </compilerArgs>
  </configuration>
</plugin>
```

- [ ] **Step 2: Update graphql/pom.xml similarly**

Same pattern: `casehub-ledger-api` compile, `casehub-ledger` test. APT with `generateRest=false`.

```xml
<compilerArgs>
  <arg>-AdomainFilter=ledger/entries,ledger/attestations,ledger/verification,ledger/trust</arg>
  <arg>-AgenerateRest=false</arg>
</compilerArgs>
```

- [ ] **Step 3: Build to verify APT generates code**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,rest,graphql`
Expected: BUILD SUCCESS. Check `rest/target/generated-sources/annotations/` for generated REST resources. Check `graphql/target/generated-sources/annotations/` for generated GraphQL resolvers.

- [ ] **Step 4: Delete hand-written REST resources and DTOs**

Use `ide_refactor_safe_delete` for each file:
- `LedgerEntryResource.java`
- `AttestationResource.java`
- `MerkleVerificationResource.java`
- `TrustScoreResource.java`
- All files in `rest/dto/` directory

- [ ] **Step 5: Delete hand-written GraphQL resolvers and DTOs**

Use `ide_refactor_safe_delete` for:
- `LedgerQueryResolver.java`
- `LedgerMutationResolver.java`
- All files in `graphql/dto/` directory

Keep: `LedgerModelEnricher.java`

- [ ] **Step 6: Update REST tests to use generated paths**

The generated paths follow kebab-case convention. Update test paths:
- `/api/v1/ledger/entries` → `/api/ledger-entries/list-entries` (or whatever the generated class-level path is — check generated source)
- `/api/v1/ledger/entries/{id}` → `/api/ledger-entries/get-entry/{id}`
- etc.

Check generated source at `rest/target/generated-sources/annotations/` to determine exact paths before updating tests.

- [ ] **Step 7: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean test`
Expected: all tests pass across all modules

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat(#207): wire APT generation, delete hand-written endpoints

Configure graphql-generator APT in rest/ and graphql/ modules.
Delete all hand-written REST resources (4), GraphQL resolvers (2),
and DTOs (23 files). Generated code produces the same API surface
from SPI interfaces.

rest/ depends on casehub-ledger-api (compile) not casehub-ledger
(runtime). Domain filtering prevents generating for other modules.

Closes casehubio/ledger#207"
```

---

## References

- [specs/issue-295-unified-api-generation/2026-09-14-unified-api-generation-design.md] — design spec
- [rest/src/main/java/io/casehub/ledger/rest/LedgerEntryResource.java] — existing REST resource (to be deleted)
- [rest/src/main/java/io/casehub/ledger/rest/AttestationResource.java] — existing REST resource
- [rest/src/main/java/io/casehub/ledger/rest/MerkleVerificationResource.java] — existing REST resource
- [rest/src/main/java/io/casehub/ledger/rest/TrustScoreResource.java] — existing REST resource
- [graphql/src/main/java/io/casehub/ledger/graphql/LedgerQueryResolver.java] — existing GraphQL resolver
- [graphql/src/main/java/io/casehub/ledger/graphql/LedgerMutationResolver.java] — existing GraphQL resolver
- [platform GraphQLResolverProcessor.java] — the code generator
- [platform CallbackApi.java, NotificationPreferenceApi.java] — migration examples
- GE-20260914-3854b8 — APT domain filtering
- GE-20260816-d18a02 — GraphQL modules depend on API not runtime
- GE-20260810-31134a — casehub-ledger-rest already provides endpoints
- casehubio/platform#295 — parent epic
- casehubio/ledger#207 — focal issue
