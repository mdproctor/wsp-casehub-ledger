---
layout: post
title: "The Adapter That Was Always Missing"
date: 2026-07-06
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [rest, architecture, ports-and-adapters, platform]
series: issue-162-ledger-rest-module
---

The tension surfaced from a simple question: what does scaffold actually do for engine? I expected REST code generation or some framework magic. It's a standalone Quarkus app that hand-writes JAX-RS resources on top of engine's SPIs.

That's when the inconsistency became visible. Three modules, three different answers to "where does REST live." Engine has scaffold — a separate repo providing REST from the outside. Work bakes REST into its runtime module — auto-registered whether the consumer wants it or not. Ledger has no REST at all, leaving every consumer to reimplement the same query endpoints.

None of these is obviously wrong in isolation. Together they're a platform that hasn't decided what it thinks.

I worked through it from first principles. The core question: in a modular platform where libraries compose into vertical applications, where should the HTTP boundary sit? The answer fell out once I looked at the Quarkus augmentation model. JAX-RS resources in a runtime module are auto-discovered via Jandex. You can't easily suppress them. That's the wrong default — it forces REST onto consumers who only need the Java SPI.

A separate module inverts this. Include `casehub-ledger-rest` as a dependency and you get standard audit query endpoints. Don't include it and you pay zero coupling cost. Ports & Adapters applied to HTTP — the library's SPI is the port, the REST module is one possible adapter.

The pattern isn't CaseHub-specific. Any modular platform with the same composition problem benefits from the same structure. I wrote it up as a universal protocol in the garden rather than a CaseHub-only convention, then added the platform-specific migration path to PLATFORM.md.

`casehub-ledger-rest` is the first module following the pattern — four JAX-RS resources delegating to existing services, dedicated DTO records decoupled from the JPA entities, a single exception mapper. Claude tripped on `JpaLedgerEntry` being abstract during test setup — `PlainLedgerEntry` is the concrete subclass for domain-agnostic writes, which isn't obvious from the class hierarchy. The augmentation also demanded an H2 datasource even though in-memory CDI alternatives handle all persistence. Both gotchas went into the garden.

Three issues track the platform-wide migration: engine#657 extracts scaffold's REST into `casehub-engine-rest`, work#292 extracts work's runtime REST into `casehub-work-rest` (breaking change, lower priority), and ledger#162 — now closed — was the proof that the module shape works.

The interesting question is what scaffold becomes once each library owns its own REST module. Right now it's the only place engine gets HTTP endpoints. With `-rest` modules, scaffold becomes a thin composition — engine-rest + work-rest + ledger-rest + persistence + config. A reference deployment, not a bottleneck.
