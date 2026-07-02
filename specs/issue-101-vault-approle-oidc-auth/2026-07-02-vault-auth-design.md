# Vault AppRole/Kubernetes Auth for VaultTransitAgentSigner

**Issue:** casehubio/ledger#101
**Branch:** issue-101-vault-approle-oidc-auth
**Date:** 2026-07-02

## Problem

`VaultTransitSigningClient` authenticates with a static Vault token. Production
deployments require dynamic auth — AppRole (machine-to-machine) or Kubernetes
(workload identity federation). The token has a TTL and must be renewed.

## Approach — Lazy renewal in the token source

A `VaultTokenSource` interface replaces the static `String token` in
`VaultTransitSigningConfig`. Implementations cache the token internally and
re-login when expired. The signing client calls `tokenSource.token()` on each
HTTP request — it never knows or cares which auth method produced the token.

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
}
```

### StaticVaultTokenSource

Wraps a `String`. `token()` returns it unchanged. Used for dev and testing.

### LoginBasedVaultTokenSource

Abstract base. Owns the token cache: stores `clientToken` + `expiresAt`.
Thread safety via `synchronized` on the login check — login is ~50ms, acceptable
for a once-per-hour event.

Constructor takes:
- `String vaultAddress` — base URL (e.g. `http://vault:8200`)
- `HttpClient http` — shared HTTP client
- `Clock clock` — defaults to `Clock.systemUTC()`; tests use `Clock.fixed()`

Abstract methods:
- `String loginPath()` — e.g. `/v1/auth/approle/login`
- `String loginRequestBody()` — JSON body for the login POST

Login response parsing extracts `auth.client_token` and `auth.lease_duration`
from the standard Vault login response. `expiresAt` is set to
`now + leaseDuration - 30 seconds`.

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

Breaking change — `token` field replaced by `tokenSource`:

```java
// BEFORE
public record VaultTransitSigningConfig(
    String address,
    String token,
    Map<String, String> keyMapping) {}

// AFTER
public record VaultTransitSigningConfig(
    String address,
    VaultTokenSource tokenSource,
    Map<String, String> keyMapping) {}
```

## Changes to VaultTransitSigningClient

Stores `VaultTokenSource tokenSource` instead of `String token`. Both
`fetchPublicKey()` and `callVaultSign()` change `.header("X-Vault-Token", token)`
to `.header("X-Vault-Token", tokenSource.token())`. Two lines changed.

## Error handling

Follows the `AgentSigner` contract: throw `RuntimeException` for transient failures.

- **Login fails:** `token()` throws → `sign()` throws → entry saved unsigned.
  Nothing cached on failure — next call retries.
- **Token expired between check and use:** Vault returns 403 on the sign call →
  signing client throws → entry saved unsigned. The context cache (public keys)
  is unaffected. Next sign call gets a fresh token via `token()`.
- **No 403-retry in the signing client.** The signing client doesn't own token
  lifecycle — the token source does. Adding retry creates a coupling between the
  client and the source (the client would need to know the source can produce a
  different token). Clean layering: call `tokenSource.token()`, use it, throw if
  Vault rejects it.

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

`VaultTransitAgentSigner` constructor switches on `auth().method()` to create the
appropriate `VaultTokenSource`, validating that required fields are present at
startup.

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
- `VaultTransitSigningClientTest` — updated for `StaticVaultTokenSource` in config

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
| `VaultTransitSigningConfig.java` | BREAKING — `token` → `tokenSource` |
| `VaultTransitSigningClient.java` | MODIFY — use `tokenSource.token()` |
| `VaultTransitContext.java` | unchanged |
| Tests: 4 new, 1 modified | see Testing section |

### Quarkus adapter (`signing/vault-transit-quarkus/`)

| File | Change |
|------|--------|
| `VaultTransitConfig.java` | BREAKING — drop `token()`, add `auth()` |
| `VaultTransitAgentSigner.java` | MODIFY — create token source from config |
| `VaultTransitAgentSignerIT.java` | MODIFY — update config structure |

**Total: 8 new files, 5 modified. Zero new modules. Zero new dependencies.**

## Platform coherence

- Module tier structure: ✅ pure Java SPI + Quarkus CDI adapter
- No heavy SDK types in SPI: ✅ `VaultTokenSource.token()` returns `String`
- Config prefix: ✅ `casehub.ledger.vault-transit.auth.*`
- Error handling: ✅ follows `AgentSigner` contract
- No impact on other signing modules (AWS/GCP/Azure use SDK credential chains)
- No schema, entity, or repository changes
