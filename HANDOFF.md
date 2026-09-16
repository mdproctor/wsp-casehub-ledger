# Handover — Slot 194

## Branch
- Clinical: `issue-171-mcpdomain-spi` (4 commits, not pushed)
- All other repos: `main`

## Active Issue
`casehubio/clinical#171` — migrate clinical to @McpDomain SPI.
Queue position 5/22.

## Session Summary

### Clinical #171 — @McpDomain SPI Migration (Phase 1)

Created 13 @McpDomain SPI interfaces with view records, Default implementations,
and APT-generated REST resources. Deleted 12 hand-written resource classes (-1,390 lines).

**SPI interfaces created (api/src/main/java/.../api/spi/):**
1. ClinicalTrialApi (5 methods) — list, register, get, updateSponsorConfig, activate
2. ClinicalSiteApi (2 methods) — addSite, getSite
3. ClinicalDeviationApi (2 methods) — reportDeviation, getDeviation
4. ClinicalGdprApi (1 method) — erasePatient
5. ClinicalVisitApi (4 methods) — create, list, get, update
6. ClinicalVitalApi (3 methods) — record, list, get
7. ClinicalLabResultApi (3 methods) — record, list, get
8. ClinicalStudyDrugApi (3 methods) — record, list, get
9. ClinicalMedicationApi (4 methods) — record, list, get, update
10. ClinicalCascadeApi (1 method) — getCascade
11. ClinicalAmendmentApi (4 methods) — list, propose, get, precedents
12. ClinicalPatientApi (5 methods) — list, enroll, get, screen, evaluateAndScreen
13. ClinicalAdverseEventApi (5 methods) — list, report, get, regrade, gradeHistory

**View records created (api/src/main/java/.../api/view/):**
27 view/request/response records extracted from inline resource records.

**Default implementations (runtime/src/main/java/.../service/api/):**
13 `@ApplicationScoped` CDI beans implementing the SPIs, delegating to existing services.

**APT generator wired:**
- Jandex index + `-parameters` flag added to api/pom.xml
- `casehub-platform-graphql-generator` annotation processor added to runtime/pom.xml
- Domain filter covers all 13 clinical domains
- All 13 `GeneratedClinical*Resource` classes generated and compile correctly

**Deleted (runtime/src/main/java/.../resource/):**
TrialResource, SiteResource, DeviationResource, GdprErasureResource,
VisitResource, VitalSignResource, LabResultResource, StudyDrugResource,
ConcomitantMedicationResource, ProtocolAmendmentResource, PatientResource,
CascadeResource

**Extracted:**
PatientComplianceResource — compliance endpoints (verifyLedger, withdrawConsent,
getAuditProv, getMerkleProof) extracted from PatientResource, stays hand-written
due to special content types (`application/ld+json`) and response codes (409).

### Pre-existing CDI Deployment Failures

33 CDI deployment problems prevent ALL `@QuarkusTest` classes from running.
Confirmed pre-existing (same failures on main without any migration changes).
Root cause: `InMemoryPlanItemStore` unsatisfied dependency in test context.
Likely caused by a SNAPSHOT dependency update since the last green test run.

## What's Next

1. **Fix pre-existing CDI failures** — investigate the 33 deployment problems,
   likely a SNAPSHOT update broke the `selected-alternatives` or `index-dependency`
   configuration in test `application.properties`
2. **Update tests** — resource tests reference old inline DTOs
   (`TrialResource.RegisterTrialRequest`, etc.) and expect entity responses instead of
   view records. Update imports and assertions to match the new `api/view/` types
3. **Migrate remaining resources:**
   - TrialDashboardResource (872 lines, 15 methods, ~15 DTOs) — largest piece
   - NarrativeResource — stateful cache + event bus subscription
   - EscalationPlanResource — returns `AdaptedPlan` from runtime, needs view type
   - DemoActionResource stays hand-written
4. **Push branch** to local remote once tests pass

## Standing Instructions
1. Always rebase from origin/main before starting work.
2. Slot-local `.m2` at `slots/194/.m2` — install platform there, not `~/.m2`.
3. Push slot clones to `local` remote first, then push from canonical local repos to GitHub.
4. The JDK 26 surefire fix is in `webapp/pom.xml` — apply to other casehub webapps if they hit the same hang.
5. `casehub-platform-graphql-generator` is the APT processor — version managed by slot .m2 SNAPSHOT.
