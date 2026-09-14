# Unified API Generation — Migrate Ledger to @McpDomain SPI

**Issue:** casehubio/ledger#207
**Parent epic:** casehubio/platform#295 (CLOSED — platform generator done)
**Date:** 2026-09-14

## Summary

Replace hand-written REST endpoints and GraphQL resolvers in `casehub-ledger`
with generated code from the platform `graphql-generator` APT. Define SPI
interfaces with `@McpDomain` + `@PlatformQuery`/`@PlatformMutation` annotations.
The generator produces both JAX-RS REST resources and SmallRye GraphQL resolvers
from the same interface. One SPI → three API surfaces (REST, GraphQL, MCP), zero
hand-written endpoints, zero drift.

## Annotation Model

All SPI interfaces use platform-specific annotations from
`io.casehub.platform.api.mcp` — never JAX-RS annotations on the SPI interface.
This avoids the Quarkus REST server resource registration conflict
(GE-20260612-4f9a47).

```java
@McpDomain("ledger/entries")
public interface LedgerEntryApi {

    @PlatformQuery("Get a ledger entry by ID")
    LedgerEntryView getEntry(@PathParam UUID id, String tenancyId);

    @PlatformMutation("Append a new ledger entry")
    LedgerEntryView appendEntry(AppendEntryRequest request);
}
```

- `@McpDomain` — hierarchical domain label for MCP progressive discovery
- `@PlatformQuery("description")` → generates `@GET` REST + `@Query` GraphQL
- `@PlatformMutation("description")` → generates `@POST` REST + `@Mutation` GraphQL
- `@PathParam` — marks path segment parameters (generates `/{name}` in REST path)
- `@RestMethod(HttpMethod.PUT/DELETE)` — overrides default HTTP verb
- Non-`@PathParam` simple types → `@QueryParam` in generated REST
- Single complex non-`@PathParam` parameter → `@Valid` request body

REST paths are derived from kebab-cased method names (e.g. `getEntry` →
`/get-entry/{id}`). Pre-release, no consumers — path changes are acceptable.

## SPI Interfaces

Four interfaces matching the current REST resource grouping:

### LedgerEntryApi (`ledger/entries`)

| Method | Type | Description | Parameters |
|---|---|---|---|
| `listEntries` | Query | List entries by subject/actor | `UUID subjectId`, `UUID actorId`, `String tenancyId`, `Instant from`, `Instant to` |
| `getEntry` | Query | Get entry by ID | `@PathParam UUID id`, `String tenancyId` |
| `getCausedBy` | Query | Causal chain for an entry | `@PathParam UUID id`, `String tenancyId` |
| `appendEntry` | Mutation | Append a new entry | `AppendEntryRequest request`, `String tenancyId` |

### LedgerAttestationApi (`ledger/attestations`)

| Method | Type | Description | Parameters |
|---|---|---|---|
| `listAttestations` | Query | Attestations for an entry | `@PathParam UUID entryId`, `String tenancyId`, `String capabilityTag` |
| `createAttestation` | Mutation | Create an attestation | `CreateAttestationRequest request`, `String tenancyId` |

### LedgerVerificationApi (`ledger/verification`)

| Method | Type | Description | Parameters |
|---|---|---|---|
| `verify` | Query | Verify Merkle integrity | `UUID subjectId`, `String tenancyId` |
| `inclusionProof` | Query | Merkle inclusion proof | `@PathParam UUID entryId`, `String tenancyId` |

### LedgerTrustApi (`ledger/trust`)

| Method | Type | Description | Parameters |
|---|---|---|---|
| `trustScore` | Query | Global trust score | `@PathParam String actorId` |
| `capabilityScore` | Query | Capability-scoped trust | `@PathParam String actorId`, `@PathParam String capabilityTag` |
| `routingProfile` | Query | Composite routing profile | `@PathParam String actorId`, `@PathParam String capabilityTag` |

### Return Types (in `api/`)

New plain records in `api/src/main/java/io/casehub/ledger/api/view/`:

| Record | Fields | Replaces |
|---|---|---|
| `LedgerEntryView` | id, subjectId, actorId, actorType, entryType, sequenceNumber, occurredAt, leafHash, traceId, causedByEntryId, metadata | `LedgerEntryResponse` (REST), `LedgerEntryType` (GraphQL) |
| `LedgerEntryPage` | entries (List), totalCount, hasMore | `LedgerEntryPage` (GraphQL) |
| `AttestationView` | id, entryId, attestorId, verdict, confidence, capabilityTag, comment, occurredAt | `AttestationResponse` (REST), `LedgerAttestationType` (GraphQL) |
| `VerificationView` | subjectId, verified, treeSize, treeRoot | `VerificationResponse` (REST), `MerkleVerificationType` (GraphQL) |
| `InclusionProofView` | entryId, leafHash, treeSize, proofSteps | `InclusionProofResponse` (REST) |
| `TrustScoreView` | actorId, globalScore, alpha, beta, decisionCount | `TrustScoreResponse` (REST), `TrustScoreType` (GraphQL) |
| `CapabilityScoreView` | actorId, capabilityTag, score, alpha, beta, decisionCount | `CapabilityScoreResponse` (REST), `TrustCapabilityScoreType` (GraphQL) |
| `TrustRoutingProfileView` | actorId, globalScore, capabilityScore, decisionCount | `TrustRoutingProfileType` (GraphQL) |

### Request Types (in `api/`)

New records in `api/src/main/java/io/casehub/ledger/api/view/`:

| Record | Fields | Used by |
|---|---|---|
| `AppendEntryRequest` | subjectId, actorId, actorType, entryType, metadata, domainData | `LedgerEntryApi.appendEntry` |
| `CreateAttestationRequest` | entryId, attestorId, verdict, confidence, capabilityTag, comment | `LedgerAttestationApi.createAttestation` |

## Service Implementations

Four `@ApplicationScoped` CDI beans in `runtime/service/api/`:

| Bean | Implements | Injects |
|---|---|---|
| `DefaultLedgerEntryApi` | `LedgerEntryApi` | `LedgerEntryRepository`, `LedgerAppender` |
| `DefaultLedgerAttestationApi` | `LedgerAttestationApi` | `LedgerEntryRepository`, `OutcomeRecorder` |
| `DefaultLedgerVerificationApi` | `LedgerVerificationApi` | `LedgerVerificationService` |
| `DefaultLedgerTrustApi` | `LedgerTrustApi` | `TrustScoreSource` |

Each bean:
- Delegates to existing services — no new business logic
- Maps between entity/service types and `api/view/` return types
- Handles tenancy defaulting (via `LedgerRestUtil.defaultTenancyId` pattern)
- Is `@DefaultBean` so consumers can override with their own implementations

## APT Configuration

### rest/pom.xml

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

### graphql/pom.xml

Same `annotationProcessorPaths`, with:
```xml
<compilerArgs>
  <arg>-AdomainFilter=ledger/entries,ledger/attestations,ledger/verification,ledger/trust</arg>
  <arg>-AgenerateRest=false</arg>
</compilerArgs>
```

### Dependencies

Both modules depend on `api/` only (compile scope):
```xml
<dependency>
  <groupId>io.casehub</groupId>
  <artifactId>casehub-ledger-api</artifactId>
</dependency>
```

Runtime resolution: consumers provide `casehub-ledger` (runtime module) which
supplies the `DefaultXxxApi` CDI beans. Generated code injects the SPI
interface → CDI resolves to runtime implementation.

## Module Dependency Graph

```
api/  ←────── rest/   (compile: SPI interfaces + view records)
  │           ↑ APT: graphql-generator (generateGraphQL=false)
  │
  ├────────── graphql/ (compile: SPI interfaces + view records)
  │           ↑ APT: graphql-generator (generateRest=false)
  │
  └────────── runtime/ (compile: SPI interfaces; implements DefaultXxxApi beans)
              ↑ injects: LedgerEntryRepository, TrustScoreSource, etc.
```

## What Gets Deleted

### rest/ module
- `LedgerEntryResource.java` — replaced by generated `GeneratedLedgerEntriesResource`
- `AttestationResource.java` — replaced by generated `GeneratedLedgerAttestationsResource`
- `MerkleVerificationResource.java` — replaced by generated `GeneratedLedgerVerificationResource`
- `TrustScoreResource.java` — replaced by generated `GeneratedLedgerTrustResource`
- `dto/LedgerEntryResponse.java` — replaced by `api/view/LedgerEntryView`
- `dto/AttestationResponse.java` — replaced by `api/view/AttestationView`
- `dto/CreateAttestationRequest.java` — replaced by `api/view/CreateAttestationRequest`
- `dto/InclusionProofResponse.java` — replaced by `api/view/InclusionProofView`
- `dto/TrustScoreResponse.java` — replaced by `api/view/TrustScoreView`
- `dto/VerificationResponse.java` — replaced by `api/view/VerificationView`
- `dto/LedgerDtoMapper.java` — mapping moves into `DefaultXxxApi` beans

### graphql/ module
- `LedgerQueryResolver.java` — replaced by generated resolvers
- `LedgerMutationResolver.java` — replaced by generated resolvers
- `dto/LedgerEntryType.java` — replaced by `api/view/LedgerEntryView`
- `dto/LedgerAttestationType.java` — replaced by `api/view/AttestationView`
- `dto/TrustScoreType.java` — replaced by `api/view/TrustScoreView`
- `dto/TrustCapabilityScoreType.java` — replaced by `api/view/CapabilityScoreView`
- `dto/TrustRoutingProfileType.java` — replaced by `api/view/TrustRoutingProfileView`
- `dto/MerkleVerificationType.java` — replaced by `api/view/VerificationView`
- `dto/LedgerEntryPage.java` — replaced by `api/view/LedgerEntryPage`
- `dto/LedgerEntryFilterInput.java` — filter params become method parameters
- `dto/AppendLedgerEntryInput.java` — replaced by `api/view/AppendEntryRequest`
- `dto/CreateAttestationInput.java` — replaced by `api/view/CreateAttestationRequest`

### Kept (not generated)
- `LedgerExceptionMapper.java` — cross-cutting `@Provider`
- `LedgerNotFoundException.java` — exception type
- `LedgerRestUtil.java` — shared utility
- `LedgerModelEnricher.java` — MCP infrastructure, not an endpoint

## Tenancy Handling

All SPI methods take explicit `String tenancyId` as a separate method parameter
(not inside request records). For queries, the generator makes it a
`@QueryParam("tenancyId")`. For mutations with a request body, the generator
treats it as a `@QueryParam` (it's a simple `String`, not the complex body type).
Both surfaces expose tenancyId consistently as a query parameter.

In GraphQL, tenancyId appears as a method argument on every operation.

Service implementations default it via the existing
`LedgerRestUtil.defaultTenancyId()` pattern when null.

This matches CLAUDE.md: "tenancyId is an explicit String parameter on every
tenant-scoped SPI method."

## Testing Strategy

1. **API module tests** — pure JUnit: verify view records are complete, request
   validation works
2. **Runtime service tests** — `@QuarkusTest` with in-memory repositories:
   verify `DefaultXxxApi` beans delegate correctly, return correct view types,
   handle null tenancyId
3. **Generated endpoint tests** — update existing REST and GraphQL tests to hit
   generated endpoints (new paths, same semantics)
4. **MCP dispatch test** — verify generated resolvers are discoverable by
   `GraphQLModelScanner` (technique from GE-20260818-c2f072)

## References

- casehubio/platform#295 — parent epic
- casehubio/ledger#207 — this issue
- `GraphQLResolverProcessor.java` (platform) — the code generator
- `CallbackApi.java`, `NotificationPreferenceApi.java` (platform) — migration pattern
- GE-20260914-3854b8 — APT domain filtering essential for multi-module
- GE-20260816-d18a02 — GraphQL modules depend on API, not runtime
- GE-20260810-31134a — casehub-ledger-rest already provides endpoints
- GE-20260827-50b67b — wrap external types in local records for schema isolation
- GE-20260827-d453bd — sealed interfaces cause SmallRye GraphQL failure
- GE-20260818-c2f072 — testing MCP dispatch without CDI
- GE-20260612-4f9a47 — JAX-RS @Path on interface causes server resource conflict
- GE-20260616-17187e — TrustGateService delegates via TrustScoreSource SPI
