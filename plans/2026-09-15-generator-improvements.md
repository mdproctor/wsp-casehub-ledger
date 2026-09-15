# Generator Improvements Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/platform#300 — generator improvements (10 features)
**Issue group:** casehubio/platform#301–#310

**Goal:** Extend `GraphQLResolverProcessor` to produce fully-featured REST and GraphQL endpoints — including streaming, proper HTTP status codes, parameter naming, validation, authorization, OpenAPI, and pagination — so no casehub repo needs hand-written endpoints.

**Architecture:** Single-file APT processor (`GraphQLResolverProcessor.java`, ~1010 lines) plus 3 new annotations in `casehub-platform-api`. Each feature adds scan logic + generation logic to the existing processor. Tests use `google-testing-compile` for APT output verification.

**Tech Stack:** Java 21, Jandex, javax.annotation.processing APT, google-testing-compile, Mutiny `Multi<T>` (streaming)

## Global Constraints

- All new annotations in `io.casehub.platform.api.mcp` package
- Generator output packages: `io.casehub.platform.rest.generated` (REST), `io.casehub.platform.graphql.generated` (GraphQL)
- Annotation retention: `RUNTIME` (Jandex reads bytecode; APT reads source)
- Test pattern: `JavaFileObjects.forSourceString()` → `Compiler.javac().withProcessors()` → assert generated source content
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api,graphql-generator`
- Project repo: `/Users/mdproctor/claude/casehub/slots/194/platform`

---

## Batch 1: Quick wins — parameter names, OpenAPI, response codes

### Task 1: Parameter names + OpenAPI + response codes (#301, #307, #303)

**Files:**
- Modify: `graphql-generator/src/main/java/io/casehub/platform/graphql/generator/GraphQLResolverProcessor.java`
  - `generateResponseCode()` — add `isMutation` parameter, return 201 for mutations
  - `generateRestMethod()` — emit `@Operation(summary=...)` from description, pass `isMutation` to response code
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/RestStatus.java`
- Test: `graphql-generator/src/test/java/io/casehub/platform/graphql/generator/GraphQLResolverProcessorTest.java`

**Interfaces:**
- Consumes: existing `GraphQLResolverProcessor`, `PlatformQuery`, `PlatformMutation`
- Produces: `@RestStatus` annotation; updated `generateResponseCode()` with mutation awareness; `@Operation` on generated REST methods

- [ ] **Step 1: Write tests for response codes and OpenAPI**

```java
@Test
void responseWrapping_mutation_returns201() {
    assertThat(GraphQLResolverProcessor.generateResponseCode("String", "spi.create(arg0)", true, -1))
            .isEqualTo("return Response.status(201).entity(spi.create(arg0)).build();");
}

@Test
void responseWrapping_query_returns200() {
    assertThat(GraphQLResolverProcessor.generateResponseCode("String", "spi.get(arg0)", false, -1))
            .isEqualTo("return Response.ok(spi.get(arg0)).build();");
}

@Test
void responseWrapping_restStatusOverride() {
    assertThat(GraphQLResolverProcessor.generateResponseCode("String", "spi.create(arg0)", true, 202))
            .isEqualTo("return Response.status(202).entity(spi.create(arg0)).build();");
}

@Test
void generatedRestIncludesOperationSummary() throws Exception {
    var spi = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.DocApi",
            """
            package test;
            import io.casehub.platform.api.mcp.McpDomain;
            import io.casehub.platform.api.mcp.PlatformQuery;

            @McpDomain("docs")
            public interface DocApi {
                @PlatformQuery("List all documents")
                java.util.List<String> listDocs();
            }
            """);

    var compilation = com.google.testing.compile.Compiler.javac()
            .withProcessors(new GraphQLResolverProcessor())
            .withOptions("-AdomainFilter=docs", "-AgenerateGraphQL=false")
            .compile(spi);

    String content = compilation.generatedSourceFile(
            "io.casehub.platform.rest.generated.GeneratedDocsResource")
            .get().getCharContent(true).toString();
    assertThat(content).contains("@Operation(summary = \"List all documents\")");
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl graphql-generator -f /Users/mdproctor/claude/casehub/slots/194/platform/pom.xml -Dtest=GraphQLResolverProcessorTest`
Expected: compilation failures — `generateResponseCode` signature changed

- [ ] **Step 3: Create `@RestStatus` annotation**

Create `platform-api/src/main/java/io/casehub/platform/api/mcp/RestStatus.java`:
```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RestStatus {
    int value();
}
```

- [ ] **Step 4: Update `generateResponseCode()` signature and logic**

Change signature from `(String returnType, String delegateCall)` to
`(String returnType, String delegateCall, boolean isMutation, int restStatusOverride)`.

Update logic:
```java
static String generateResponseCode(String returnType, String delegateCall,
                                     boolean isMutation, int restStatusOverride) {
    if ("void".equals(returnType)) {
        return delegateCall + "; return Response.noContent().build();";
    }
    if (returnType.startsWith("Optional<")) {
        return "return " + delegateCall + ".map(v -> Response.ok(v).build()).orElse(Response.status(404).build());";
    }
    if (restStatusOverride > 0) {
        return "return Response.status(" + restStatusOverride + ").entity(" + delegateCall + ").build();";
    }
    if (isMutation) {
        return "return Response.status(201).entity(" + delegateCall + ").build();";
    }
    return "return Response.ok(" + delegateCall + ").build();";
}
```

Update the single call site in `generateRestMethod()` to pass `isMutation` and `restStatusOverride`.

- [ ] **Step 5: Add `@Operation` emission in `generateRestMethod()`**

Before the method signature line, add:
```java
if (!op.description().isEmpty()) {
    out.println("    @org.eclipse.microprofile.openapi.annotations.Operation(summary = \""
                + escapeJavaString(op.description()) + "\")");
}
```

- [ ] **Step 6: Add `@RestStatus` scanning**

Add `REST_STATUS_ANN` DotName constant. In both scan paths (Jandex + RoundEnv), read the annotation and store the value in `ResolvedOperation` as `int restStatusOverride` (default -1).

- [ ] **Step 7: Update existing `generateResponseCode` tests**

Update the 3 existing tests that call `generateResponseCode` to use the new 4-parameter signature (pass `false, -1` for backward compatibility).

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api,graphql-generator -f /Users/mdproctor/claude/casehub/slots/194/platform/pom.xml`
Expected: all tests pass

- [ ] **Step 9: Commit**

```
feat(#301,#303,#307): parameter names, response codes, OpenAPI summary

generateResponseCode now returns 201 for mutations, supports
@RestStatus override. REST methods emit @Operation(summary=...)
from @PlatformQuery/@PlatformMutation description.

Refs casehubio/platform#300
```

---

## Batch 2: Annotation handling — null→404, validation, RestName, RolesAllowed

### Task 2: Null→404 + validation pass-through + @RestName + @RolesAllowed (#302, #306, #308, #309)

**Files:**
- Modify: `GraphQLResolverProcessor.java`
  - `generateResponseCode()` — add `hasPathParam` for null→404
  - `generateRestMethod()` — emit constraint annotations, use restName, emit @RolesAllowed
  - `ResolvedParam` record — add `restName`, `constraintAnnotations`
  - `ResolvedOperation` record — add `rolesAllowed`, `restStatusOverride`
  - Scan methods — collect @RestName, @RolesAllowed, validation annotations
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/RestName.java`
- Test: `GraphQLResolverProcessorTest.java`

**Interfaces:**
- Consumes: Task 1's updated `generateResponseCode()`
- Produces: `@RestName` annotation; null→404 for `@PathParam` get-by-ID; constraint annotation pass-through; `@RolesAllowed` pass-through

- [ ] **Step 1: Write tests**

```java
@Test
void responseWrapping_pathParamNonCollection_nullChecks() {
    assertThat(GraphQLResolverProcessor.generateResponseCode("String", "spi.get(id)", false, -1, true))
            .contains("if (result == null) return Response.status(404).build();");
}

@Test
void generatedRestUsesRestNameForQueryParam() throws Exception {
    var spi = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.PageApi",
            """
            package test;
            import io.casehub.platform.api.mcp.*;

            @McpDomain("pages")
            public interface PageApi {
                @PlatformQuery("List pages")
                java.util.List<String> listPages(@RestName("page_size") Integer pageSize);
            }
            """);

    var compilation = com.google.testing.compile.Compiler.javac()
            .withProcessors(new GraphQLResolverProcessor())
            .withOptions("-AdomainFilter=pages", "-AgenerateGraphQL=false")
            .compile(spi);

    String content = compilation.generatedSourceFile(
            "io.casehub.platform.rest.generated.GeneratedPagesResource")
            .get().getCharContent(true).toString();
    assertThat(content).contains("@QueryParam(\"page_size\")");
}

@Test
void generatedRestPassesThroughRolesAllowed() throws Exception {
    var spi = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.SecureApi",
            """
            package test;
            import io.casehub.platform.api.mcp.*;
            import jakarta.annotation.security.RolesAllowed;

            @McpDomain("secure")
            public interface SecureApi {
                @PlatformQuery("Admin only")
                @RolesAllowed({"admin"})
                String getSecret();
            }
            """);

    var compilation = com.google.testing.compile.Compiler.javac()
            .withProcessors(new GraphQLResolverProcessor())
            .withOptions("-AdomainFilter=secure", "-AgenerateGraphQL=false")
            .compile(spi);

    String content = compilation.generatedSourceFile(
            "io.casehub.platform.rest.generated.GeneratedSecureResource")
            .get().getCharContent(true).toString();
    assertThat(content).contains("@RolesAllowed");
}
```

- [ ] **Step 2: Create `@RestName` annotation**

```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.PARAMETER)
@Retention(RetentionPolicy.RUNTIME)
public @interface RestName {
    String value();
}
```

- [ ] **Step 3: Extend `ResolvedParam` and `ResolvedOperation` records**

Add `String restName` to `ResolvedParam` (nullable — null means use Java param name).
Add `List<String> rolesAllowed` and `int restStatusOverride` to `ResolvedOperation`.

- [ ] **Step 4: Add scanning for @RestName, @RolesAllowed, validation annotations**

In both Jandex and RoundEnv scan paths:
- Read `@RestName` value from parameter → store in `ResolvedParam.restName`
- Read `@RolesAllowed` value array from method → store in `ResolvedOperation.rolesAllowed`
- Read `jakarta.validation.constraints.*` annotations from parameters → store as strings

- [ ] **Step 5: Update `generateRestMethod()` for all four features**

- Use `p.restName() != null ? p.restName() : p.name()` for `@QueryParam` name
- Emit `@RolesAllowed({"role1", "role2"})` before method if present
- Emit constraint annotations on parameters
- Pass `hasPathParam` to `generateResponseCode()`

- [ ] **Step 6: Update `generateResponseCode()` — add `hasPathParam` parameter**

```java
static String generateResponseCode(String returnType, String delegateCall,
                                     boolean isMutation, int restStatusOverride,
                                     boolean hasPathParam) {
    // ... existing void and Optional cases ...
    if (hasPathParam && !isCollectionType(returnType)) {
        return "var result = " + delegateCall + "; if (result == null) return Response.status(404).build(); "
               + (restStatusOverride > 0
                  ? "return Response.status(" + restStatusOverride + ").entity(result).build();"
                  : "return Response.ok(result).build();");
    }
    // ... existing mutation and default cases ...
}
```

- [ ] **Step 7: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api,graphql-generator -f /Users/mdproctor/claude/casehub/slots/194/platform/pom.xml`

- [ ] **Step 8: Commit**

```
feat(#302,#306,#308,#309): null-404, validation, RestName, RolesAllowed

get-by-ID methods with @PathParam now return 404 on null. Validation
constraint annotations pass through from SPI to generated endpoints.
@RestName overrides query param names. @RolesAllowed propagates to
generated REST and GraphQL.

Refs casehubio/platform#300
```

---

## Batch 3: Streaming — @PlatformStream + thread dispatch

### Task 3: @PlatformStream and thread dispatch awareness (#304, #305)

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/PlatformStream.java`
- Modify: `GraphQLResolverProcessor.java`
  - `OperationType` enum — add `STREAM`
  - New `PLATFORM_STREAM` DotName constant
  - Scan methods — detect `@PlatformStream`
  - `generateRestResourceSource()` — move `@RunOnVirtualThread` from class to method, skip for STREAM
  - `generateRestMethod()` — STREAM: `@Produces(SSE)`, `@RestStreamElementType(JSON)`, return `Multi<T>` directly
  - `generateMethod()` (GraphQL) — STREAM: `@Subscription` instead of `@Query`
- Test: `GraphQLResolverProcessorTest.java`

**Interfaces:**
- Consumes: Tasks 1-2's updated generator
- Produces: `@PlatformStream` annotation; SSE REST generation; `@Subscription` GraphQL generation; per-method `@RunOnVirtualThread`

- [ ] **Step 1: Write streaming tests**

```java
@Test
void streamingRestProducesSseEndpoint() throws Exception {
    var spi = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.EventApi",
            """
            package test;
            import io.casehub.platform.api.mcp.*;
            import io.smallrye.mutiny.Multi;

            @McpDomain("events")
            public interface EventApi {
                @PlatformStream("Real-time events")
                Multi<String> eventStream(java.util.UUID scopeId);
            }
            """);

    var compilation = com.google.testing.compile.Compiler.javac()
            .withProcessors(new GraphQLResolverProcessor())
            .withOptions("-AdomainFilter=events", "-AgenerateGraphQL=false")
            .compile(spi);

    String content = compilation.generatedSourceFile(
            "io.casehub.platform.rest.generated.GeneratedEventsResource")
            .get().getCharContent(true).toString();
    assertThat(content).contains("@Produces(MediaType.SERVER_SENT_EVENTS)");
    assertThat(content).contains("@RestStreamElementType(MediaType.APPLICATION_JSON)");
    assertThat(content).contains("public Multi<String> eventStream(");
    assertThat(content).doesNotContain("Response.ok");
    assertThat(content).doesNotContain("RunOnVirtualThread");
}

@Test
void streamingGraphqlProducesSubscription() throws Exception {
    var spi = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.EventApi",
            """
            package test;
            import io.casehub.platform.api.mcp.*;
            import io.smallrye.mutiny.Multi;

            @McpDomain("events")
            public interface EventApi {
                @PlatformStream("Real-time events")
                Multi<String> eventStream(java.util.UUID scopeId);
            }
            """);

    var compilation = com.google.testing.compile.Compiler.javac()
            .withProcessors(new GraphQLResolverProcessor())
            .withOptions("-AdomainFilter=events", "-AgenerateRest=false")
            .compile(spi);

    String content = compilation.generatedSourceFile(
            "io.casehub.platform.graphql.generated.GeneratedEventsResolver")
            .get().getCharContent(true).toString();
    assertThat(content).contains("@Subscription");
    assertThat(content).doesNotContain("@Query");
    assertThat(content).contains("public Multi<String> eventStream(");
}

@Test
void nonStreamingRestStillHasRunOnVirtualThread() throws Exception {
    var spi = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.ItemApi",
            """
            package test;
            import io.casehub.platform.api.mcp.*;

            @McpDomain("items")
            public interface ItemApi {
                @PlatformQuery("Get item")
                String getItem(@PathParam java.util.UUID id);
            }
            """);

    var compilation = com.google.testing.compile.Compiler.javac()
            .withProcessors(new GraphQLResolverProcessor())
            .withOptions("-AdomainFilter=items", "-AgenerateGraphQL=false")
            .compile(spi);

    String content = compilation.generatedSourceFile(
            "io.casehub.platform.rest.generated.GeneratedItemsResource")
            .get().getCharContent(true).toString();
    assertThat(content).contains("@RunOnVirtualThread");
}
```

- [ ] **Step 2: Create `@PlatformStream` annotation**

```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PlatformStream {
    String value() default "";
}
```

- [ ] **Step 3: Add `STREAM` to OperationType and scan logic**

Add `PLATFORM_STREAM` DotName constant. In both scan paths, detect `@PlatformStream` alongside `@PlatformQuery`/`@PlatformMutation`. Set `OperationType.STREAM`.

- [ ] **Step 4: Move `@RunOnVirtualThread` from class level to method level**

In `generateRestResourceSource()`: remove `@RunOnVirtualThread` from class-level annotations (line 664). In `generateRestMethod()`: emit `@RunOnVirtualThread` only for `QUERY` and `MUTATION` operations, not for `STREAM`.

- [ ] **Step 5: Implement REST streaming generation**

In `generateRestMethod()`, when `op.type() == OperationType.STREAM`:
```java
out.println("    @GET");
out.println("    @Path(\"" + pathSuffix + "\")");
out.println("    @Produces(MediaType.SERVER_SENT_EVENTS)");
out.println("    @RestStreamElementType(MediaType.APPLICATION_JSON)");
// method returns Multi<T> directly, no Response wrapper
out.println("    public " + op.returnTypeStr() + " " + op.methodName() + "(" + params + ") {");
out.println("        return " + fieldName + "." + op.methodName() + "(" + args + ");");
out.println("    }");
```

Add imports: `org.jboss.resteasy.reactive.RestStreamElementType`

- [ ] **Step 6: Implement GraphQL streaming generation**

In `generateMethod()`, when `op.type() == OperationType.STREAM`:
- Use `@Subscription` instead of `@Query`
- Add import: `io.smallrye.graphql.api.Subscription`

- [ ] **Step 7: Add `@PlatformStream` to `getSupportedOptions()`**

No change needed — the processor scans annotations via Jandex/RoundEnv, not via `getSupportedAnnotationTypes`.

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api,graphql-generator -f /Users/mdproctor/claude/casehub/slots/194/platform/pom.xml`

- [ ] **Step 9: Commit**

```
feat(#304,#305): @PlatformStream for SSE + GraphQL subscriptions

New @PlatformStream annotation for Multi<T> streaming methods.
REST: @Produces(SSE) + @RestStreamElementType(JSON), returns Multi<T>
directly. GraphQL: @Subscription. Thread dispatch moved to method
level — STREAM methods skip @RunOnVirtualThread.

Refs casehubio/platform#300
```

---

## Batch 4: Pagination

### Task 4: Pagination response wrapping (#310)

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/PaginatedResponse.java`
- Modify: `GraphQLResolverProcessor.java`
  - New `PAGINATED_RESPONSE_ANN` DotName constant
  - Scan: detect `@PaginatedResponse` on methods
  - `ResolvedOperation` — add `boolean paginated`
  - `generateRestMethod()` — emit pagination header wrapping
- Test: `GraphQLResolverProcessorTest.java`

**Interfaces:**
- Consumes: Tasks 1-3's updated generator
- Produces: `@PaginatedResponse` annotation; generated REST methods with `X-Total-Count` header

- [ ] **Step 1: Write pagination test**

```java
@Test
void paginatedResponseAddsHeaders() throws Exception {
    var spi = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.ListApi",
            """
            package test;
            import io.casehub.platform.api.mcp.*;

            @McpDomain("lists")
            public interface ListApi {
                @PlatformQuery("List items")
                @PaginatedResponse
                test.ItemPage listItems(Integer offset, Integer limit);
            }
            """);

    var pageType = com.google.testing.compile.JavaFileObjects.forSourceString(
            "test.ItemPage",
            """
            package test;
            public record ItemPage(java.util.List<String> items, int totalCount, boolean hasMore) {}
            """);

    var compilation = com.google.testing.compile.Compiler.javac()
            .withProcessors(new GraphQLResolverProcessor())
            .withOptions("-AdomainFilter=lists", "-AgenerateGraphQL=false")
            .compile(spi, pageType);

    String content = compilation.generatedSourceFile(
            "io.casehub.platform.rest.generated.GeneratedListsResource")
            .get().getCharContent(true).toString();
    assertThat(content).contains("X-Total-Count");
}
```

- [ ] **Step 2: Create `@PaginatedResponse` annotation**

```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PaginatedResponse {
    String totalCountMethod() default "totalCount";
    String hasMoreMethod() default "hasMore";
}
```

- [ ] **Step 3: Add scanning and generation for pagination**

Scan: detect `@PaginatedResponse`, store in `ResolvedOperation.paginated`.

Generate: for paginated methods, wrap the response:
```java
var page = spi.listItems(offset, limit);
return Response.ok(page)
    .header("X-Total-Count", String.valueOf(page.totalCount()))
    .build();
```

GraphQL: no change — return the page object directly.

- [ ] **Step 4: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl platform-api,graphql-generator -f /Users/mdproctor/claude/casehub/slots/194/platform/pom.xml`

- [ ] **Step 5: Commit**

```
feat(#310): @PaginatedResponse for pagination headers

New @PaginatedResponse annotation. Generated REST methods add
X-Total-Count header from page object's totalCount() method.
GraphQL returns the page object directly.

Refs casehubio/platform#300
```

---

## Batch 5: Install and verify

### Task 5: Install and verify with ledger

- [ ] **Step 1: Install platform to slot .m2**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -f /Users/mdproctor/claude/casehub/slots/194/platform/pom.xml -DskipTests`

- [ ] **Step 2: Rebuild ledger to verify improved output**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl api,rest,graphql -f /Users/mdproctor/claude/casehub/slots/194/ledger/pom.xml`
Check generated sources for: `@Operation`, proper param names, 201 on mutations, `@RunOnVirtualThread` on methods.

- [ ] **Step 3: Commit verification result**

Note: ledger SPI may need updates (add `@RestStatus`, `@PaginatedResponse` etc.) — that's separate work for the ledger migration cleanup.

---

## References

- [specs/issue-300-generator-improvements/2026-09-15-generator-improvements-design.md] — design spec
- [graphql-generator/src/main/java/.../GraphQLResolverProcessor.java] — the generator (~1010 lines)
- [graphql-generator/src/test/java/.../GraphQLResolverProcessorTest.java] — existing tests (22 tests)
- [platform-api/src/main/java/.../mcp/PlatformQuery.java] — annotation pattern to follow
- casehubio/platform#300 — parent epic
- casehubio/platform#301–#310 — child issues
