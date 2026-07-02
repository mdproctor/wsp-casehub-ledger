# Vault AppRole/Kubernetes Auth for VaultTransitAgentSigner

**Issue:** casehubio/ledger#101
**Branch:** issue-101-vault-approle-oidc-auth
**Date:** 2026-07-02

## Problem

`VaultTransitSigningClient` authenticates with a static Vault token. Production
deployments require dynamic auth — AppRole (machine-to-machine) or Kubernetes
(workload identity federation). The token has a TTL and must be renewed.

**Terminology note:** Issue #101 uses "OIDC" in its title and body, but the
described use case (Kubernetes service account JWT → Vault login endpoint) is
Vault's **Kubernetes** auth method (`auth/kubernetes`), not OIDC
(`auth/oidc`). These are distinct Vault auth backends. Kubernetes auth validates
service account JWTs against the Kubernetes API server; OIDC auth uses any
OIDC-compliant IdP (Okta, Auth0, etc.) with browser-based or JWT flows. This
spec implements Kubernetes auth, which is the correct method for Kubernetes
workloads. Actual OIDC auth (for non-Kubernetes environments) is out of scope
and tracked as casehubio/ledger#170.

## Approach — Lazy renewal in the token source

A `VaultTokenSource` interface provides dynamic token resolution.
Implementations cache the token internally and re-login when expired.
The Quarkus adapter calls `tokenSource.token()` before each signing client
method and passes the `String` token as a parameter — the signing client
receives a bare string and never knows or cares which auth method produced it.
This keeps the pure Java client stateless with respect to auth, consistent
with how it is tested today (static tokens).

Renewal is lazy: on each `token()` call, check if the cached token is within
30 seconds of expiry; if so, re-login synchronously before returning. The 30s
buffer covers clock skew. The extra latency hits once per token TTL (typically
60 minutes) — ~100-200ms per hour is noise for a `@PrePersist` pipeline.

Proactive renewal (background timer) was evaluated and rejected: it adds
background threads, timer lifecycle, shutdown hooks, and interval coordination
for a latency saving that doesn't matter.

## Token source hierarchy

```
VaultTokenSource                              (interface: String token())
├── StaticVaultTokenSource(String token)      (wraps a string — dev/testing)
└── LoginBasedVaultTokenSource                (abstract — login + cache + expiry)
    ├── AppRoleVaultTokenSource               (role_id + secret_id)
    └── KubernetesVaultTokenSource            (reads JWT from file + role)
```

All implementations live in the pure Java module (`vault-transit/`).

### VaultTokenSource

```java
public interface VaultTokenSource {
    String token();
    void invalidate();
}
```

`invalidate()` signals that the last token returned by `token()` was rejected
by Vault (e.g. 403). The token source clears its cached token so the next
`token()` call triggers re-login. The caller (adapter) is responsible for
calling `invalidate()` on auth failure — the token source itself cannot detect
out-of-band revocation.

### StaticVaultTokenSource

Wraps a `String`. `token()` returns it unchanged. `invalidate()` is a no-op
(static tokens cannot be refreshed). Used for dev and testing.

### LoginBasedVaultTokenSource

Abstract base. Owns the token cache: stores `clientToken` + `expiresAt` as
`volatile` fields. Thread safety via double-checked locking — the common path
(cached token valid, ~59 minutes out of 60) is lock-free; `synchronized` is
only acquired during the brief renewal window. On Java 26 JVM (biased locking
removed since JDK 18), this avoids thin-lock acquisition on every `token()` call
under concurrent `@PrePersist` load.

```java
private volatile String clientToken;
private volatile Instant expiresAt = Instant.EPOCH;

public String token() {
    if (!isExpired()) return clientToken; // fast path, no lock
    synchronized (this) {
        if (!isExpired()) return clientToken; // recheck under lock
        login();
        return clientToken;
    }
}

public void invalidate() {
    clientToken = null;
    expiresAt = Instant.EPOCH;
}
```

`login()` must set `clientToken` before `expiresAt` to ensure correct
visibility ordering — a concurrent reader may see the new token with the
old (expired) `expiresAt`, triggering a harmless re-login, but never an old
token with a new `expiresAt`.

Constructor takes:
- `String vaultAddress` — base URL (e.g. `http://vault:8200`)
- `HttpClient http` — shared HTTP client (same instance as the signing client)
- `ObjectMapper mapper` — shared Jackson mapper for JSON response parsing
- `Clock clock` — defaults to `Clock.systemUTC()`; tests use `Clock.fixed()`

Abstract methods:
- `String loginPath()` — e.g. `/v1/auth/approle/login`
- `String loginRequestBody()` — JSON body for the login POST

Login response parsing uses Jackson (`ObjectMapper`) to extract
`auth.client_token` and `auth.lease_duration` from the standard Vault login
response. The mapper is shared with `VaultTransitSigningClient` (both live
in the same pure Java module and Jackson is already a dependency).

The expiry buffer is clamped: `expiresAt = now + leaseDuration - min(30, leaseDuration / 2)`.
This prevents a login storm when Vault returns an unexpectedly short
`lease_duration` (e.g. < 30 seconds from a misconfigured policy). If
`leaseDuration < 10`, a warning is logged. Normal TTLs (60+ minutes)
use the full 30-second buffer.

### AppRoleVaultTokenSource

Constructor: `(String vaultAddress, String roleId, String secretId, String mountPath)`

- `mountPath` defaults to `"approle"`. Configurable for custom Vault mount points.
- `loginPath()` → `/v1/auth/<mountPath>/login`
- `loginRequestBody()` → `{"role_id":"...","secret_id":"..."}`

### KubernetesVaultTokenSource

Constructor: `(String vaultAddress, String role, Path jwtPath, String mountPath)`

- `mountPath` defaults to `"kubernetes"`.
- `jwtPath` defaults to `/var/run/secrets/kubernetes.io/serviceaccount/token`.
- `loginRequestBody()` re-reads the JWT file on every login call. Kubernetes
  rotates service account tokens (default: 1 hour). Reading at login time
  ensures we always use the current token.
- `loginPath()` → `/v1/auth/<mountPath>/login`
- `loginRequestBody()` → `{"role":"...","jwt":"<contents of jwtPath>"}`

## Changes to VaultTransitSigningConfig

Breaking change — `token` field removed (auth is no longer the config's concern):

```java
// BEFORE
public record VaultTransitSigningConfig(
    String address,
    String token,
    Map<String, String> keyMapping) {}

// AFTER
public record VaultTransitSigningConfig(
    String address,
    Map<String, String> keyMapping) {}
```

The config is a pure data carrier with no behaviour. Auth is managed by the
adapter, which calls `tokenSource.token()` and passes the string to the client.

## Changes to VaultTransitSigningClient

The client no longer stores a token. Both `fetchPublicKey()` and `sign()` take
a `String token` parameter:

```java
public PublicKey fetchPublicKey(String token, String keyName) { ... }
public byte[] sign(String token, String keyName, byte[] data) { ... }
```

The constructor drops the `token` extraction from config. The constructor's
`HttpClient` parameter is promoted from test-visible to public — the adapter
creates a shared `HttpClient` and passes it to both the signing client and the
token source, avoiding duplicate connection pools against the same Vault server.

Similarly, the `ObjectMapper` is accepted as a constructor parameter and shared
with the token source.

## Error handling

Follows the `AgentSigner` contract: throw `RuntimeException` for transient failures.

- **Login fails:** `token()` throws → adapter's `performSign()` throws →
  entry saved unsigned. Nothing cached on failure — next call retries.
- **Token expired between check and use:** Vault returns 403 on the sign call →
  signing client throws → adapter catches, invalidates token source, retries
  once with a fresh token (see below).
- **Out-of-band token revocation:** Vault token revoked externally (operator
  action, Vault restart, policy change) while the cached token appears valid.
  Without invalidation, every signing call fails for the remaining token TTL
  (up to ~60 minutes). The adapter's 403-retry with `invalidate()` bounds the
  outage to one failed call per actor.

### 403-retry in the adapter, not the signing client

The signing client (`VaultTransitSigningClient`) is stateless with respect to
auth — it receives a `String token` per call and throws on 403. The 403-retry
lives in `VaultTransitAgentSigner` (Quarkus adapter), which owns the
`VaultTokenSource`:

1. Adapter calls `tokenSource.token()` → gets cached token
2. Passes token to `client.sign(token, keyName, data)`
3. Client throws on 403
4. Adapter calls `tokenSource.invalidate()` → clears cached token
5. Adapter calls `tokenSource.token()` → triggers re-login, gets fresh token
6. Adapter retries `client.sign(newToken, keyName, data)` — once only
7. If retry also fails → throws → entry saved unsigned

This preserves clean layering: the signing client doesn't know about token
sources or lifecycle. The adapter bridges auth and signing. The retry is
bounded (once) and the cost is one re-login attempt (~100-200ms).

The same pattern applies to `fetchPublicKey()` calls in `loadContext()`.

## Quarkus configuration model

`VaultTransitConfig` gains an `auth()` sub-interface:

```java
@ConfigMapping(prefix = "casehub.ledger.vault-transit")
public interface VaultTransitConfig {
    String address();
    Map<String, String> keyMapping();
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

The old top-level `token()` method is removed. `casehub.ledger.vault-transit.token`
becomes `casehub.ledger.vault-transit.auth.token`. Breaking config change — no
deployed instances, costs nothing.

`VaultTransitAgentSigner` constructor creates a shared `HttpClient` and
`ObjectMapper`, switches on `auth().method()` to create the appropriate
`VaultTokenSource`, and passes the shared instances to both the token source
and the signing client. Required fields are validated at startup.

### Configuration examples

```properties
# Static token (dev)
casehub.ledger.vault-transit.auth.method=token
casehub.ledger.vault-transit.auth.token=hvs.my-dev-token

# AppRole
casehub.ledger.vault-transit.auth.method=approle
casehub.ledger.vault-transit.auth.role-id=my-role-id
casehub.ledger.vault-transit.auth.secret-id=my-secret-id

# Kubernetes
casehub.ledger.vault-transit.auth.method=kubernetes
casehub.ledger.vault-transit.auth.role=my-vault-role

# Custom mount path (any method)
casehub.ledger.vault-transit.auth.mount-path=custom-approle
```

## Testing

### Layer 1 — Pure Java unit tests (`vault-transit/`)

- `StaticVaultTokenSourceTest` — `token()` returns the string
- `AppRoleVaultTokenSourceTest` — WireMock: happy path, caching, expiry/re-login,
  login failure, custom mount path. Uses `Clock.fixed()` for time control.
- `KubernetesVaultTokenSourceTest` — WireMock + temp JWT file: happy path,
  JWT re-read on re-login, missing JWT file error
- `VaultTransitSigningClientTest` — updated: token passed per-call, config drops `token` field

### Layer 2 — Integration test (`vault-transit/`)

`VaultTransitAuthIntegrationTest` — WireMock end-to-end: AppRole login →
fetch public key → sign data. Validates the full flow through
`VaultTransitSigningClient` with `AppRoleVaultTokenSource`.

### Layer 3 — Quarkus CDI test (`vault-transit-quarkus/`)

`VaultTransitAgentSignerIT` — updated config. Config validation tests for
missing required fields per auth method.

No real Vault dev server. No Docker requirement.

## File inventory

### Pure Java (`signing/vault-transit/`)

| File | Change |
|------|--------|
| `VaultTokenSource.java` | NEW |
| `StaticVaultTokenSource.java` | NEW |
| `LoginBasedVaultTokenSource.java` | NEW |
| `AppRoleVaultTokenSource.java` | NEW |
| `KubernetesVaultTokenSource.java` | NEW |
| `VaultTransitSigningConfig.java` | BREAKING — drop `token`, keep `(address, keyMapping)` |
| `VaultTransitSigningClient.java` | BREAKING — `token` as per-call parameter, accept shared `HttpClient` + `ObjectMapper` |
| `VaultTransitContext.java` | unchanged |
| Tests: 4 new, 1 modified | see Testing section |

### Quarkus adapter (`signing/vault-transit-quarkus/`)

| File | Change |
|------|--------|
| `VaultTransitConfig.java` | BREAKING — drop `token()`, add `auth()` |
| `VaultTransitAgentSigner.java` | MODIFY — create token source from config |
| `VaultTransitAgentSignerIT.java` | MODIFY — update config structure |

**Total: 9 new files, 6 modified. Zero new modules. Zero new dependencies.**

New source: 5 (`VaultTokenSource`, `StaticVaultTokenSource`, `LoginBasedVaultTokenSource`,
`AppRoleVaultTokenSource`, `KubernetesVaultTokenSource`).
New test: 4 (`StaticVaultTokenSourceTest`, `AppRoleVaultTokenSourceTest`,
`KubernetesVaultTokenSourceTest`, `VaultTransitAuthIntegrationTest`).
Modified source: 4. Modified test: 2.

## Platform coherence

- Module tier structure: ✅ pure Java SPI + Quarkus CDI adapter
- No heavy SDK types in SPI: ✅ `VaultTokenSource.token()` returns `String`
- Config prefix: ✅ `casehub.ledger.vault-transit.auth.*`
- Error handling: ✅ follows `AgentSigner` contract
- No impact on other signing modules (AWS/GCP/Azure use SDK credential chains)
- No schema, entity, or repository changes
