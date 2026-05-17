---
layout: post
title: "Trust Without Ceremony"
date: 2026-05-13
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, cdi, trust, spi]
excerpt: "Adding a trust read-model and import SPI to casehub-ledger so actors arriving from other deployments can bring their reputation with them — without requiring configuration switches in consumer repos."
---

The trust model in casehub-ledger has always been local — an actor builds reputation within one deployment through attestations, and that reputation resets to Beta(1,1) = 0.5 wherever they land next. That's fine for a new deployment with new actors. It's frustrating when the actor has a documented track record elsewhere.

This session added three things: a read-model for trust scores, an import SPI for seeding them, and a hook in the nightly job that fires before computation for actors appearing for the first time.

The read-model is `TrustExportService` — three methods over `ActorTrustScore`. The structured export separates GLOBAL, CAPABILITY, and DIMENSION scores by design: upper layers need different shapes for each. A flat list of rows forces callers to filter and decode; three typed sub-structures let callers go straight to what they need.

The import SPI is `TrustImportService` — a single method:

```java
public interface TrustImportService {
    void importTrust(TrustExportPayload payload);
}
```

I initially sketched a `MergeStrategy` enum inside the interface (SEED_ONLY, WEIGHTED_AVERAGE, REPLACE), but that closes off every merge policy that isn't one of the three. Making the implementation the strategy — each class embodies one policy — is more extensible and matches how every other SPI in the project works. The built-in alternative does the simplest useful thing: if the actor has no GLOBAL row, seed all score types; otherwise skip.

I cut the REST endpoint and `HttpTrustBootstrapSource` entirely. The original spec included them to allow cross-deployment bootstrapping over HTTP, but there are no two deployments to connect and no one asking for it. The SPI is the value — when the topology exists, the HTTP source is a CDI bean that any consumer can drop in. Building it now means maintaining code with no callers.

Two non-obvious things surfaced during implementation.

The first was a SmallRye Config constraint. The `deploymentId` field was initially typed as `@WithDefault("") String`. Claude flagged this: SmallRye treats an empty string default on a `String` return type as "no value provided" and throws `ConfigValidationException` at startup. The fix is `Optional<String>` with no `@WithDefault` — the pattern already used throughout `LedgerConfig` by `datasource()` and the Merkle publisher URL. It compiles with the string form; it just doesn't boot.

The second was CDI inner class activation. The test doubles for the bootstrap SPI are `@Alternative @ApplicationScoped` static inner classes inside the test class — the established pattern for SPI test doubles in this codebase. Those classes are indexed by Quarkus at build time. But being indexed is not the same as being activated. Without an explicit entry in `quarkus.arc.selected-alternatives`, the `@DefaultBean` is used regardless. The test ran, the wrong bean was injected, and there were no assertions to catch it until we added the activation line. The protocol covering this pattern said Quarkus auto-discovers inner class alternatives — it doesn't. The garden entry has been corrected.

366 tests.
