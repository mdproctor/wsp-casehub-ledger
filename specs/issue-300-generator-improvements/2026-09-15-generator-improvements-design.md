# Generator Improvements — 10 Features for Full API Generation

**Issue:** casehubio/platform#300
**Date:** 2026-09-15

## Summary

Ten improvements to `GraphQLResolverProcessor` in `casehub-platform-graphql-generator`
to support full API generation across all casehub repos — including streaming. After
these changes, no repo needs hand-written REST or GraphQL endpoints for standard
patterns (query, mutation, streaming).

## Target File

`graphql-generator/src/main/java/io/casehub/platform/graphql/generator/GraphQLResolverProcessor.java`
(~1010 lines currently, single-file generator)

## New Platform Annotations

All in `io.casehub.platform.api.mcp` (existing package in `casehub-platform-api`):

| Annotation | Target | Purpose |
|---|---|---|
| `@PlatformStream("description")` | Method | Marks a streaming method returning `Multi<T>` |
| `@RestName("page_size")` | Parameter | Overrides query/path param name in generated REST |
| `@RestStatus(201)` | Method | Overrides HTTP status code in generated REST |

Existing annotations unchanged: `@McpDomain`, `@PlatformQuery`, `@PlatformMutation`,
`@PathParam`, `@RestMethod`, `@RestPath`.

## Improvements

### 1. Parameter Names (#301)

**Problem:** Jandex path (line 303) produces `arg0` when bytecode lacks parameter names.
**Fix:** Two-part:
- a) Document: API modules MUST compile with `<parameters>true</parameters>` in
  maven-compiler-plugin. Jandex reads bytecode parameter names when present.
- b) Generator: use `@PathParam(value="name")` for path params (already works),
  use `@RestName("name")` for query params (new annotation, see #8).
- c) Fallback: if neither is available, keep `arg` + index but emit a WARNING.

### 2. Null → 404 for Get-by-ID (#302)

**Problem:** Non-Optional, non-collection returns produce `Response.ok(null)` → 200.
**Fix:** In `generateResponseCode()` (line 961), add a new case:
- If method has `@PathParam` AND return type is non-collection, non-void, non-Optional:
  ```java
  var result = spi.getEntry(id, tenancyId);
  if (result == null) return Response.status(404).build();
  return Response.ok(result).build();
  ```
- Pass `hasPathParam` flag from `generateRestMethod()` to `generateResponseCode()`.
- `Optional<T>` already maps to 404 — no change needed there.

### 3. Response Codes (#303)

**Problem:** Mutations always return 200. Creations should return 201.
**Fix:**
- `@PlatformMutation` → `Response.status(201).entity(result).build()` (was `Response.ok()`)
- `void` mutation → 204 (already works)
- New `@RestStatus(N)` annotation overrides any convention
- `Optional.empty()` → 404 (already works)

In `generateResponseCode()`, add `isMutation` and `restStatusOverride` parameters.

### 4. Thread Dispatch Awareness (#304)

**Problem:** Generator hardcodes `@RunOnVirtualThread` on class level (line 664).
Streaming endpoints (`Multi<T>`) should NOT run on virtual threads.
**Fix:**
- Move `@RunOnVirtualThread` from class level to method level
- Only emit it for `QUERY` and `MUTATION` operations
- `STREAM` operations: no dispatch annotation (reactive, stays on event loop)
- New `@Blocking` pass-through: if present on SPI method, emit `@Blocking` on generated method

### 5. Streaming — `@PlatformStream` (#305)

**Problem:** No streaming support. SSE and GraphQL subscriptions hand-written.
**Fix:**
- New `OperationType.STREAM` in the enum (line 972)
- New `PLATFORM_STREAM` DotName constant
- Scan logic: detect `@PlatformStream` alongside `@PlatformQuery`/`@PlatformMutation`
- REST generator for STREAM:
  ```java
  @GET
  @Path("/case-events/{caseId}")
  @Produces(MediaType.SERVER_SENT_EVENTS)
  @RestStreamElementType(MediaType.APPLICATION_JSON)
  public Multi<CaseEvent> caseEvents(@PathParam("caseId") UUID caseId) {
      return caseStreamApi.caseEvents(caseId);
  }
  ```
  No `Response` wrapper. No `@RunOnVirtualThread`. Return `Multi<T>` directly.
- GraphQL generator for STREAM:
  ```java
  @Subscription
  @Description("Real-time case lifecycle events")
  public Multi<CaseEvent> caseEvents(@Name("caseId") UUID caseId) {
      return caseStreamApi.caseEvents(caseId);
  }
  ```
  Uses `@Subscription` instead of `@Query`.
- Import additions: `io.smallrye.mutiny.Multi`, `org.jboss.resteasy.reactive.RestStreamElementType`,
  `io.smallrye.graphql.api.Subscription`

### 6. Validation Annotation Pass-Through (#306)

**Problem:** Generator hardcodes `@Valid` on first complex param. No constraint annotations.
**Fix:**
- In `ResolvedParam`, add `List<AnnotationInfo> constraintAnnotations`
- During scan: collect all `jakarta.validation.constraints.*` and custom constraint
  annotations from SPI method parameters
- During generation: emit each constraint annotation on the generated parameter
- Keep `@Valid` on complex body params (existing behavior)
- Jandex path: read from `AnnotationInstance` on parameter targets
- RoundEnv path: read from `AnnotationMirror` on `VariableElement`

### 7. OpenAPI `@Operation` from Description (#307)

**Problem:** Generated REST has no API documentation.
**Fix:**
- In `generateRestMethod()`, if `op.description()` is non-empty:
  ```java
  out.println("    @org.eclipse.microprofile.openapi.annotations.Operation(summary = \""
              + escapeJavaString(op.description()) + "\")");
  ```
- Add import: `org.eclipse.microprofile.openapi.annotations.Operation`
- GraphQL already emits `@Description` — no change needed there.

### 8. Query Param Name Override — `@RestName` (#308)

**Problem:** Query param names always match Java parameter names.
**Fix:**
- New `@RestName("page_size")` annotation in `casehub-platform-api`
- New `REST_NAME_ANN` DotName constant
- In scan: read `@RestName` value, store in `ResolvedParam.restName`
- In `generateRestMethod()`: use `restName` if present, else `paramName`
  ```java
  @QueryParam("page_size") Integer pageSize
  ```

### 9. `@RolesAllowed` Pass-Through (#309)

**Problem:** Authorization must be on service impl, not generated endpoint.
**Fix:**
- In `ResolvedOperation`, add `List<String> rolesAllowed`
- During scan: read `@RolesAllowed` from SPI method (Jandex + RoundEnv)
- During REST generation: emit `@RolesAllowed({"role1", "role2"})` on method
- During GraphQL generation: emit same (SmallRye GraphQL respects it)
- Add import: `jakarta.annotation.security.RolesAllowed`

### 10. Pagination Response Wrapping (#310)

**Problem:** No pagination support — consumers hand-write Link headers.
**Fix:**
- New `@PaginatedResponse` annotation on SPI methods that return paginated results
- Generator detects it and wraps the response:
  ```java
  var page = spi.listEntries(filter, offset, limit);
  return Response.ok(page)
      .header("X-Total-Count", String.valueOf(page.totalCount()))
      .link(buildNextLink(offset, limit, page.totalCount()), "next")
      .build();
  ```
- Convention: the return type must have `totalCount()` and `hasMore()` methods
  (detected via Jandex/reflection)
- The annotation carries `offsetParam` and `limitParam` names for Link header construction
- REST only — GraphQL returns the page object directly (no HTTP headers)

## Implementation Order

Dependencies between features:
- #4 (thread dispatch) MUST ship with #5 (streaming) — streaming breaks without it
- #1 (param names) should ship before #8 (RestName override) — base case before override
- All others are independent

Recommended order (batch by dependency):
1. **Batch 1:** #1 (param names) + #7 (OpenAPI) + #3 (response codes) — quick wins
2. **Batch 2:** #2 (null→404) + #6 (validation) + #8 (RestName) + #9 (RolesAllowed) — annotation handling
3. **Batch 3:** #4 (thread dispatch) + #5 (streaming) — must be together
4. **Batch 4:** #10 (pagination) — most complex, builds on everything else

## Testing Strategy

Each improvement gets:
1. Unit test in `graphql-generator` — verify generated source output contains expected annotations/code
2. Integration test — compile generated code, verify runtime behavior (SSE, status codes, etc.)
3. Ledger re-migration — after all 10, re-run ledger with the improved generator to verify end-to-end

## References

- `GraphQLResolverProcessor.java` (platform) — the generator
- `casehubio/ledger#207` — first consumer, exposed the gaps
- Engine SSE resources — `CaseStreamResource.java`, `ExecutionStateResource.java` — reference for streaming
- Engine GraphQL subscriptions — `CaseSubscriptionResolver.java` — reference for `@Subscription`
