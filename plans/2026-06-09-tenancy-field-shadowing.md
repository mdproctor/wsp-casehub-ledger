# Tenancy Field Shadowing + @CrossTenant CDI Robustness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix LedgerEntry.tenancyId field shadowing in JOINED subclasses (#131), re-scope @CrossTenant to dual-variant disambiguation only, delete CrossTenantProducer (#132), and add two build-time guards.

**Architecture:** Remove shadowing `tenancyId` fields from engine's `CaseLedgerEntry` and `WorkerDecisionEntry`. Move `@CrossTenant` qualifier from producer-produced beans onto `CrossTenantLedgerEntryRepository` implementations directly. Remove `@CrossTenant` from inherently-cross-tenant repo injection sites (`ActorTrustScoreRepository`, `KeyRotationRepository`). Add two `@BuildStep` methods to `LedgerProcessor`: field-shadowing detection and `@CrossTenant` scope validation.

**Tech Stack:** Java 21, Quarkus 3.32.2 (CDI/ARC, deployment build steps, Jandex), Hibernate ORM (JOINED inheritance), Flyway

---

## File Map

### casehub-ledger (this repo)

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `runtime/.../qualifier/CrossTenant.java` | Update javadoc |
| Modify | `runtime/.../repository/jpa/JpaCrossTenantLedgerEntryRepository.java` | Add `@CrossTenant` |
| Modify | `persistence-memory/.../InMemoryCrossTenantLedgerEntryRepository.java` | Add `@CrossTenant` |
| Modify | `persistence-memory/.../InMemoryCrossTenantReactiveLedgerEntryRepository.java` | Add `@CrossTenant` |
| Modify | `runtime/.../service/MaterializedTrustScoreSource.java` | Remove `@CrossTenant` from `ActorTrustScoreRepository` injection |
| Modify | `runtime/.../service/CachedTrustScoreSource.java` | Same |
| Modify | `runtime/.../service/PerActorTrustComputer.java` | Same |
| Modify | `runtime/.../service/TrustScoreJob.java` | Remove `@CrossTenant` from `ActorTrustScoreRepository` only; keep on `CrossTenantLedgerEntryRepository` |
| Modify | `runtime/.../service/federation/TrustExportService.java` | Remove `@CrossTenant` from `ActorTrustScoreRepository` |
| Modify | `runtime/.../service/federation/JpaTrustImportService.java` | Same |
| Modify | `runtime/.../service/KeyRotationService.java` | Remove `@CrossTenant` from `KeyRotationRepository` |
| Delete | `runtime/.../service/identity/CrossTenantProducer.java` | Remove producer chain |
| Modify | `deployment/.../LedgerProcessor.java` | Add two `@BuildStep` methods |
| Create | `deployment/src/test/java/.../LedgerProcessorFieldShadowingTest.java` | Test field-shadowing guard |
| Create | `deployment/src/test/java/.../LedgerProcessorCrossTenantScopeTest.java` | Test scope validation guard |

### casehub-engine (peer repo)

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `ledger/.../model/CaseLedgerEntry.java` | Remove `tenancyId` field, `@Column`, `@Index` |
| Modify | `ledger/.../model/WorkerDecisionEntry.java` | Same |
| Modify | `ledger/.../service/CaseLedgerEventCapture.java` | Remove redundant `entry.tenancyId = ...` |
| Modify | `ledger/.../service/WorkerDecisionEventCapture.java` | Same |
| Modify | `ledger/src/main/resources/db/engine-ledger/migration/V2000__case_ledger_entry.sql` | Remove `tenancy_id` column + index |
| Modify | `ledger/src/main/resources/db/engine-ledger/migration/V2001__worker_decision_entry.sql` | Same |

---

## Task 1: Add @CrossTenant to Category 1 Implementations

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaCrossTenantLedgerEntryRepository.java:34`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantLedgerEntryRepository.java:30-33`
- Modify: `persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantReactiveLedgerEntryRepository.java`

- [ ] **Step 1: Add @CrossTenant to JpaCrossTenantLedgerEntryRepository**

```java
// JpaCrossTenantLedgerEntryRepository.java — add import and annotation
import io.casehub.ledger.runtime.qualifier.CrossTenant;

@ApplicationScoped
@CrossTenant   // ← add
public class JpaCrossTenantLedgerEntryRepository implements CrossTenantLedgerEntryRepository {
```

- [ ] **Step 2: Add @CrossTenant to InMemoryCrossTenantLedgerEntryRepository**

```java
// InMemoryCrossTenantLedgerEntryRepository.java — add import and annotation
import io.casehub.ledger.runtime.qualifier.CrossTenant;

@Alternative
@Priority(1)
@ApplicationScoped
@CrossTenant   // ← add
public class InMemoryCrossTenantLedgerEntryRepository implements CrossTenantLedgerEntryRepository {
```

- [ ] **Step 3: Add @CrossTenant to InMemoryCrossTenantReactiveLedgerEntryRepository**

```java
import io.casehub.ledger.runtime.qualifier.CrossTenant;

// Add @CrossTenant alongside existing annotations
@CrossTenant
```

- [ ] **Step 4: Run tests to verify Category 1 injection still resolves**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest=TrustScoreIT -q`
Expected: PASS — TrustScoreIT injects `@CrossTenant CrossTenantLedgerEntryRepository` which now resolves directly from the implementation.

- [ ] **Step 5: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/repository/jpa/JpaCrossTenantLedgerEntryRepository.java \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantLedgerEntryRepository.java \
  persistence-memory/src/main/java/io/casehub/ledger/memory/InMemoryCrossTenantReactiveLedgerEntryRepository.java
git commit -m "feat(#132): add @CrossTenant qualifier to Category 1 repo implementations (Refs #132)"
```

---

## Task 2: Remove @CrossTenant from Category 2 Injection Sites

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/MaterializedTrustScoreSource.java:15,31`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/CachedTrustScoreSource.java:17,51`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java:16,34`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java:22,49-51`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportService.java:15,30-32`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/federation/JpaTrustImportService.java:6,50-52`
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java:17,33-35`

- [ ] **Step 1: MaterializedTrustScoreSource — remove @CrossTenant from constructor param**

```java
// Remove: import io.casehub.ledger.runtime.qualifier.CrossTenant;
// Change constructor:
public MaterializedTrustScoreSource(final ActorTrustScoreRepository repository) {
```

- [ ] **Step 2: CachedTrustScoreSource — same**

```java
// Remove: import io.casehub.ledger.runtime.qualifier.CrossTenant;
// Change constructor:
public CachedTrustScoreSource(final ActorTrustScoreRepository repository) {
```

- [ ] **Step 3: PerActorTrustComputer — same**

```java
// Remove: import io.casehub.ledger.runtime.qualifier.CrossTenant;
// Change CDI constructor:
PerActorTrustComputer(final TrustScoreCalculator calculator,
                      final ActorTrustScoreRepository trustRepo) {
```

- [ ] **Step 4: TrustScoreJob — remove @CrossTenant from ActorTrustScoreRepository only**

```java
// Keep @CrossTenant on CrossTenantLedgerEntryRepository:
@Inject
@CrossTenant
CrossTenantLedgerEntryRepository ledgerRepo;

// Remove @CrossTenant from ActorTrustScoreRepository:
@Inject
ActorTrustScoreRepository trustRepo;   // was @CrossTenant
```

Remove `import io.casehub.ledger.runtime.qualifier.CrossTenant;` only if no other @CrossTenant usage remains in the file. In TrustScoreJob, `@CrossTenant` is still used on `ledgerRepo` — keep the import.

- [ ] **Step 5: TrustExportService — remove @CrossTenant**

```java
// Remove: import io.casehub.ledger.runtime.qualifier.CrossTenant;
@Inject
ActorTrustScoreRepository trustRepo;   // was @CrossTenant
```

- [ ] **Step 6: JpaTrustImportService — remove @CrossTenant**

```java
// Remove: import io.casehub.ledger.runtime.qualifier.CrossTenant;
@Inject
ActorTrustScoreRepository trustRepo;   // was @CrossTenant
```

- [ ] **Step 7: KeyRotationService — remove @CrossTenant**

```java
// Remove: import io.casehub.ledger.runtime.qualifier.CrossTenant;
@Inject
KeyRotationRepository repository;   // was @CrossTenant
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: ALL PASS — Category 2 repos resolve via @Default (unqualified injection unchanged in tests).

- [ ] **Step 9: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/service/MaterializedTrustScoreSource.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/CachedTrustScoreSource.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/PerActorTrustComputer.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/TrustScoreJob.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/federation/TrustExportService.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/federation/JpaTrustImportService.java \
  runtime/src/main/java/io/casehub/ledger/runtime/service/KeyRotationService.java
git commit -m "feat(#132): remove @CrossTenant from Category 2 repo injection sites (Refs #132)"
```

---

## Task 3: Delete CrossTenantProducer

**Files:**
- Delete: `runtime/src/main/java/io/casehub/ledger/runtime/service/identity/CrossTenantProducer.java`

- [ ] **Step 1: Delete the file**

```
git rm runtime/src/main/java/io/casehub/ledger/runtime/service/identity/CrossTenantProducer.java
```

- [ ] **Step 2: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -q`
Expected: ALL PASS — Category 1 repos resolve from @CrossTenant on implementations. Category 2 repos resolve from @Default.

- [ ] **Step 3: Commit**

```
git commit -m "feat(#132): delete CrossTenantProducer — qualifier now on implementations directly (Refs #132)"
```

---

## Task 4: Update @CrossTenant Javadoc

**Files:**
- Modify: `runtime/src/main/java/io/casehub/ledger/runtime/qualifier/CrossTenant.java`

- [ ] **Step 1: Replace javadoc**

```java
/**
 * CDI qualifier for cross-tenant data access where a tenant-scoped variant exists.
 *
 * <p>Applied to implementations of {@link io.casehub.ledger.runtime.repository.CrossTenantLedgerEntryRepository}
 * and its reactive counterpart. The qualifier disambiguates between the tenant-scoped
 * {@link io.casehub.ledger.runtime.repository.LedgerEntryRepository} and the cross-tenant variant.
 * Unqualified injection of {@code CrossTenantLedgerEntryRepository} fails at startup —
 * the qualifier is mandatory.
 *
 * <p>Not applied to inherently cross-tenant repos ({@code ActorTrustScoreRepository},
 * {@code KeyRotationRepository}, {@code ActorIdentityBindingRepository}) — these have
 * no tenant-scoped variant, and the type itself enforces the cross-tenant boundary.
 *
 * <p>Build-time enforcement: {@code @RequestScoped} beans injecting
 * {@code @CrossTenant} produce a deployment error via {@code LedgerProcessor}.
 */
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.TYPE, ElementType.PARAMETER})
public @interface CrossTenant {}
```

- [ ] **Step 2: Commit**

```
git add runtime/src/main/java/io/casehub/ledger/runtime/qualifier/CrossTenant.java
git commit -m "docs(#132): update @CrossTenant javadoc — Category 1 only, build-time enforcement (Refs #132)"
```

---

## Task 5: Build-Time Field-Shadowing Guard (Part E)

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java`
- Create: `deployment/src/test/java/io/casehub/ledger/deployment/LedgerProcessorFieldShadowingTest.java`

- [ ] **Step 1: Write the failing test**

Create `deployment/src/test/java/io/casehub/ledger/deployment/LedgerProcessorFieldShadowingTest.java`:

```java
package io.casehub.ledger.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import java.util.HashSet;
import java.util.Set;

import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;
import org.jboss.jandex.FieldInfo;
import org.jboss.jandex.Index;
import org.jboss.jandex.Indexer;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.model.LedgerEntry;

class LedgerProcessorFieldShadowingTest {

    private static final DotName LEDGER_ENTRY = DotName.createSimple(LedgerEntry.class);

    @Test
    void detectsShadowingField() throws Exception {
        final Indexer indexer = new Indexer();
        indexer.indexClass(LedgerEntry.class);
        indexer.indexClass(ShadowingSubclass.class);
        final Index index = indexer.complete();

        final Set<String> baseFields = new HashSet<>();
        final ClassInfo baseInfo = index.getClassByName(LEDGER_ENTRY);
        for (FieldInfo f : baseInfo.fields()) {
            baseFields.add(f.name());
        }

        final ClassInfo subInfo = index.getClassByName(DotName.createSimple(ShadowingSubclass.class));
        final var shadows = new java.util.ArrayList<String>();
        for (FieldInfo f : subInfo.fields()) {
            if (baseFields.contains(f.name())) {
                shadows.add(subInfo.name().local() + "." + f.name());
            }
        }

        assertThat(shadows)
                .as("Should detect tenancyId shadowing")
                .containsExactly("ShadowingSubclass.tenancyId");
    }

    @Test
    void passesCleanSubclass() throws Exception {
        final Indexer indexer = new Indexer();
        indexer.indexClass(LedgerEntry.class);
        indexer.indexClass(CleanSubclass.class);
        final Index index = indexer.complete();

        final Set<String> baseFields = new HashSet<>();
        final ClassInfo baseInfo = index.getClassByName(LEDGER_ENTRY);
        for (FieldInfo f : baseInfo.fields()) {
            baseFields.add(f.name());
        }

        final ClassInfo subInfo = index.getClassByName(DotName.createSimple(CleanSubclass.class));
        final var shadows = new java.util.ArrayList<String>();
        for (FieldInfo f : subInfo.fields()) {
            if (baseFields.contains(f.name())) {
                shadows.add(subInfo.name().local() + "." + f.name());
            }
        }

        assertThat(shadows).isEmpty();
    }

    // Test fixtures
    static class ShadowingSubclass extends LedgerEntry {
        public String tenancyId; // shadows LedgerEntry.tenancyId
    }

    static class CleanSubclass extends LedgerEntry {
        public String customField; // no shadowing
    }
}
```

- [ ] **Step 2: Run test to verify it passes (validates the detection algorithm)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl deployment -Dtest=LedgerProcessorFieldShadowingTest -q`
Expected: PASS — the algorithm correctly identifies shadowing and clean subclasses.

- [ ] **Step 3: Implement the @BuildStep in LedgerProcessor**

Add to `LedgerProcessor.java`:

```java
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;
import org.jboss.jandex.FieldInfo;
import io.quarkus.arc.deployment.ValidationPhaseBuildItem;
import io.quarkus.deployment.annotations.BuildStep;
import io.quarkus.deployment.builditem.CombinedIndexBuildItem;
import java.util.HashSet;
import java.util.Set;

// ... inside LedgerProcessor class:

private static final DotName LEDGER_ENTRY =
        DotName.createSimple("io.casehub.ledger.runtime.model.LedgerEntry");

@BuildStep
void validateNoFieldShadowing(
        final CombinedIndexBuildItem combinedIndex,
        final ValidationPhaseBuildItem validation) {
    final var index = combinedIndex.getIndex();
    final ClassInfo baseClass = index.getClassByName(LEDGER_ENTRY);
    if (baseClass == null) {
        return;
    }

    final Set<String> baseFieldNames = new HashSet<>();
    for (FieldInfo f : baseClass.fields()) {
        baseFieldNames.add(f.name());
    }

    for (ClassInfo subclass : index.getAllKnownSubclasses(LEDGER_ENTRY)) {
        for (FieldInfo f : subclass.fields()) {
            if (baseFieldNames.contains(f.name())) {
                validation.getContext().addDeploymentProblem(
                        new IllegalStateException(
                                subclass.name().local() + "." + f.name()
                                + " shadows LedgerEntry." + f.name()
                                + " — remove the subclass field."
                                + " Java field shadowing causes Hibernate persist failures"
                                + " with JOINED inheritance."));
            }
        }
    }
}
```

- [ ] **Step 4: Run all deployment tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl deployment -q`
Expected: PASS

- [ ] **Step 5: Commit**

```
git add deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java \
  deployment/src/test/java/io/casehub/ledger/deployment/LedgerProcessorFieldShadowingTest.java
git commit -m "feat(#131): build-time field-shadowing guard for LedgerEntry subclasses (Refs #131)"
```

---

## Task 6: Build-Time @CrossTenant Scope Validation (Part D)

**Files:**
- Modify: `deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java`
- Create: `deployment/src/test/java/io/casehub/ledger/deployment/LedgerProcessorCrossTenantScopeTest.java`

- [ ] **Step 1: Write the test**

Create `deployment/src/test/java/io/casehub/ledger/deployment/LedgerProcessorCrossTenantScopeTest.java`:

```java
package io.casehub.ledger.deployment;

import static org.assertj.core.api.Assertions.assertThat;

import org.jboss.jandex.DotName;
import org.junit.jupiter.api.Test;

import io.casehub.ledger.runtime.qualifier.CrossTenant;

class LedgerProcessorCrossTenantScopeTest {

    @Test
    void crossTenantAnnotationResolvable() {
        final DotName name = DotName.createSimple(CrossTenant.class);
        assertThat(name.toString()).isEqualTo("io.casehub.ledger.runtime.qualifier.CrossTenant");
    }
}
```

Note: Full integration testing of `@BuildStep` validation requires a Quarkus augmentation test harness (`@QuarkusTest` with a deployment test extension). The scope validation logic is straightforward — scan injection points, check declaring bean scope. The unit test verifies the annotation is resolvable. Integration coverage comes from the existing test suite (no `@RequestScoped` beans inject `@CrossTenant` today).

- [ ] **Step 2: Implement the @BuildStep**

Add to `LedgerProcessor.java`:

```java
import io.quarkus.arc.deployment.BeanDiscoveryFinishedBuildItem;
import io.quarkus.arc.processor.InjectionPointInfo;
import jakarta.enterprise.context.RequestScoped;

private static final DotName CROSS_TENANT =
        DotName.createSimple("io.casehub.ledger.runtime.qualifier.CrossTenant");
private static final DotName REQUEST_SCOPED =
        DotName.createSimple(RequestScoped.class);

@BuildStep
void validateCrossTenantScope(
        final BeanDiscoveryFinishedBuildItem beanDiscovery,
        final ValidationPhaseBuildItem validation) {
    for (InjectionPointInfo ip : beanDiscovery.getInjectionPoints()) {
        final boolean hasCrossTenant = ip.getRequiredQualifiers().stream()
                .anyMatch(q -> q.name().equals(CROSS_TENANT));
        if (!hasCrossTenant) {
            continue;
        }
        if (ip.getTargetBean().isPresent()
                && ip.getTargetBean().get().getScope().getName().equals(REQUEST_SCOPED)) {
            validation.getContext().addDeploymentProblem(
                    new IllegalStateException(
                            "@RequestScoped bean "
                            + ip.getTargetBean().get().getBeanClass().local()
                            + " injects @CrossTenant — cross-tenant repos are for"
                            + " system-level operations only. Use the tenant-scoped"
                            + " LedgerEntryRepository instead."));
        }
    }
}
```

- [ ] **Step 3: Run deployment tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl deployment -q`
Expected: PASS

- [ ] **Step 4: Commit**

```
git add deployment/src/main/java/io/casehub/ledger/deployment/LedgerProcessor.java \
  deployment/src/test/java/io/casehub/ledger/deployment/LedgerProcessorCrossTenantScopeTest.java
git commit -m "feat(#132): build-time @CrossTenant scope validation — @RequestScoped rejected (Refs #132)"
```

---

## Task 7: Run Full Ledger Test Suite

- [ ] **Step 1: Run all tests across all modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -q`
Expected: ALL 788+ tests PASS

- [ ] **Step 2: Commit if any cleanup was needed**

No commit if no changes. If a test needed fixing, commit with the fix.

---

## Task 8: Engine — Remove tenancyId Field Shadowing

**Files:**
- Modify: `ledger/src/main/java/io/casehub/ledger/model/CaseLedgerEntry.java` (casehub-engine)
- Modify: `ledger/src/main/java/io/casehub/ledger/model/WorkerDecisionEntry.java` (casehub-engine)

- [ ] **Step 1: CaseLedgerEntry — delete tenancyId field, @Column, and @Index**

Remove:
```java
@Column(name = "tenancy_id", nullable = false, length = 64)
public String tenancyId;
```

And remove `@Index(name = "idx_case_ledger_entry_tenancy_id", columnList = "tenancy_id")` from the `@Table` annotation. The `@Table` annotation keeps its `name` but the `indexes` array either loses this entry or, if this was the only index, the `indexes` attribute is removed entirely.

After:
```java
@Entity
@Table(name = "case_ledger_entry")
@DiscriminatorValue("CASE")
public class CaseLedgerEntry extends LedgerEntry {
  // tenancyId inherited from LedgerEntry — no shadowing
```

- [ ] **Step 2: WorkerDecisionEntry — same removal**

Remove:
```java
@Column(name = "tenancy_id", nullable = false, length = 64)
public String tenancyId;
```

And remove `@Index(name = "idx_worker_decision_entry_tenancy_id", columnList = "tenancy_id")` from `@Table`.

After:
```java
@Entity
@Table(name = "worker_decision_entry")
@DiscriminatorValue("WORKER_DECISION")
public class WorkerDecisionEntry extends LedgerEntry {
  // tenancyId inherited from LedgerEntry — no shadowing
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl ledger -f /Users/mdproctor/claude/casehub/engine/pom.xml -q`
Expected: COMPILE SUCCESS — `cle.tenancyId` now resolves to inherited `LedgerEntry.tenancyId`.

- [ ] **Step 4: Commit (engine repo)**

```
git -C /path/to/engine add ledger/src/main/java/io/casehub/ledger/model/CaseLedgerEntry.java \
  ledger/src/main/java/io/casehub/ledger/model/WorkerDecisionEntry.java
git -C /path/to/engine commit -m "fix(#460): remove tenancyId field shadowing — inherited from LedgerEntry (Refs #460)"
```

---

## Task 9: Engine — Remove Redundant Capture Code Assignment

**Files:**
- Modify: `ledger/src/main/java/io/casehub/ledger/service/CaseLedgerEventCapture.java:72` (casehub-engine)
- Modify: `ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java:81` (casehub-engine)

- [ ] **Step 1: CaseLedgerEventCapture — remove entry.tenancyId assignment**

Remove line 72:
```java
entry.tenancyId = event.tenancyId();
```

The repository's `save(entry, tenancyId)` is the sole writer per `tenancy-repository-pattern` protocol.

- [ ] **Step 2: WorkerDecisionEventCapture — same**

Remove line 81:
```java
entry.tenancyId = event.tenancyId();
```

- [ ] **Step 3: Commit (engine repo)**

```
git -C /path/to/engine add ledger/src/main/java/io/casehub/ledger/service/CaseLedgerEventCapture.java \
  ledger/src/main/java/io/casehub/ledger/service/WorkerDecisionEventCapture.java
git -C /path/to/engine commit -m "fix(#460): remove redundant entry.tenancyId assignment — save() is the sole writer (Refs #460)"
```

---

## Task 10: Engine — Remove tenancy_id from Join Table Migrations

**Files:**
- Modify: `ledger/src/main/resources/db/engine-ledger/migration/V2000__case_ledger_entry.sql` (casehub-engine)
- Modify: `ledger/src/main/resources/db/engine-ledger/migration/V2001__worker_decision_entry.sql` (casehub-engine)

- [ ] **Step 1: V2000 — remove tenancy_id column and index**

Rewrite V2000__case_ledger_entry.sql:
```sql
-- V2000: case_ledger_entry — CaseHub audit ledger
-- Extends ledger_entry (JOINED inheritance). V1000–V1004 are reserved by quarkus-ledger.

CREATE TABLE case_ledger_entry (
    id           UUID         NOT NULL,
    case_id      UUID         NOT NULL,
    command_type VARCHAR(100),
    event_type   VARCHAR(100),
    case_status  VARCHAR(50),
    CONSTRAINT pk_case_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_case_ledger_entry FOREIGN KEY (id) REFERENCES ledger_entry(id)
);

CREATE INDEX idx_cle_case_id ON case_ledger_entry (case_id);
```

- [ ] **Step 2: V2001 — same**

Rewrite V2001__worker_decision_entry.sql:
```sql
-- V2001: worker_decision_entry — per-worker capability decision records for trust scoring
-- Extends ledger_entry (JOINED inheritance). Written once per successful worker execution.

CREATE TABLE worker_decision_entry (
    id                       UUID             NOT NULL,
    worker_id                VARCHAR(255)     NOT NULL,
    capability_tag           VARCHAR(255),
    case_id                  UUID             NOT NULL,
    trust_score_at_routing   DOUBLE PRECISION,
    threshold_applied        DOUBLE PRECISION,
    CONSTRAINT pk_worker_decision_entry PRIMARY KEY (id),
    CONSTRAINT fk_worker_decision_entry FOREIGN KEY (id) REFERENCES ledger_entry(id)
);

CREATE INDEX idx_wde_case_id      ON worker_decision_entry (case_id);
CREATE INDEX idx_wde_worker_id    ON worker_decision_entry (worker_id);
CREATE INDEX idx_wde_capability   ON worker_decision_entry (capability_tag);
```

- [ ] **Step 3: Commit (engine repo)**

```
git -C /path/to/engine add ledger/src/main/resources/db/engine-ledger/migration/V2000__case_ledger_entry.sql \
  ledger/src/main/resources/db/engine-ledger/migration/V2001__worker_decision_entry.sql
git -C /path/to/engine commit -m "fix(#460): remove denormalized tenancy_id from join table migrations (Refs #460)"
```

---

## Task 11: Run Engine Tests

- [ ] **Step 1: Run engine-ledger module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl ledger -f /Users/mdproctor/claude/casehub/engine/pom.xml -q`
Expected: ALL PASS

- [ ] **Step 2: Fix any failures and commit**

If test failures occur (e.g., tests that set `cle.tenancyId` directly and verify the value), fix them. After the shadowing fix, `cle.tenancyId` sets the inherited base field — behaviour should be identical.

---

## Task 12: Protocol Update

**Files:**
- Modify: protocol file `ledger-subclass-extension.md` in garden

- [ ] **Step 1: Add shadowing guard rule to checklist**

Add to the checklist section:
```markdown
- [ ] Subclass entity does NOT redeclare any field that exists on `LedgerEntry` — Java field shadowing causes Hibernate persist failures with JOINED inheritance. Enforced at build time by `LedgerProcessor`.
```

- [ ] **Step 2: Commit (garden repo)**

```
git -C /path/to/garden add docs/protocols/casehub/ledger-subclass-extension.md
git -C /path/to/garden commit -m "protocol: add field-shadowing guard rule to ledger-subclass-extension"
```
