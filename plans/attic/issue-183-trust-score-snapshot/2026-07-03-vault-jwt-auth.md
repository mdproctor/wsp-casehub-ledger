# Vault JWT Auth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use hortora:subagent-driven-development (recommended) or hortora:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add JWT-based Vault authentication and consolidate KubernetesVaultTokenSource into JwtVaultTokenSource.

**Architecture:** `JwtVaultTokenSource` extends `LoginBasedVaultTokenSource` with a `Supplier<String>` for JWT acquisition. `KubernetesVaultTokenSource` is deleted — both Kubernetes and JWT auth are served by `JwtVaultTokenSource` with different mount paths. The Quarkus adapter gains `AuthMethod.JWT` and updates `KUBERNETES` to use the consolidated class.

**Tech Stack:** Java 21, Quarkus 3.32.2, WireMock (tests), JUnit 5, AssertJ

## Global Constraints

- Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build signing modules: `mvn test -Pwith-signing`
- Run a specific signing module: `mvn test -pl signing/vault-transit -Pwith-signing`
- Use `mvn` not `./mvnw`
- Pure Java module (`vault-transit`) has zero framework dependencies — no Quarkus, no CDI
- Test pattern: WireMock for HTTP stubs, `Clock.fixed()` for time control

---

### Task 1: JwtVaultTokenSource — TDD in pure Java module

**Files:**
- Create: `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/JwtVaultTokenSourceTest.java`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/JwtVaultTokenSource.java`

**Interfaces:**
- Consumes: `LoginBasedVaultTokenSource` (abstract base — `loginPath()`, `loginRequestBody()`)
- Produces: `JwtVaultTokenSource(String vaultAddress, String role, Supplier<String> jwtSupplier, String mountPath, HttpClient http, ObjectMapper mapper, Clock clock)`, static factory `JwtVaultTokenSource.fromFile(String vaultAddress, String role, Path jwtPath, String mountPath, HttpClient http, ObjectMapper mapper, Clock clock)`

- [ ] **Step 1: Write failing test — Supplier-based login**

Create `JwtVaultTokenSourceTest.java`:

```java
package io.casehub.ledger.signing.vault;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.net.http.HttpClient;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.tomakehurst.wiremock.WireMockServer;

class JwtVaultTokenSourceTest {

    static WireMockServer wireMock;
    static final ObjectMapper mapper = new ObjectMapper();
    static final HttpClient http = HttpClient.newHttpClient();

    @TempDir
    Path tempDir;

    @BeforeAll
    static void startWireMock() {
        wireMock = new WireMockServer(0);
        wireMock.start();
    }

    @AfterAll
    static void stopWireMock() {
        wireMock.stop();
    }

    @BeforeEach
    void resetWireMock() {
        wireMock.resetAll();
    }

    private static String loginResponse(final String token, final int leaseDuration) {
        return "{\"auth\":{\"client_token\":\"" + token
                + "\",\"lease_duration\":" + leaseDuration + ",\"renewable\":true}}";
    }

    @Test
    void token_logsInWithSupplierJwt() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.jwt-token", 3600))));

        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", () -> "eyJhbGciOiJSUzI1NiJ9.test",
                "jwt", http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.jwt-token");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/jwt/login"))
                .withRequestBody(containing("eyJhbGciOiJSUzI1NiJ9.test"))
                .withRequestBody(containing("my-role")));
    }

    @Test
    void token_fromFile_readsJwtFromFile() throws IOException {
        final Path jwtFile = tempDir.resolve("token.jwt");
        Files.writeString(jwtFile, "eyJhbGciOiJSUzI1NiJ9.from-file");

        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.file-token", 3600))));

        final var source = JwtVaultTokenSource.fromFile(
                wireMock.baseUrl(), "my-role", jwtFile, "jwt",
                http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.file-token");
        wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/jwt/login"))
                .withRequestBody(containing("eyJhbGciOiJSUzI1NiJ9.from-file")));
    }

    @Test
    void token_returnsCachedWithinTTL() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.cached", 3600))));

        final Instant now = Instant.parse("2026-07-03T12:00:00Z");
        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", () -> "jwt-value",
                "jwt", http, mapper, Clock.fixed(now, ZoneOffset.UTC));

        assertThat(source.token()).isEqualTo("hvs.cached");
        assertThat(source.token()).isEqualTo("hvs.cached");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/jwt/login")));
    }

    @Test
    void token_reLoginsAfterInvalidate() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.original", 3600))));

        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", () -> "jwt-value",
                "jwt", http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.original");

        wireMock.resetAll();
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.refreshed", 3600))));

        source.invalidate();
        assertThat(source.token()).isEqualTo("hvs.refreshed");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/jwt/login")));
    }

    @Test
    void token_throwsOnLoginFailure() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(aResponse().withStatus(403)));

        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", () -> "bad-jwt",
                "jwt", http, mapper, Clock.systemUTC());

        assertThatThrownBy(source::token)
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("403");
    }

    @Test
    void token_usesCustomMountPath() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/oidc/login"))
                .willReturn(okJson(loginResponse("hvs.oidc", 3600))));

        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", () -> "jwt-value",
                "oidc", http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.oidc");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/oidc/login")));
    }

    @Test
    void token_fromFile_throwsOnMissingFile() {
        final Path missing = tempDir.resolve("nonexistent");

        final var source = JwtVaultTokenSource.fromFile(
                wireMock.baseUrl(), "my-role", missing, "jwt",
                http, mapper, Clock.systemUTC());

        assertThatThrownBy(source::token)
                .isInstanceOf(UncheckedIOException.class)
                .hasMessageContaining("nonexistent");
    }

    @Test
    void token_supplierCalledOnEachLogin() {
        final AtomicInteger callCount = new AtomicInteger();
        final Supplier<String> counting = () -> {
            callCount.incrementAndGet();
            return "jwt-v" + callCount.get();
        };

        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.first", 3600))));

        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", counting,
                "jwt", http, mapper, Clock.systemUTC());

        source.token();
        assertThat(callCount.get()).isEqualTo(1);

        wireMock.resetAll();
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.second", 3600))));

        source.invalidate();
        source.token();
        assertThat(callCount.get()).isEqualTo(2);
    }

    @Test
    void token_supplierReturningNull_throwsNPE() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.token", 3600))));

        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", () -> null,
                "jwt", http, mapper, Clock.systemUTC());

        assertThatThrownBy(source::token)
                .isInstanceOf(NullPointerException.class)
                .hasMessageContaining("JWT supplier returned null");
    }

    @Test
    void token_fromFile_kubernetesMountPath() throws IOException {
        final Path jwtFile = tempDir.resolve("sa-token");
        Files.writeString(jwtFile, "eyJhbGciOiJSUzI1NiJ9.k8s-jwt");

        wireMock.stubFor(post(urlEqualTo("/v1/auth/kubernetes/login"))
                .willReturn(okJson(loginResponse("hvs.k8s", 3600))));

        final var source = JwtVaultTokenSource.fromFile(
                wireMock.baseUrl(), "my-role", jwtFile, "kubernetes",
                http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.k8s");
        wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/kubernetes/login"))
                .withRequestBody(containing("eyJhbGciOiJSUzI1NiJ9.k8s-jwt")));
    }

    @Test
    void token_fromFile_reReadsOnReLogin() throws IOException {
        final Path jwtFile = tempDir.resolve("rotating-token");
        Files.writeString(jwtFile, "jwt-v1");

        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.first", 3600))));

        final var source = JwtVaultTokenSource.fromFile(
                wireMock.baseUrl(), "my-role", jwtFile, "jwt",
                http, mapper, Clock.systemUTC());

        source.token();

        Files.writeString(jwtFile, "jwt-v2");
        wireMock.resetAll();
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.second", 3600))));

        source.invalidate();
        source.token();

        wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/jwt/login"))
                .withRequestBody(containing("jwt-v2")));
    }

    @Test
    void token_defaultsMountPathToJwt() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
                .willReturn(okJson(loginResponse("hvs.default", 3600))));

        final var source = new JwtVaultTokenSource(
                wireMock.baseUrl(), "my-role", () -> "jwt-value",
                null, http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.default");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/jwt/login")));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest=JwtVaultTokenSourceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — `JwtVaultTokenSource` does not exist.

- [ ] **Step 3: Implement JwtVaultTokenSource**

Create `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/JwtVaultTokenSource.java`:

```java
package io.casehub.ledger.signing.vault;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.net.http.HttpClient;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Clock;
import java.util.Objects;
import java.util.function.Supplier;

import com.fasterxml.jackson.databind.ObjectMapper;

public final class JwtVaultTokenSource extends LoginBasedVaultTokenSource {

    private final String role;
    private final Supplier<String> jwtSupplier;
    private final String mountPath;

    public JwtVaultTokenSource(final String vaultAddress, final String role,
            final Supplier<String> jwtSupplier, final String mountPath,
            final HttpClient http, final ObjectMapper mapper, final Clock clock) {
        super(vaultAddress, http, mapper, clock);
        this.role = Objects.requireNonNull(role);
        this.jwtSupplier = Objects.requireNonNull(jwtSupplier);
        this.mountPath = mountPath != null ? mountPath : "jwt";
    }

    public static JwtVaultTokenSource fromFile(final String vaultAddress, final String role,
            final Path jwtPath, final String mountPath,
            final HttpClient http, final ObjectMapper mapper, final Clock clock) {
        Objects.requireNonNull(jwtPath, "jwtPath must not be null");
        return new JwtVaultTokenSource(vaultAddress, role, () -> {
            try {
                return Files.readString(jwtPath).trim();
            } catch (final IOException e) {
                throw new UncheckedIOException("Failed to read JWT from " + jwtPath, e);
            }
        }, mountPath, http, mapper, clock);
    }

    @Override
    protected String loginPath() {
        return "/v1/auth/" + mountPath + "/login";
    }

    @Override
    protected String loginRequestBody() {
        final String jwt = Objects.requireNonNull(jwtSupplier.get(), "JWT supplier returned null");
        return "{\"role\":\"" + role + "\",\"jwt\":\"" + jwt + "\"}";
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest=JwtVaultTokenSourceTest`
Expected: All 12 tests PASS.

- [ ] **Step 5: Commit**

```bash
git add signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/JwtVaultTokenSource.java \
       signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/JwtVaultTokenSourceTest.java
git commit -m "feat(vault): add JwtVaultTokenSource with Supplier-based JWT acquisition

Refs #170"
```

---

### Task 2: Delete KubernetesVaultTokenSource and migrate its tests

**Files:**
- Delete: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/KubernetesVaultTokenSource.java`
- Delete: `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/KubernetesVaultTokenSourceTest.java`

**Interfaces:**
- Consumes: `JwtVaultTokenSource.fromFile(...)` from Task 1
- Produces: Nothing new — this is a deletion. All K8s test cases already covered by Task 1 tests (`token_fromFile_kubernetesMountPath`, `token_fromFile_reReadsOnReLogin`).

**Pre-check:** Task 1 tests `token_fromFile_kubernetesMountPath` and `token_fromFile_reReadsOnReLogin` already cover the K8s-specific scenarios from `KubernetesVaultTokenSourceTest`. Verify by cross-referencing:

| KubernetesVaultTokenSourceTest | JwtVaultTokenSourceTest equivalent |
|---|---|
| `token_readsJwtFromFileAndLogsIn` | `token_fromFile_readsJwtFromFile` |
| `token_reReadsJwtOnReLogin` | `token_fromFile_reReadsOnReLogin` |
| `token_throwsOnMissingJwtFile` | `token_fromFile_throwsOnMissingFile` |
| K8s mount path | `token_fromFile_kubernetesMountPath` |

- [ ] **Step 1: Verify Task 1 K8s-equivalent tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest="JwtVaultTokenSourceTest#token_fromFile_kubernetesMountPath+token_fromFile_reReadsOnReLogin+token_fromFile_readsJwtFromFile+token_fromFile_throwsOnMissingFile"`
Expected: All 4 tests PASS.

- [ ] **Step 2: Check for references to KubernetesVaultTokenSource**

Use IntelliJ MCP `ide_search_text` to find all references to `KubernetesVaultTokenSource` across the project. The only references should be in the Quarkus adapter (`VaultTransitAgentSigner.java`) and the import in the same file. These will be updated in Task 3.

- [ ] **Step 3: Delete the files**

Delete `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/KubernetesVaultTokenSource.java` and `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/KubernetesVaultTokenSourceTest.java`.

- [ ] **Step 4: Verify remaining vault-transit tests still pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing`
Expected: All tests PASS (AppRoleVaultTokenSourceTest, StaticVaultTokenSourceTest, JwtVaultTokenSourceTest, VaultTransitSigningClientTest, VaultTransitAuthIntegrationTest).

- [ ] **Step 5: Commit**

```bash
git add -u signing/vault-transit/src/
git commit -m "refactor(vault): delete KubernetesVaultTokenSource — consolidated into JwtVaultTokenSource

KubernetesVaultTokenSource and JwtVaultTokenSource had identical loginPath()
and loginRequestBody() semantics. KUBERNETES auth is now served by
JwtVaultTokenSource.fromFile() with mountPath=\"kubernetes\".

Test coverage migrated to JwtVaultTokenSourceTest.

Refs #170"
```

---

### Task 3: Quarkus config and adapter wiring

**Files:**
- Modify: `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitConfig.java`
- Modify: `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSigner.java`

**Interfaces:**
- Consumes: `JwtVaultTokenSource` and `JwtVaultTokenSource.fromFile()` from Task 1
- Produces: `AuthMethod.JWT` enum value, `AuthConfig.jwt()` config property, updated `createTokenSource()` switch

- [ ] **Step 1: Update VaultTransitConfig**

Edit `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitConfig.java`:

1. Add `JWT` to the enum:
```java
enum AuthMethod { TOKEN, APPROLE, KUBERNETES, JWT }
```

2. Change `jwtPath()` from `@WithDefault(...) String` to `Optional<String>`:
```java
/**
 * Path to JWT file for KUBERNETES or JWT auth methods.
 */
Optional<String> jwtPath();
```

3. Add `jwt()`:
```java
/**
 * Static JWT string for JWT auth method. Mutually exclusive with jwt-path.
 * Quarkus config substitution applies: {@code ${MY_JWT_TOKEN}} resolves from environment.
 */
Optional<String> jwt();
```

4. Update `role()` Javadoc:
```java
/**
 * Role for Kubernetes or JWT auth methods.
 */
Optional<String> role();
```

5. Update `mountPath()` Javadoc:
```java
/**
 * Vault mount path for AppRole, Kubernetes, or JWT auth.
 * Defaults to the auth method name if not specified.
 * For Vault OIDC auth backends, use {@code mountPath=oidc} with {@code method=jwt}
 * — the JWT and OIDC auth methods share the same login API.
 */
Optional<String> mountPath();
```

- [ ] **Step 2: Update VaultTransitAgentSigner**

Edit `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSigner.java`:

1. Replace the `KubernetesVaultTokenSource` import with `JwtVaultTokenSource`:
```java
// Remove:
import io.casehub.ledger.signing.vault.KubernetesVaultTokenSource;
// Add:
import io.casehub.ledger.signing.vault.JwtVaultTokenSource;
```

2. Update the `KUBERNETES` case in `createTokenSource()`:
```java
case KUBERNETES -> {
    final String role = authConfig.role()
            .orElseThrow(() -> new IllegalStateException(
                    "casehub.ledger.vault-transit.auth.role required when auth.method=kubernetes"));
    final java.nio.file.Path jwtPath = java.nio.file.Path.of(
            authConfig.jwtPath().orElse("/var/run/secrets/kubernetes.io/serviceaccount/token"));
    final String mountPath = authConfig.mountPath().orElse("kubernetes");
    yield JwtVaultTokenSource.fromFile(config.address(), role, jwtPath, mountPath,
            httpClient, objectMapper, java.time.Clock.systemUTC());
}
```

3. Add the `JWT` case:
```java
case JWT -> {
    final String role = authConfig.role()
            .orElseThrow(() -> new IllegalStateException(
                    "casehub.ledger.vault-transit.auth.role required when auth.method=jwt"));
    final String mountPath = authConfig.mountPath().orElse("jwt");
    if (authConfig.jwtPath().isPresent() && authConfig.jwt().isPresent()) {
        throw new IllegalStateException(
                "casehub.ledger.vault-transit.auth: specify jwt-path or jwt, not both");
    }
    if (authConfig.jwtPath().isPresent()) {
        yield JwtVaultTokenSource.fromFile(config.address(), role,
                java.nio.file.Path.of(authConfig.jwtPath().get()), mountPath,
                httpClient, objectMapper, java.time.Clock.systemUTC());
    }
    final String jwt = authConfig.jwt()
            .orElseThrow(() -> new IllegalStateException(
                    "casehub.ledger.vault-transit.auth.jwt or auth.jwt-path required when auth.method=jwt"));
    yield new JwtVaultTokenSource(config.address(), role, () -> jwt, mountPath,
            httpClient, objectMapper, java.time.Clock.systemUTC());
}
```

- [ ] **Step 3: Verify existing IT tests pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit-quarkus -Pwith-signing`
Expected: All existing `VaultTransitAgentSignerIT` tests PASS (they use `auth.method=token` so the changes are invisible).

- [ ] **Step 4: Commit**

```bash
git add signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitConfig.java \
       signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSigner.java
git commit -m "feat(vault): add AuthMethod.JWT and wire createTokenSource for JWT + KUBERNETES consolidation

KUBERNETES case now uses JwtVaultTokenSource.fromFile() instead of the
deleted KubernetesVaultTokenSource. JWT case supports jwt-path (file)
and jwt (static string) with mutual exclusion validation.

jwtPath config changed from @WithDefault String to Optional<String> —
K8s default path applied in adapter code, auth-method-specific.

Refs #170"
```

---

### Task 4: Quarkus IT tests for JWT auth and KUBERNETES regression

**Files:**
- Modify: `signing/vault-transit-quarkus/src/test/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSignerIT.java`

**Interfaces:**
- Consumes: `VaultTransitAgentSigner` with JWT auth wiring from Task 3
- Produces: IT test coverage for JWT auth pipeline and KUBERNETES config regression

The existing IT tests use `auth.method=token` (from `application.properties`). To test JWT auth, we need tests that construct the signer with JWT config. The test-visible constructor `VaultTransitAgentSigner(config, client, tokenSource)` bypasses `createTokenSource()`, so we need a separate test that exercises the static factory method directly.

- [ ] **Step 1: Add createTokenSource unit tests to VaultTransitAgentSignerIT**

Add these tests to `VaultTransitAgentSignerIT.java`. These test the `createTokenSource()` switch directly, not through CDI:

```java
@Test
void jwtAuth_fileBasedJwt_signsSuccessfully(@TempDir final Path tempDir) throws Exception {
    final Path jwtFile = tempDir.resolve("vault-jwt");
    Files.writeString(jwtFile, "eyJhbGciOiJSUzI1NiJ9.file-jwt");

    // Stub JWT auth login
    wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
            .willReturn(okJson("{\"auth\":{\"client_token\":\"hvs.jwt-file\","
                    + "\"lease_duration\":3600,\"renewable\":true}}")));

    final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    final byte[] data = "jwt file auth test".getBytes();
    stubKeyInfo(kp);
    stubSign(kp, data);

    // Build config stub for JWT file auth
    final VaultTransitConfig config = jwtConfig("jwt",
            Optional.of(jwtFile.toString()), Optional.empty(), Optional.empty());

    final var signer = new VaultTransitAgentSigner(config);
    final Optional<AgentSignature> result = signer.sign("claude:reviewer@v1", data);

    assertThat(result).isPresent();
    assertThat(result.get().publicKey()).isEqualTo(kp.getPublic().getEncoded());

    wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/jwt/login"))
            .withRequestBody(containing("eyJhbGciOiJSUzI1NiJ9.file-jwt")));
}

@Test
void jwtAuth_staticJwt_signsSuccessfully() throws Exception {
    // Stub JWT auth login
    wireMock.stubFor(post(urlEqualTo("/v1/auth/jwt/login"))
            .willReturn(okJson("{\"auth\":{\"client_token\":\"hvs.jwt-static\","
                    + "\"lease_duration\":3600,\"renewable\":true}}")));

    final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    final byte[] data = "jwt static auth test".getBytes();
    stubKeyInfo(kp);
    stubSign(kp, data);

    final VaultTransitConfig config = jwtConfig("jwt",
            Optional.empty(), Optional.of("eyJhbGciOiJSUzI1NiJ9.static-jwt"), Optional.empty());

    final var signer = new VaultTransitAgentSigner(config);
    final Optional<AgentSignature> result = signer.sign("claude:reviewer@v1", data);

    assertThat(result).isPresent();
    wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/jwt/login"))
            .withRequestBody(containing("eyJhbGciOiJSUzI1NiJ9.static-jwt")));
}

@Test
void jwtAuth_missingRole_failsFast() {
    final VaultTransitConfig config = jwtConfig("jwt",
            Optional.of("/tmp/jwt"), Optional.empty(), Optional.empty(),
            Optional.empty()); // no role

    assertThatThrownBy(() -> new VaultTransitAgentSigner(config))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("auth.role required");
}

@Test
void jwtAuth_bothJwtAndJwtPath_failsFast() {
    final VaultTransitConfig config = jwtConfig("jwt",
            Optional.of("/tmp/jwt"), Optional.of("inline-jwt"), Optional.empty());

    assertThatThrownBy(() -> new VaultTransitAgentSigner(config))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("specify jwt-path or jwt, not both");
}

@Test
void jwtAuth_neitherJwtNorJwtPath_failsFast() {
    final VaultTransitConfig config = jwtConfig("jwt",
            Optional.empty(), Optional.empty(), Optional.empty());

    assertThatThrownBy(() -> new VaultTransitAgentSigner(config))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("auth.jwt or auth.jwt-path required");
}

@Test
void kubernetesAuth_defaultJwtPath_configPath(@TempDir final Path tempDir) throws Exception {
    // This test verifies the KUBERNETES case in createTokenSource() applies
    // the default K8s service account path when jwtPath is absent.
    // Since the default path won't exist in test, we provide an explicit path.
    final Path jwtFile = tempDir.resolve("sa-token");
    Files.writeString(jwtFile, "eyJhbGciOiJSUzI1NiJ9.k8s-sa");

    wireMock.stubFor(post(urlEqualTo("/v1/auth/kubernetes/login"))
            .willReturn(okJson("{\"auth\":{\"client_token\":\"hvs.k8s\","
                    + "\"lease_duration\":3600,\"renewable\":true}}")));

    final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
    final byte[] data = "k8s auth test".getBytes();
    stubKeyInfo(kp);
    stubSign(kp, data);

    final VaultTransitConfig config = kubernetesConfig(
            Optional.of("my-role"), Optional.of(jwtFile.toString()), Optional.empty());

    final var signer = new VaultTransitAgentSigner(config);
    final Optional<AgentSignature> result = signer.sign("claude:reviewer@v1", data);

    assertThat(result).isPresent();
    wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/kubernetes/login"))
            .withRequestBody(containing("eyJhbGciOiJSUzI1NiJ9.k8s-sa")));
}
```

- [ ] **Step 2: Add config stub helper methods**

Add to `VaultTransitAgentSignerIT.java` — these create `VaultTransitConfig` stubs for non-CDI construction:

```java
private static VaultTransitConfig jwtConfig(final String mountPath,
        final Optional<String> jwtPath, final Optional<String> jwt,
        final Optional<String> mountPathOpt) {
    return jwtConfig(mountPath, jwtPath, jwt, mountPathOpt, Optional.of("my-role"));
}

private static VaultTransitConfig jwtConfig(final String mountPath,
        final Optional<String> jwtPath, final Optional<String> jwt,
        final Optional<String> mountPathOpt, final Optional<String> role) {
    return new VaultTransitConfig() {
        @Override public String address() { return "http://localhost:8098"; }
        @Override public java.util.Map<String, String> keyMapping() {
            return java.util.Map.of("claude:reviewer@v1", "reviewer-key");
        }
        @Override public String refreshInterval() { return "24h"; }
        @Override public AuthConfig auth() {
            return new AuthConfig() {
                @Override public AuthMethod method() { return AuthMethod.JWT; }
                @Override public Optional<String> token() { return Optional.empty(); }
                @Override public Optional<String> roleId() { return Optional.empty(); }
                @Override public Optional<String> secretId() { return Optional.empty(); }
                @Override public Optional<String> role() { return role; }
                @Override public Optional<String> jwtPath() { return jwtPath; }
                @Override public Optional<String> jwt() { return jwt; }
                @Override public Optional<String> mountPath() { return mountPathOpt; }
            };
        }
    };
}

private static VaultTransitConfig kubernetesConfig(final Optional<String> role,
        final Optional<String> jwtPath, final Optional<String> mountPath) {
    return new VaultTransitConfig() {
        @Override public String address() { return "http://localhost:8098"; }
        @Override public java.util.Map<String, String> keyMapping() {
            return java.util.Map.of("claude:reviewer@v1", "reviewer-key");
        }
        @Override public String refreshInterval() { return "24h"; }
        @Override public AuthConfig auth() {
            return new AuthConfig() {
                @Override public AuthMethod method() { return AuthMethod.KUBERNETES; }
                @Override public Optional<String> token() { return Optional.empty(); }
                @Override public Optional<String> roleId() { return Optional.empty(); }
                @Override public Optional<String> secretId() { return Optional.empty(); }
                @Override public Optional<String> role() { return role; }
                @Override public Optional<String> jwtPath() { return jwtPath; }
                @Override public Optional<String> jwt() { return Optional.empty(); }
                @Override public Optional<String> mountPath() { return mountPath; }
            };
        }
    };
}
```

- [ ] **Step 3: Add required imports**

Add to the import block:
```java
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import org.junit.jupiter.api.io.TempDir;
```

- [ ] **Step 4: Run all IT tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit-quarkus -Pwith-signing`
Expected: All tests PASS (existing 7 + new 6 = 13 tests).

- [ ] **Step 5: Commit**

```bash
git add signing/vault-transit-quarkus/src/test/
git commit -m "test(vault): add IT tests for JWT auth pipeline and KUBERNETES regression

Covers: file-based JWT, static JWT, mutual exclusion validation,
missing role, neither-jwt-nor-path, and KUBERNETES case using
JwtVaultTokenSource.fromFile() after consolidation.

Refs #170"
```

---

### Task 5: Housekeeping — ARC42STORIES tech debt, GitHub issue, CLAUDE.md

**Files:**
- Modify: `ARC42STORIES.MD` (line 1306 — remove stale tech debt entry)
- Modify: `CLAUDE.md` (update signing module documentation)

**Interfaces:**
- Consumes: Nothing from prior tasks
- Produces: Clean tech debt table, GitHub issue for browser-based OIDC

- [ ] **Step 1: Update ARC42STORIES.MD §12 tech debt**

At line 1306, the entry reads:
```
| Vault AppRole/OIDC auth for `VaultTransitAgentSigner` not implemented | L5 | #101 |
```

Issue #101 is CLOSED (delivered AppRole + Kubernetes auth). This branch (#170) delivers JWT auth. Remove the line entirely — the gap is closed.

- [ ] **Step 2: File GitHub issue for browser-based OIDC**

```bash
gh issue create --repo casehubio/ledger \
  --title "feat: Vault browser-based OIDC auth flow (two-step auth URL + callback)" \
  --body "$(cat <<'EOF'
Deferred from #170 (JWT auth method).

Vault's JWT/OIDC plugin supports a two-step browser-based OIDC flow:
1. `GET /v1/auth/oidc/auth_url` → redirect URL
2. `GET /v1/auth/oidc/callback?state=...&nonce=...&code=...` → Vault token

This is for interactive (human) auth, not machine-to-machine. Different use case from #170's JWT-based flow.

Not needed until casehub has interactive admin tooling that authenticates to Vault.
EOF
)"
```

- [ ] **Step 3: Update CLAUDE.md signing documentation**

Update the `VaultTransitConfig.AuthConfig` section to reflect `AuthMethod.JWT` and the `KubernetesVaultTokenSource` consolidation. Update the project structure tree to show `JwtVaultTokenSource` instead of `KubernetesVaultTokenSource`.

- [ ] **Step 4: Run full signing module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-signing`
Expected: All signing module tests PASS.

- [ ] **Step 5: Commit**

```bash
git add ARC42STORIES.MD CLAUDE.md
git commit -m "docs: remove stale Vault auth tech debt entry, update signing docs

§12 tech debt entry for Vault AppRole/OIDC auth (#101) removed — #101
delivered AppRole + Kubernetes, #170 delivers JWT auth.

Browser-based OIDC flow filed as separate issue.

CLAUDE.md updated: JwtVaultTokenSource replaces KubernetesVaultTokenSource,
AuthMethod.JWT documented.

Refs #170"
```
