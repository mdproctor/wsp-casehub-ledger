---
layout: post
title: "The Token That Belonged to Nobody"
date: 2026-07-02
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [vault, signing, auth, concurrency]
---

## The Token That Belonged to Nobody

The Vault Transit adapter had a static token baked into its config record — `VaultTransitSigningConfig(address, token, keyMapping)`. Every HTTP call to Vault used that same string. For dev that's fine. For anything else it's a liability: the token never expires, never renews, and sits in a properties file.

The obvious fix is a `Supplier<String>` or some token provider interface. But the interesting question was where the boundary should fall between the signing client and the auth mechanism.

### Per-call, not per-client

I started with the token source inside the config record — `VaultTransitSigningConfig(address, tokenSource, keyMapping)`. A design review caught the problem: the config is a data carrier (a Java record), and a `VaultTokenSource` is behaviour — it does HTTP calls, caches tokens, tracks expiry. Putting behaviour inside a record is the wrong layering.

The fix was to push the token out of the client entirely. `fetchPublicKey` and `sign` now take a `String token` as their first parameter. The client is stateless with respect to auth — it receives a bare string and sends it as `X-Vault-Token`. The Quarkus adapter calls `tokenSource.token()` before each client method and passes the result.

This also made 403-retry clean. The adapter catches `VaultAuthenticationException`, calls `tokenSource.invalidate()`, gets a fresh token, retries once. The signing client doesn't know this happened — it just got a different string the second time.

### The cache that needed to be atomic

The token source caches a token and its expiry. Two volatile fields (`clientToken` and `expiresAt`) create a torn-read race: a concurrent `invalidate()` between reading one field and the other can leave `token()` returning null. The review caught this — the fix was an immutable `TokenState` record behind a single volatile reference:

```java
private record TokenState(String token, Instant expiresAt) {}
private volatile TokenState state = new TokenState(null, Instant.EPOCH);
```

One volatile read gives you both fields, always consistent. Double-checked locking means the common path (cached token valid, ~59 minutes out of 60) is lock-free.

### AppRole and Kubernetes share everything except the login body

Both auth methods POST to a Vault login endpoint and get back a token with a TTL. The only differences: which endpoint, and what JSON goes in the body. An abstract `LoginBasedVaultTokenSource` handles the token lifecycle — expiry tracking, lazy renewal, the clamped buffer (`min(30, leaseDuration / 2)` prevents login storms on misconfigured short TTLs). The concrete subclasses override two methods: `loginPath()` and `loginRequestBody()`.

Kubernetes has one extra wrinkle: it re-reads the JWT file on every login call. Kubernetes rotates service account tokens hourly, so the file content at login time may differ from the content at construction time. Reading lazily is the right choice — caching the JWT at startup would silently use a stale token after the first rotation.

The four signing adapters now have feature parity: AWS KMS, GCP Cloud KMS, Azure Key Vault, and Vault Transit all handle their own auth lifecycle without the signing client knowing or caring.
