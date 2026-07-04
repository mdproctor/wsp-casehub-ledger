---
layout: post
title: "When Two Classes Do the Same Thing"
date: 2026-07-04
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [vault, signing, consolidation]
---

## When Two Classes Do the Same Thing

I wrote `JwtVaultTokenSource` for Vault's JWT auth backend — the one that lets non-Kubernetes environments (Okta, Auth0, Keycloak, Azure AD) exchange a pre-existing JWT for a Vault token. The implementation was straightforward: extend `LoginBasedVaultTokenSource`, take a `Supplier<String>` for the JWT acquisition, and let the base class handle caching, TTL, and thread safety. Same pattern as `AppRoleVaultTokenSource` — two abstract methods and nothing else.

Then the design review caught something I'd talked myself out of.

`KubernetesVaultTokenSource` and the new `JwtVaultTokenSource` have identical `loginPath()` and `loginRequestBody()` semantics. Both submit `{"role":"...","jwt":"..."}` to `/v1/auth/<mountPath>/login`. The only difference is the default mount path — `kubernetes` vs `jwt` — and where the JWT comes from. With the `Supplier<String>` constructor, Kubernetes auth is just `JwtVaultTokenSource.fromFile(addr, role, path, "kubernetes", ...)`.

I'd originally kept them separate because they map to different Vault auth backends — Kubernetes and JWT are distinct server-side plugins with different validation logic. But that's a server concern, not a client one. The client code is identical. Keeping two classes that produce the same HTTP request is concept duplication, and the consolidation makes the relationship explicit: Kubernetes auth IS a JWT exchange with a specific mount path and a file-based token.

The naming question came up early. The issue said "OIDC" — but this class does no OIDC discovery, no browser flow, no token endpoint call. It submits a pre-existing JWT. The existing convention names classes after the Vault auth method (`AppRoleVaultTokenSource` → `approle`), so `JwtVaultTokenSource` with `AuthMethod.JWT` is the right answer. Operators who mount Vault's JWT/OIDC backend at `oidc` just set `mountPath=oidc` in config — already supported.

One config change worth noting: `jwtPath` moved from `@WithDefault("/var/run/secrets/kubernetes.io/serviceaccount/token") String` to `Optional<String>`. The Kubernetes default path is a universal standard — every K8s pod has it — but baking it into a `@WithDefault` annotation means JWT auth silently tries to read the K8s service account token if you forget to configure `jwt-path`. Moving the default into the adapter's switch case makes it auth-method-specific: Kubernetes applies the default, JWT requires explicit config.

The `Supplier<String>` for JWT acquisition was the right call over a dual-constructor approach. A constructor taking both `Path jwtPath` and `String jwt` with null-check branching has two internal modes and the constructor signatures are ambiguous. The Supplier eliminates the branching — one code path, one constructor — and `fromFile()` provides the convenience factory. The Supplier is called on every login, so file-based JWTs are re-read each time, supporting rotation without any extra machinery.
