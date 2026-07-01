---
layout: post
title: "Follow the Provider That Got It Right"
date: 2026-07-01
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [signing, performance, azure, gcp, aws, cloud-kms]
---

Four signing adapters, four cloud KMS providers. AWS was correct from the start. GCP and Azure weren't — and the surface symptoms (eight items in the issue) all traced to one architectural mistake.

The pure Java `sign()` method in GCP and Azure didn't accept cached metadata. Each call re-fetched key information the CDI adapter had already resolved during `loadContext()`. GCP made two redundant API calls per sign — one to re-fetch the algorithm it had already discarded, another to actually sign. Azure was worse: it re-fetched the key metadata *and* constructed a fresh `CryptographyClient` with a new `DefaultAzureCredential` on every invocation. That credential scan walks environment variables, managed identity, and Azure CLI each time.

AWS didn't have this problem because its `sign()` accepted the algorithm as a parameter: `sign(keyArn, data, signingAlgorithm)`. The adapter passed it from the cached context. One API call per sign. The fix for GCP and Azure was to follow the same pattern — make `sign()` accept what the caller already knows.

The GCP fix was clean. `getPublicKey()` already returned the algorithm in its response — at line 82 of the old code, the algorithm was validated for EC and then discarded. The second API call (`getCryptoKeyVersion()`) existed solely to re-fetch what had just been thrown away. After the change, `fetchPublicKey()` returns a `GcpKmsContext` carrying both the key and the algorithm. The `getCryptoKeyVersion()` method disappears from the wrapper interface entirely.

Azure had a subtler design question. The original context record held `componentSize` (32 for P-256, 48 for P-384, 66 for P-521) — the byte length of each R and S component in the raw signature, needed for DER conversion. The spec initially proposed passing `componentSize` to `sign()`. Claude caught this in the design review: componentSize is a derived encoding detail, not a primary abstraction. The algorithm determines the component size, not the reverse. The fix passes `SignatureAlgorithm` instead — symmetric with AWS, and `componentSize` is derived inside `sign()` where DER conversion needs it.

The Azure SDK client caching was a separate fix in the same wrapper. `DefaultAzureKeyVaultClientWrapper` now holds one `DefaultAzureCredential` (created once in the constructor), one `KeyClient` per vault URL, and one `CryptographyClient` per key identifier — all in `ConcurrentHashMap` with `computeIfAbsent`. The Azure SDK clients are thread-safe and designed for reuse; the per-call construction was pure waste.

The housekeeping turned up something worth noting: the P-384 test key in `GcpKmsSigningClientTest` was fabricated — repeated byte sequences where a real EC point coordinate should be pseudorandom. It was declared but never used in any test. I replaced it with a real key and added the test that should have existed.

Across both commits: 33 files changed, 188 insertions, 604 deletions. The codebase got smaller and the API surface got consistent. All four providers now follow the same pattern: pure Java client is stateless, caching lives in the CDI adapter, `sign()` takes what the caller already knows.
