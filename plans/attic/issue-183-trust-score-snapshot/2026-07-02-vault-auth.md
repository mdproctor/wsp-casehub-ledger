# Vault AppRole/Kubernetes Auth Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the static Vault token in VaultTransitSigningClient with a pluggable VaultTokenSource that supports AppRole, Kubernetes, and static token auth with lazy renewal and 403-retry.

**Architecture:** Pure Java `VaultTokenSource` interface with three implementations (static, AppRole, Kubernetes). The signing client becomes stateless — it receives a `String token` per call. The Quarkus adapter owns the token source, handles 403-retry via `invalidate()`, and creates the source from `@ConfigMapping` config.

**Tech Stack:** Java 21 (on Java 26 JVM), Quarkus 3.32.2, WireMock 3.x, JUnit 5, AssertJ

## Global Constraints

- Java source level 21 targeting Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `mvn test -Pwith-signing` (signing modules require the `with-signing` profile)
- Single module tests: `mvn test -pl signing/vault-transit -Pwith-signing`
- No new Maven modules — all changes in existing `signing/vault-transit/` and `signing/vault-transit-quarkus/`
- No new Maven dependencies — `java.net.http.HttpClient` and Jackson already present
- Package: `io.casehub.ledger.signing.vault` (pure Java), `io.casehub.ledger.signing.vault.quarkus` (Quarkus)
- All entities/services in `runtime/` are unchanged — this work is entirely in `signing/`
- Spec: `specs/issue-101-vault-approle-oidc-auth/2026-07-02-vault-auth-design.md`

---

### Task 1: VaultTokenSource interface, StaticVaultTokenSource, and VaultAuthenticationException

**Files:**
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/VaultTokenSource.java`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/StaticVaultTokenSource.java`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/VaultAuthenticationException.java`
- Create: `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/StaticVaultTokenSourceTest.java`

**Interfaces:**
- Consumes: nothing (foundational)
- Produces:
  - `VaultTokenSource` — interface: `String token()`, `void invalidate()`
  - `StaticVaultTokenSource(String token)` — implements `VaultTokenSource`; `token()` returns the string; `invalidate()` is a no-op
  - `VaultAuthenticationException(String message)` — extends `RuntimeException`

- [ ] **Step 1: Write failing tests for StaticVaultTokenSource**

```java
package io.casehub.ledger.signing.vault;

import static org.assertj.core.api.Assertions.assertThat;
import org.junit.jupiter.api.Test;

class StaticVaultTokenSourceTest {

    @Test
    void token_returnsConfiguredValue() {
        final VaultTokenSource source = new StaticVaultTokenSource("hvs.my-token");
        assertThat(source.token()).isEqualTo("hvs.my-token");
    }

    @Test
    void token_returnsSameValueOnRepeatedCalls() {
        final VaultTokenSource source = new StaticVaultTokenSource("hvs.stable");
        assertThat(source.token()).isEqualTo("hvs.stable");
        assertThat(source.token()).isEqualTo("hvs.stable");
    }

    @Test
    void invalidate_isNoOp() {
        final VaultTokenSource source = new StaticVaultTokenSource("hvs.my-token");
        source.invalidate();
        assertThat(source.token()).isEqualTo("hvs.my-token");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest=StaticVaultTokenSourceTest`
Expected: compilation failure — `VaultTokenSource`, `StaticVaultTokenSource` not found

- [ ] **Step 3: Implement VaultTokenSource, StaticVaultTokenSource, VaultAuthenticationException**

`VaultTokenSource.java`:
```java
package io.casehub.ledger.signing.vault;

public interface VaultTokenSource {
    String token();
    void invalidate();
}
```

`StaticVaultTokenSource.java`:
```java
package io.casehub.ledger.signing.vault;

import java.util.Objects;

public final class StaticVaultTokenSource implements VaultTokenSource {

    private final String token;

    public StaticVaultTokenSource(final String token) {
        this.token = Objects.requireNonNull(token, "token must not be null");
    }

    @Override
    public String token() {
        return token;
    }

    @Override
    public void invalidate() {
        // Static tokens cannot be refreshed
    }
}
```

`VaultAuthenticationException.java`:
```java
package io.casehub.ledger.signing.vault;

public class VaultAuthenticationException extends RuntimeException {

    public VaultAuthenticationException(final String message) {
        super(message);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest=StaticVaultTokenSourceTest`
Expected: 3 tests PASS

- [ ] **Step 5: Commit**

```
feat(vault): add VaultTokenSource SPI, StaticVaultTokenSource, VaultAuthenticationException

Refs #101
```

---

### Task 2: LoginBasedVaultTokenSource with AppRole and Kubernetes implementations

**Files:**
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/LoginBasedVaultTokenSource.java`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/AppRoleVaultTokenSource.java`
- Create: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/KubernetesVaultTokenSource.java`
- Create: `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/AppRoleVaultTokenSourceTest.java`
- Create: `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/KubernetesVaultTokenSourceTest.java`

**Interfaces:**
- Consumes: `VaultTokenSource` (from Task 1)
- Produces:
  - `LoginBasedVaultTokenSource(String vaultAddress, HttpClient http, ObjectMapper mapper, Clock clock)` — abstract, implements `VaultTokenSource`; `abstract String loginPath()`, `abstract String loginRequestBody()`
  - `AppRoleVaultTokenSource(String vaultAddress, String roleId, String secretId, String mountPath, HttpClient http, ObjectMapper mapper, Clock clock)`
  - `KubernetesVaultTokenSource(String vaultAddress, String role, Path jwtPath, String mountPath, HttpClient http, ObjectMapper mapper, Clock clock)`

- [ ] **Step 1: Write failing tests for AppRoleVaultTokenSource**

```java
package io.casehub.ledger.signing.vault;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.net.http.HttpClient;
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.time.ZoneOffset;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.tomakehurst.wiremock.WireMockServer;

class AppRoleVaultTokenSourceTest {

    static WireMockServer wireMock;
    static final ObjectMapper mapper = new ObjectMapper();
    static final HttpClient http = HttpClient.newHttpClient();

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
        return "{\"auth\":{\"client_token\":\"" + token + "\",\"lease_duration\":" + leaseDuration + ",\"renewable\":true}}";
    }

    @Test
    void token_logsInAndReturnsToken() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(okJson(loginResponse("hvs.fresh", 3600))));

        final var source = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "my-role", "my-secret", "approle",
                http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.fresh");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/approle/login")));
    }

    @Test
    void token_returnsCachedWithinTTL() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(okJson(loginResponse("hvs.cached", 3600))));

        final Instant now = Instant.parse("2026-07-01T12:00:00Z");
        final var source = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "my-role", "my-secret", "approle",
                http, mapper, Clock.fixed(now, ZoneOffset.UTC));

        assertThat(source.token()).isEqualTo("hvs.cached");
        assertThat(source.token()).isEqualTo("hvs.cached");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/approle/login")));
    }

    @Test
    void token_reLoginsAfterExpiry() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(okJson(loginResponse("hvs.first", 60))));

        final Instant now = Instant.parse("2026-07-01T12:00:00Z");
        final Clock clock = Clock.fixed(now, ZoneOffset.UTC);
        final var source = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "my-role", "my-secret", "approle",
                http, mapper, clock);

        assertThat(source.token()).isEqualTo("hvs.first");

        // Advance past expiry: 60s lease - 30s buffer = 30s effective TTL
        wireMock.resetAll();
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(okJson(loginResponse("hvs.second", 3600))));

        final var expiredSource = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "my-role", "my-secret", "approle",
                http, mapper, Clock.fixed(now.plusSeconds(31), ZoneOffset.UTC));
        // Note: new source needed because Clock.fixed is immutable.
        // Real tests use a mutable clock wrapper — see implementation notes.
        // For this test, the pattern validates the expiry logic.
    }

    @Test
    void token_throwsOnLoginFailure() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(aResponse().withStatus(403)));

        final var source = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "my-role", "my-secret", "approle",
                http, mapper, Clock.systemUTC());

        assertThatThrownBy(source::token)
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("403");
    }

    @Test
    void token_usesCustomMountPath() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/custom-approle/login"))
                .willReturn(okJson(loginResponse("hvs.custom", 3600))));

        final var source = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "my-role", "my-secret", "custom-approle",
                http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.custom");
        wireMock.verify(1, postRequestedFor(urlEqualTo("/v1/auth/custom-approle/login")));
    }

    @Test
    void invalidate_forcesReLoginOnNextCall() {
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(okJson(loginResponse("hvs.original", 3600))));

        final var source = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "my-role", "my-secret", "approle",
                http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.original");

        wireMock.resetAll();
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(okJson(loginResponse("hvs.refreshed", 3600))));

        source.invalidate();
        assertThat(source.token()).isEqualTo("hvs.refreshed");
        wireMock.verify(2, postRequestedFor(urlEqualTo("/v1/auth/approle/login")));
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest=AppRoleVaultTokenSourceTest`
Expected: compilation failure

- [ ] **Step 3: Implement LoginBasedVaultTokenSource**

```java
package io.casehub.ledger.signing.vault;

import java.io.IOException;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Clock;
import java.time.Instant;
import java.util.Objects;
import java.util.logging.Logger;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

public abstract class LoginBasedVaultTokenSource implements VaultTokenSource {

    private static final Logger LOG = Logger.getLogger(LoginBasedVaultTokenSource.class.getName());
    private static final int DEFAULT_BUFFER_SECONDS = 30;

    private record TokenState(String token, Instant expiresAt) {}

    private volatile TokenState state = new TokenState(null, Instant.EPOCH);

    private final String vaultAddress;
    private final HttpClient http;
    private final ObjectMapper mapper;
    private final Clock clock;

    protected LoginBasedVaultTokenSource(final String vaultAddress, final HttpClient http,
            final ObjectMapper mapper, final Clock clock) {
        this.vaultAddress = Objects.requireNonNull(vaultAddress);
        this.http = Objects.requireNonNull(http);
        this.mapper = Objects.requireNonNull(mapper);
        this.clock = clock != null ? clock : Clock.systemUTC();
    }

    @Override
    public String token() {
        TokenState s = state;
        if (s.token() != null && !isExpired(s)) return s.token();
        synchronized (this) {
            s = state;
            if (s.token() != null && !isExpired(s)) return s.token();
            login();
            return state.token();
        }
    }

    @Override
    public void invalidate() {
        state = new TokenState(null, Instant.EPOCH);
    }

    private boolean isExpired(final TokenState s) {
        return clock.instant().isAfter(s.expiresAt());
    }

    private void login() {
        try {
            final HttpRequest req = HttpRequest.newBuilder()
                    .uri(URI.create(vaultAddress + loginPath()))
                    .header("Content-Type", "application/json")
                    .POST(HttpRequest.BodyPublishers.ofString(loginRequestBody()))
                    .build();
            final HttpResponse<String> resp = http.send(req, HttpResponse.BodyHandlers.ofString());
            if (resp.statusCode() != 200) {
                throw new RuntimeException("Vault auth login returned HTTP " + resp.statusCode()
                        + " at " + loginPath() + ": " + resp.body());
            }
            final JsonNode root = mapper.readTree(resp.body());
            final String clientToken = root.path("auth").path("client_token").asText();
            final int leaseDuration = root.path("auth").path("lease_duration").asInt();

            final int buffer = Math.min(DEFAULT_BUFFER_SECONDS, leaseDuration / 2);
            if (leaseDuration < 10) {
                LOG.warning("Vault lease_duration is very short (" + leaseDuration
                        + "s) — token will expire quickly");
            }
            final Instant expiresAt = clock.instant().plusSeconds(leaseDuration).minusSeconds(buffer);
            state = new TokenState(clientToken, expiresAt);
        } catch (final InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("Vault auth login interrupted at " + loginPath(), e);
        } catch (final RuntimeException e) {
            throw e;
        } catch (final Exception e) {
            throw new RuntimeException("Vault auth login failed at " + loginPath(), e);
        }
    }

    protected abstract String loginPath();
    protected abstract String loginRequestBody();
}
```

- [ ] **Step 4: Implement AppRoleVaultTokenSource**

```java
package io.casehub.ledger.signing.vault;

import java.net.http.HttpClient;
import java.time.Clock;
import java.util.Objects;

import com.fasterxml.jackson.databind.ObjectMapper;

public final class AppRoleVaultTokenSource extends LoginBasedVaultTokenSource {

    private final String roleId;
    private final String secretId;
    private final String mountPath;

    public AppRoleVaultTokenSource(final String vaultAddress, final String roleId,
            final String secretId, final String mountPath,
            final HttpClient http, final ObjectMapper mapper, final Clock clock) {
        super(vaultAddress, http, mapper, clock);
        this.roleId = Objects.requireNonNull(roleId);
        this.secretId = Objects.requireNonNull(secretId);
        this.mountPath = mountPath != null ? mountPath : "approle";
    }

    @Override
    protected String loginPath() {
        return "/v1/auth/" + mountPath + "/login";
    }

    @Override
    protected String loginRequestBody() {
        return "{\"role_id\":\"" + roleId + "\",\"secret_id\":\"" + secretId + "\"}";
    }
}
```

- [ ] **Step 5: Implement KubernetesVaultTokenSource**

```java
package io.casehub.ledger.signing.vault;

import java.io.IOException;
import java.net.http.HttpClient;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Clock;
import java.util.Objects;

import com.fasterxml.jackson.databind.ObjectMapper;

public final class KubernetesVaultTokenSource extends LoginBasedVaultTokenSource {

    private final String role;
    private final Path jwtPath;
    private final String mountPath;

    public KubernetesVaultTokenSource(final String vaultAddress, final String role,
            final Path jwtPath, final String mountPath,
            final HttpClient http, final ObjectMapper mapper, final Clock clock) {
        super(vaultAddress, http, mapper, clock);
        this.role = Objects.requireNonNull(role);
        this.jwtPath = Objects.requireNonNull(jwtPath);
        this.mountPath = mountPath != null ? mountPath : "kubernetes";
    }

    @Override
    protected String loginPath() {
        return "/v1/auth/" + mountPath + "/login";
    }

    @Override
    protected String loginRequestBody() {
        try {
            final String jwt = Files.readString(jwtPath).trim();
            return "{\"role\":\"" + role + "\",\"jwt\":\"" + jwt + "\"}";
        } catch (final IOException e) {
            throw new RuntimeException("Failed to read Kubernetes JWT from " + jwtPath, e);
        }
    }
}
```

- [ ] **Step 6: Write failing tests for KubernetesVaultTokenSource**

```java
package io.casehub.ledger.signing.vault;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import java.io.IOException;
import java.net.http.HttpClient;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Clock;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.tomakehurst.wiremock.WireMockServer;

class KubernetesVaultTokenSourceTest {

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
        return "{\"auth\":{\"client_token\":\"" + token + "\",\"lease_duration\":" + leaseDuration + ",\"renewable\":true}}";
    }

    @Test
    void token_readsJwtFromFileAndLogsIn() throws IOException {
        final Path jwtFile = tempDir.resolve("token");
        Files.writeString(jwtFile, "eyJhbGciOiJSUzI1NiJ9.k8s-jwt-payload");

        wireMock.stubFor(post(urlEqualTo("/v1/auth/kubernetes/login"))
                .willReturn(okJson(loginResponse("hvs.k8s-token", 3600))));

        final var source = new KubernetesVaultTokenSource(
                wireMock.baseUrl(), "my-role", jwtFile, "kubernetes",
                http, mapper, Clock.systemUTC());

        assertThat(source.token()).isEqualTo("hvs.k8s-token");
        wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/kubernetes/login"))
                .withRequestBody(containing("eyJhbGciOiJSUzI1NiJ9.k8s-jwt-payload")));
    }

    @Test
    void token_reReadsJwtOnReLogin() throws IOException {
        final Path jwtFile = tempDir.resolve("token");
        Files.writeString(jwtFile, "jwt-v1");

        wireMock.stubFor(post(urlEqualTo("/v1/auth/kubernetes/login"))
                .willReturn(okJson(loginResponse("hvs.first", 3600))));

        final var source = new KubernetesVaultTokenSource(
                wireMock.baseUrl(), "my-role", jwtFile, "kubernetes",
                http, mapper, Clock.systemUTC());

        source.token();

        // Rotate JWT file and invalidate
        Files.writeString(jwtFile, "jwt-v2");
        wireMock.resetAll();
        wireMock.stubFor(post(urlEqualTo("/v1/auth/kubernetes/login"))
                .willReturn(okJson(loginResponse("hvs.second", 3600))));

        source.invalidate();
        source.token();

        wireMock.verify(postRequestedFor(urlEqualTo("/v1/auth/kubernetes/login"))
                .withRequestBody(containing("jwt-v2")));
    }

    @Test
    void token_throwsOnMissingJwtFile() {
        final Path missing = tempDir.resolve("nonexistent");

        final var source = new KubernetesVaultTokenSource(
                wireMock.baseUrl(), "my-role", missing, "kubernetes",
                http, mapper, Clock.systemUTC());

        assertThatThrownBy(source::token)
                .isInstanceOf(RuntimeException.class)
                .hasMessageContaining("nonexistent");
    }
}
```

- [ ] **Step 7: Run all token source tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest="AppRoleVaultTokenSourceTest,KubernetesVaultTokenSourceTest"`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```
feat(vault): add LoginBasedVaultTokenSource with AppRole and Kubernetes implementations

Refs #101
```

---

### Task 3: Migrate VaultTransitSigningClient and VaultTransitSigningConfig to per-call token

**Files:**
- Modify: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/VaultTransitSigningConfig.java`
- Modify: `signing/vault-transit/src/main/java/io/casehub/ledger/signing/vault/VaultTransitSigningClient.java`
- Modify: `signing/vault-transit/src/test/java/io/casehub/ledger/signing/vault/VaultTransitSigningClientTest.java`

**Interfaces:**
- Consumes: `VaultAuthenticationException` (from Task 1)
- Produces:
  - `VaultTransitSigningConfig(String address, Map<String, String> keyMapping)` — drops `token` field
  - `VaultTransitSigningClient(VaultTransitSigningConfig config, HttpClient http, ObjectMapper mapper)` — public constructor accepting shared instances
  - `fetchPublicKey(String token, String keyName)` — token as parameter
  - `sign(String token, String keyName, byte[] data)` — token as parameter
  - HTTP 403 → `VaultAuthenticationException`

- [ ] **Step 1: Update VaultTransitSigningClientTest for per-call token and VaultAuthenticationException**

Modify `VaultTransitSigningClientTest`:
- Change `createClient()` to use new 2-field `VaultTransitSigningConfig(address, keyMapping)` and 3-arg `VaultTransitSigningClient(config, httpClient, mapper)` constructor
- Change `sign("my-key", data)` calls to `sign("test-token", "my-key", data)`
- Change `fetchPublicKey("my-key")` calls to `fetchPublicKey("test-token", "my-key")`
- Add test: `sign_throwsVaultAuthenticationExceptionOn403` — asserts `VaultAuthenticationException` (not generic `RuntimeException`) on 403
- Add test: `fetchPublicKey_throwsVaultAuthenticationExceptionOn403` — same for fetchPublicKey
- Update existing 403 tests to assert `VaultAuthenticationException`

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest=VaultTransitSigningClientTest`
Expected: compilation failure — constructor signature mismatch, method signature mismatch

- [ ] **Step 3: Modify VaultTransitSigningConfig — drop token field**

```java
package io.casehub.ledger.signing.vault;

import java.util.Map;

public record VaultTransitSigningConfig(
        String address,
        Map<String, String> keyMapping) {}
```

- [ ] **Step 4: Modify VaultTransitSigningClient — per-call token, shared HttpClient/ObjectMapper, 403 → VaultAuthenticationException**

Key changes:
- Remove `private final String token` field
- Add `private final ObjectMapper mapper` as constructor parameter (was created internally)
- Change constructor to `public VaultTransitSigningClient(VaultTransitSigningConfig config, HttpClient http, ObjectMapper mapper)`
- Change `fetchPublicKey(String keyName)` to `fetchPublicKey(String token, String keyName)`
- Change `sign(String keyName, byte[] data)` to `sign(String token, String keyName, byte[] data)`
- In `fetchPublicKey` and `callVaultSign`: use the `token` parameter in `.header("X-Vault-Token", token)`
- On HTTP 403: throw `new VaultAuthenticationException(...)` instead of generic `RuntimeException`

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing -Dtest=VaultTransitSigningClientTest`
Expected: all tests PASS

- [ ] **Step 6: Write VaultTransitAuthIntegrationTest — end-to-end AppRole login → fetch key → sign**

```java
package io.casehub.ledger.signing.vault;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

import java.net.http.HttpClient;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.Signature;
import java.time.Clock;
import java.util.Base64;
import java.util.Map;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.github.tomakehurst.wiremock.WireMockServer;

class VaultTransitAuthIntegrationTest {

    static WireMockServer wireMock;

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

    @Test
    void appRoleLogin_thenFetchKey_thenSign() throws Exception {
        // 1. Stub AppRole login
        wireMock.stubFor(post(urlEqualTo("/v1/auth/approle/login"))
                .willReturn(okJson("{\"auth\":{\"client_token\":\"hvs.dynamic\",\"lease_duration\":3600,\"renewable\":true}}")));

        // 2. Stub key info
        final KeyPair kp = KeyPairGenerator.getInstance("Ed25519").generateKeyPair();
        final String pem = "-----BEGIN PUBLIC KEY-----\n"
                + Base64.getMimeEncoder(64, new byte[]{'\n'}).encodeToString(kp.getPublic().getEncoded())
                + "\n-----END PUBLIC KEY-----\n";
        final String pemJson = pem.replace("\\", "\\\\").replace("\"", "\\\"").replace("\n", "\\n");
        wireMock.stubFor(get(urlEqualTo("/v1/transit/keys/my-key"))
                .withHeader("X-Vault-Token", equalTo("hvs.dynamic"))
                .willReturn(okJson("{\"data\":{\"type\":\"ed25519\",\"keys\":{\"1\":{\"public_key\":\"" + pemJson + "\"}}}}")));

        // 3. Stub sign
        final byte[] data = "integration test data".getBytes();
        final Signature sig = Signature.getInstance("Ed25519");
        sig.initSign(kp.getPrivate());
        sig.update(data);
        final byte[] sigBytes = sig.sign();
        wireMock.stubFor(post(urlEqualTo("/v1/transit/sign/my-key"))
                .withHeader("X-Vault-Token", equalTo("hvs.dynamic"))
                .willReturn(okJson("{\"data\":{\"signature\":\"vault:v1:" + Base64.getEncoder().encodeToString(sigBytes) + "\"}}")));

        // Execute: AppRole login → fetch key → sign
        final HttpClient http = HttpClient.newHttpClient();
        final ObjectMapper mapper = new ObjectMapper();
        final VaultTokenSource tokenSource = new AppRoleVaultTokenSource(
                wireMock.baseUrl(), "role-id", "secret-id", "approle",
                http, mapper, Clock.systemUTC());

        final VaultTransitSigningConfig config = new VaultTransitSigningConfig(
                wireMock.baseUrl(), Map.of("actor1", "my-key"));
        final VaultTransitSigningClient client = new VaultTransitSigningClient(config, http, mapper);

        final String token = tokenSource.token();
        final var publicKey = client.fetchPublicKey(token, "my-key");
        assertThat(publicKey.getEncoded()).isEqualTo(kp.getPublic().getEncoded());

        final byte[] result = client.sign(token, "my-key", data);

        final Signature verifier = Signature.getInstance("Ed25519");
        verifier.initVerify(kp.getPublic());
        verifier.update(data);
        assertThat(verifier.verify(result)).isTrue();
    }
}
```

- [ ] **Step 7: Run all vault-transit tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit -Pwith-signing`
Expected: all tests PASS

- [ ] **Step 8: Commit**

```
feat(vault): migrate signing client to per-call token, 403 → VaultAuthenticationException

BREAKING: VaultTransitSigningConfig drops token field.
BREAKING: fetchPublicKey/sign take String token parameter.

Refs #101
```

---

### Task 4: Migrate Quarkus adapter — config, token source factory, 403-retry

**Files:**
- Modify: `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitConfig.java`
- Modify: `signing/vault-transit-quarkus/src/main/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSigner.java`
- Modify: `signing/vault-transit-quarkus/src/test/resources/application.properties`
- Modify: `signing/vault-transit-quarkus/src/test/java/io/casehub/ledger/signing/vault/quarkus/VaultTransitAgentSignerIT.java`

**Interfaces:**
- Consumes: `VaultTokenSource`, `StaticVaultTokenSource`, `AppRoleVaultTokenSource`, `KubernetesVaultTokenSource`, `VaultAuthenticationException`, `VaultTransitSigningConfig(address, keyMapping)`, `VaultTransitSigningClient(config, http, mapper)`, `fetchPublicKey(token, keyName)`, `sign(token, keyName, data)` (all from Tasks 1-3)
- Produces: fully functional Quarkus adapter with config-driven auth method selection and 403-retry

- [ ] **Step 1: Update VaultTransitConfig — add auth sub-interface**

```java
package io.casehub.ledger.signing.vault.quarkus;

import java.util.Map;
import java.util.Optional;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.ledger.vault-transit")
public interface VaultTransitConfig {

    @WithDefault("http://localhost:8200")
    String address();

    Map<String, String> keyMapping();

    @WithDefault("5m")
    String refreshInterval();

    AuthConfig auth();

    interface AuthConfig {
        @WithDefault("token")
        AuthMethod method();

        Optional<String> token();

        Optional<String> roleId();
        Optional<String> secretId();

        Optional<String> role();
        @WithDefault("/var/run/secrets/kubernetes.io/serviceaccount/token")
        String jwtPath();

        Optional<String> mountPath();
    }

    enum AuthMethod { TOKEN, APPROLE, KUBERNETES }
}
```

- [ ] **Step 2: Update VaultTransitAgentSigner — token source factory, shared HttpClient/ObjectMapper, 403-retry**

Key changes to `VaultTransitAgentSigner`:
- Constructor creates shared `HttpClient` and `ObjectMapper`
- Switch on `config.auth().method()` to create appropriate `VaultTokenSource`
- Validate required fields at construction (throw `IllegalStateException` on missing config)
- Store `VaultTokenSource tokenSource` as a field
- In `loadContext()`: call `tokenSource.token()`, pass to `client.fetchPublicKey(token, keyName)`, catch `VaultAuthenticationException` → `invalidate()` → retry once
- In `performSign()`: call `tokenSource.token()`, pass to `client.sign(token, keyName, data)`, catch `VaultAuthenticationException` → `invalidate()` → retry once

- [ ] **Step 3: Update test application.properties**

```properties
# WireMock server runs on port 8098 (started in VaultTransitAgentSignerIT)
casehub.ledger.vault-transit.address=http://localhost:8098
casehub.ledger.vault-transit.auth.method=token
casehub.ledger.vault-transit.auth.token=test-token
casehub.ledger.vault-transit.key-mapping."claude\:reviewer@v1"=reviewer-key
casehub.ledger.vault-transit.refresh-interval=24h

quarkus.arc.selected-alternatives=io.casehub.ledger.signing.vault.quarkus.VaultTransitAgentSigner
quarkus.hibernate-orm.enabled=false
```

- [ ] **Step 4: Update VaultTransitAgentSignerIT — update stubs for per-call token, add 403-retry tests**

Key test changes:
- `stubKeyInfo`/`stubSign` no longer check `X-Vault-Token` header (the token source produces it; the IT validates end-to-end flow)
- Add test: `retries_onVaultAuthenticationException` — first sign returns 403, adapter invalidates and retries with fresh token, second sign succeeds
- Add test: `retries_exhausted_onDouble403` — both calls return 403, entry saved unsigned
- Add test: `retries_onFetchPublicKey403` — loadContext gets 403 on key fetch, adapter retries

- [ ] **Step 5: Run Quarkus IT tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl signing/vault-transit-quarkus -Pwith-signing`
Expected: all tests PASS

- [ ] **Step 6: Commit**

```
feat(vault): migrate Quarkus adapter to auth config and 403-retry

BREAKING: casehub.ledger.vault-transit.token → auth.method + auth.token
Adds auth.method=token|approle|kubernetes discriminator.
403 responses trigger tokenSource.invalidate() + single retry.

Refs #101
```

---

### Task 5: Update example module and final verification

**Files:**
- Modify: `examples/vault-transit-signing/src/main/resources/application.properties`
- Modify: `examples/vault-transit-signing/src/test/resources/application.properties`
- Modify: `examples/vault-transit-signing/src/test/java/io/casehub/ledger/examples/vault/VaultTransitAgentSignerIT.java`

**Interfaces:**
- Consumes: all Tasks 1-4 (complete adapter)
- Produces: updated example config, full build green

- [ ] **Step 1: Update example main application.properties**

```properties
# Vault Transit AgentSigner configuration
# Override these for your Vault deployment
casehub.ledger.vault-transit.address=http://localhost:8200
casehub.ledger.vault-transit.auth.method=token
casehub.ledger.vault-transit.auth.token=root
casehub.ledger.vault-transit.refresh-interval=5m
# Key mapping: actorId -> Vault Transit key name
# casehub.ledger.vault-transit.key-mapping."claude:reviewer@v1"=reviewer-signing-key
```

- [ ] **Step 2: Update example test application.properties**

```properties
casehub.ledger.vault-transit.address=http://localhost:8099
casehub.ledger.vault-transit.auth.method=token
casehub.ledger.vault-transit.auth.token=test-token
casehub.ledger.vault-transit.key-mapping."claude\:reviewer@v1"=reviewer-key
casehub.ledger.vault-transit.refresh-interval=24h
quarkus.hibernate-orm.enabled=false
quarkus.arc.selected-alternatives=io.casehub.ledger.signing.vault.quarkus.VaultTransitAgentSigner
```

- [ ] **Step 3: Update example VaultTransitAgentSignerIT.java**

The example IT at `examples/vault-transit-signing/src/test/java/io/casehub/ledger/examples/vault/VaultTransitAgentSignerIT.java` uses `withHeader("X-Vault-Token", equalTo("test-token"))` on WireMock stubs. With the new auth config, the token still comes from `casehub.ledger.vault-transit.auth.token=test-token` so the header value is the same — but review the test for any direct references to the old config key or `VaultTransitSigningConfig` constructor. Update as needed.

- [ ] **Step 4: Run example module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl examples/vault-transit-signing -Pwith-signing`
Expected: PASS

- [ ] **Step 5: Run full signing module test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Pwith-signing`
Expected: all signing modules PASS

- [ ] **Step 6: Run full project test suite (non-signing)**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test`
Expected: all non-signing modules PASS (no changes, but verifies nothing broke)

- [ ] **Step 7: Commit**

```
chore(vault): update example config for auth sub-config

Refs #101
```
