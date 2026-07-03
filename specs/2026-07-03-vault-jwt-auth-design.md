# Vault JWT Auth Method — Design Spec

**Issue:** casehubio/ledger#170
**Branch:** issue-170-vault-oidc-auth
**Date:** 2026-07-03

## Problem

The Vault Transit signing adapter supports three auth methods: TOKEN (static), APPROLE, and KUBERNETES. Non-Kubernetes environments that use OIDC-compliant identity providers (Okta, Auth0, Keycloak, Azure AD) have no way to authenticate to Vault. Vault's JWT auth backend (`/v1/auth/<mount>/login`) accepts a pre-existing JWT and returns a Vault token, but no `VaultTokenSource` implementation exists for it.

## Decision: JWT, not OIDC

The issue title says "OIDC" but the actual mechanism is Vault's JWT auth method. This class submits a pre-existing JWT to Vault — it does no OIDC discovery, no browser flow, no token endpoint call. The "OIDC" label describes where the JWT may have originated, not what this class does.

Naming follows existing convention (classes named after the Vault auth method):
- `AppRoleVaultTokenSource` → Vault `approle`
- ~~`KubernetesVaultTokenSource`~~ → consolidated into `JwtVaultTokenSource` (see below)
- `JwtVaultTokenSource` → Vault `jwt` (and `kubernetes`, via `fromFile()`)

Enum value: `AuthMethod.JWT`.

### Consolidation: KubernetesVaultTokenSource → JwtVaultTokenSource

`KubernetesVaultTokenSource` and `JwtVaultTokenSource` share identical `loginPath()` and `loginRequestBody()` semantics — both submit `{"role":"<role>","jwt":"<jwt>"}` to `/v1/auth/<mountPath>/login`. The only differences are the mount path default and the JWT acquisition mechanism. Once `JwtVaultTokenSource` exists with `Supplier<String>`, `KubernetesVaultTokenSource` is `JwtVaultTokenSource.fromFile(addr, role, path, "kubernetes", ...)`.

**Action:** Delete `KubernetesVaultTokenSource`. The KUBERNETES case in `createTokenSource()` uses `JwtVaultTokenSource.fromFile(...)` with `mountPath="kubernetes"` and the default service account token path. `AuthMethod.KUBERNETES` remains as a config convenience — it sets the mount path and default file path. Existing `KubernetesVaultTokenSourceTest` tests are migrated to `JwtVaultTokenSourceTest` to cover the `fromFile()` path with `mountPath="kubernetes"`.

## Design

### Pure Java class (`vault-transit` module)

**New class:** `JwtVaultTokenSource extends LoginBasedVaultTokenSource`

**Package:** `io.casehub.ledger.signing.vault`

**Constructor:** `(String vaultAddress, String role, Supplier<String> jwtSupplier, String mountPath, HttpClient http, ObjectMapper mapper, Clock clock)`

All constructor parameters validated with `Objects.requireNonNull` (consistent with `AppRoleVaultTokenSource`): `role`, `jwtSupplier` are non-null. `mountPath` defaults to `"jwt"` if null.

**Factory method:** `fromFile(String vaultAddress, String role, Path jwtPath, String mountPath, HttpClient http, ObjectMapper mapper, Clock clock)` — wraps `Files.readString(jwtPath).trim()` in a Supplier with IOException → UncheckedIOException conversion.

**Why Supplier<String>:** The JWT credential inherently varies in how it's obtained (file, config property, env var, API call). `Supplier<String>` is `java.util.function` — zero-cost, no new types. It eliminates internal branching that a dual-constructor approach (file path vs static string) would require. `fromFile()` covers the common file-based case. The Supplier is called on every login attempt, so file-based JWTs are re-read on each login (supports rotation).

**loginPath():** `/v1/auth/<mountPath>/login` (mount path defaults to `jwt`)

**loginRequestBody():** `{"role":"<role>","jwt":"<jwtSupplier.get()>"}` — the Supplier result is validated with `Objects.requireNonNull(jwtSupplier.get(), "JWT supplier returned null")` before interpolation, failing fast instead of sending the literal string `"null"` to Vault.

### Quarkus config changes (`vault-transit-quarkus` module)

**VaultTransitConfig.AuthConfig changes:**

1. Add `JWT` to enum: `enum AuthMethod { TOKEN, APPROLE, KUBERNETES, JWT }`

2. Change `jwtPath()` from `@WithDefault("/var/run/secrets/kubernetes.io/serviceaccount/token") String` to `Optional<String>`. The Kubernetes default path moves to the adapter code (`authConfig.jwtPath().orElse("/var/run/secrets/...")` in the KUBERNETES case). This makes the default auth-method-specific — JWT auth has no sensible default path, so it requires explicit config.

3. Add `Optional<String> jwt()` for static JWT strings. Quarkus config substitution applies automatically — `casehub.ledger.vault-transit.auth.jwt=${MY_JWT_TOKEN}` resolves from environment variables at startup.

4. Update Javadocs: `role()` → "Role for Kubernetes or JWT auth methods." `jwtPath()` → "Path to JWT file for KUBERNETES or JWT auth methods." `jwt()` → "Static JWT string for JWT auth method. Mutually exclusive with jwt-path." `mountPath()` → "Vault mount path for AppRole, Kubernetes, or JWT auth. Defaults to the auth method name. For Vault OIDC auth backends, use `mountPath=oidc` with `method=jwt` — the JWT and OIDC auth methods share the same login API."

**Validation (in adapter code):**
- `method=JWT` requires `role`
- `method=JWT` requires exactly one of `jwtPath` or `jwt` (both → error, neither → error)

### Adapter wiring (`VaultTransitAgentSigner`)

Update `KUBERNETES` case to use consolidated class:
```java
case KUBERNETES -> {
    String role = authConfig.role().orElseThrow(...);
    String path = authConfig.jwtPath().orElse("/var/run/secrets/kubernetes.io/serviceaccount/token");
    String mountPath = authConfig.mountPath().orElse("kubernetes");
    yield JwtVaultTokenSource.fromFile(config.address(), role, Path.of(path), mountPath,
            httpClient, objectMapper, Clock.systemUTC());
}
```

Add `JWT` case to `createTokenSource()` switch:
- `jwtPath` present → `JwtVaultTokenSource.fromFile(...)`
- `jwt` present → `new JwtVaultTokenSource(..., () -> jwt, ...)`

### Error handling

No changes to the error handling architecture. The existing flow is correct:
- `LoginBasedVaultTokenSource.login()` throws `RuntimeException` on auth-backend 403 (credentials rejected — retrying with same credentials won't help)
- `VaultTransitSigningClient` throws `VaultAuthenticationException` on Transit API 403 (token expired — retry with fresh token helps)
- The adapter's catch-and-retry logic only fires for the second kind

### Housekeeping

- **ARC42STORIES §12 Tech Debt:** Remove the `Vault AppRole/OIDC auth for VaultTransitAgentSigner not implemented | L5 | #101` entry — issue #101 is CLOSED (delivered AppRole + Kubernetes auth), and this spec closes the remaining JWT auth gap tracked by #170. If browser-based OIDC is still wanted, a narrower entry referencing the new issue replaces it.
- **GitHub issue:** File a GitHub issue for "Browser-based OIDC flow (two-step auth URL + callback)" to track the deferred out-of-scope item.

## Tests

### JwtVaultTokenSourceTest (vault-transit, WireMock)

| Test | Verifies |
|------|----------|
| `token_logsInWithSupplierJwt` | Supplier JWT submitted to `/v1/auth/jwt/login`, token returned |
| `token_fromFile_readsJwtFromFile` | `fromFile()` reads file, sends JWT in body |
| `token_returnsCachedWithinTTL` | Second call doesn't re-login |
| `token_reLoginsAfterInvalidate` | `invalidate()` forces fresh login; Supplier called again |
| `token_throwsOnLoginFailure` | 403 from auth backend → RuntimeException |
| `token_usesCustomMountPath` | Custom mount → `/v1/auth/<custom>/login` |
| `token_fromFile_throwsOnMissingFile` | Missing file → UncheckedIOException |
| `token_supplierCalledOnEachLogin` | Supplier invoked per login, not cached |
| `token_supplierReturningNull_throwsNPE` | `jwtSupplier.get()` returning null → NullPointerException with clear message |
| `token_fromFile_kubernetesMountPath` | `fromFile()` with `mountPath="kubernetes"` → `/v1/auth/kubernetes/login` (migrated from `KubernetesVaultTokenSourceTest`) |
| `token_fromFile_reReadsOnReLogin` | File-based JWT re-read on invalidate + re-login (migrated from `KubernetesVaultTokenSourceTest`) |

### VaultTransitAgentSignerIT updates (vault-transit-quarkus, @QuarkusTest)

| Test | Verifies |
|------|----------|
| `jwtAuth_fileBasedJwt_signsSuccessfully` | Full pipeline: JWT file → login → sign |
| `jwtAuth_staticJwt_signsSuccessfully` | Full pipeline: static JWT → login → sign |
| `jwtAuth_403Retry_invalidatesAndRetries` | Transit 403 → invalidation → re-login → retry |
| `jwtAuth_missingRole_failsFast` | `method=JWT` without `role` → IllegalStateException |
| `jwtAuth_bothJwtAndJwtPath_failsFast` | Both set → IllegalStateException |
| `jwtAuth_neitherJwtNorJwtPath_failsFast` | Neither set → IllegalStateException |
| `kubernetesAuth_defaultJwtPath_configPath` | KUBERNETES case through `createTokenSource()` applies default path correctly after `jwtPath` type change to `Optional<String>` |

## Files changed

| File | Change |
|------|--------|
| `signing/vault-transit/src/main/java/.../JwtVaultTokenSource.java` | New |
| `signing/vault-transit/src/main/java/.../KubernetesVaultTokenSource.java` | Deleted (consolidated into JwtVaultTokenSource) |
| `signing/vault-transit/src/test/java/.../JwtVaultTokenSourceTest.java` | New (includes migrated KubernetesVaultTokenSourceTest cases) |
| `signing/vault-transit/src/test/java/.../KubernetesVaultTokenSourceTest.java` | Deleted |
| `signing/vault-transit-quarkus/src/main/java/.../VaultTransitConfig.java` | Add JWT enum, change jwtPath to Optional, add jwt property |
| `signing/vault-transit-quarkus/src/main/java/.../VaultTransitAgentSigner.java` | Add JWT case, update KUBERNETES case to use JwtVaultTokenSource.fromFile() |
| `signing/vault-transit-quarkus/src/test/java/.../VaultTransitAgentSignerIT.java` | Add JWT auth tests + KUBERNETES regression test |

## Out of scope

- Browser-based OIDC flow (two-step auth URL + callback) — tracked by GitHub issue (to be filed at implementation time)
- Custom JWT acquisition plugins (API-based JWT fetching) — users implement `VaultTokenSource` directly
- `LoginBasedVaultTokenSource.login()` throwing `VaultAuthenticationException` on 403 — the current RuntimeException is correct for the retry semantics
