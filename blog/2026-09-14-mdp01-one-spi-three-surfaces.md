---
title: "One SPI, Three Surfaces"
date: 2026-09-14
author: mdp
entry_type: note
subtype: diary
series: issue-295-unified-api-generation
projects:
  - casehubio/ledger
tags: [api-generation, mcp-domain, code-generation, apt]
---

# One SPI, Three Surfaces

casehub-ledger has nine REST endpoints and nine GraphQL operations. All
hand-written. All delegate to the same services. The REST DTOs and
GraphQL DTOs carry the same fields under different class names. When
we added `trustRoutingProfile` to GraphQL, we had to remember to add
the equivalent REST endpoint too. Nobody did. That's the drift that
generated code eliminates.

The platform's `graphql-generator` APT already exists — it reads
`@McpDomain` SPI interfaces at compile time and generates both JAX-RS
REST resources and SmallRye GraphQL resolvers. Four platform endpoints
were migrated in the parent epic. Ledger is the first extension to
adopt it.

## The Design

Four SPI interfaces, one per domain: entries, attestations, verification,
trust. Each uses `@PlatformQuery` or `@PlatformMutation` with platform-specific
`@PathParam` — never JAX-RS annotations on the SPI. The reason: Quarkus
REST discovers any interface annotated with `@Path` as a server resource.
Put JAX-RS `@Path` on an SPI interface and Quarkus registers a no-op
implementation alongside the generated resource. The symptom is wrong
HTTP status codes on endpoints you didn't touch — zero indication that
an interface is involved.

The SPI interfaces live in `api/`. Four `@DefaultBean` service implementations
in `runtime/` delegate to existing repositories and services. The `rest/` and
`graphql/` modules each configure the APT independently — `rest/` generates
REST only, `graphql/` generates GraphQL only. Consumers choose their API
surface by picking their dependency.

## Three Layers of Wrong

The implementation hit three obstacles, each masking the next.

**Layer 1: The isolated classpath.** Maven's `annotationProcessorPaths`
creates a completely separate classpath from the project's compile
dependencies. The SPI interface jar — sitting right there as a compile
dependency with a Jandex index — was invisible to the generator. The APT
scanned three Jandex indexes and found eight domains, but none of them
were ours. Fix: list the API jar explicitly in `annotationProcessorPaths`.

**Layer 2: The invisible buffer.** After fixing the classpath, the generator
tried to create class files with `/` in the name. The domain `ledger/entries`
became class name `GeneratedLedger/entriesResource` — Java interpreted the
`/` as a module boundary. We patched `toPascalCase` to normalize `/` to `-`
before case conversion. The patch applied cleanly via IntelliJ MCP's
`ide_replace_text_in_file`. The tool confirmed "1 replacement made." We
rebuilt. Same error.

We verified the bytecode. The fix was there — `bipush 47, bipush 45,
invokevirtual String.replace(CC)`. We deleted the jar, reinstalled,
killed daemons, decompiled the class file. The replace instruction was
present. But the generator still produced `Ledger/entries`. For an hour.

The fix existed only in IntelliJ's in-memory Document buffer.
`ide_replace_text_in_file` modifies the IDE's virtual file system, not
the disk. `mvn compile` reads from disk. The file on disk never changed.
Every verification we ran — Read tool, bytecode inspection — went
through IntelliJ's buffer and confirmed the fix. The compiler went
through the filesystem and compiled the old source. `ide_sync_files`
flushes the buffer to disk. Nobody called it.

**Layer 3: The slot's shadow.** Even after flushing to disk and rebuilding,
the error persisted. Each slot has its own `.m2` at `slots/194/.m2`,
configured via `.mvn/slot-settings.xml`. Every `mvn install` we ran
went to `~/.m2/repository`. The slot's Maven never looked there — it
resolved from its own local repo, which still had the old jar. The
host `.m2` is just a fallback remote, not the local repository.

Three independent caching layers: IntelliJ's Document buffer, the
host `.m2`, and the slot-local `.m2`. Each one appeared correct when
inspected through its own lens. The diagnosis required checking all
three simultaneously.

## The Protocol

Hierarchical `@McpDomain` values — `"ledger/entries"` rather than
`"ledger-entries"` — matter because MCP progressive discovery uses the
`/` to build a tool hierarchy. Flatten the domain and clients see four
disconnected tools instead of a ledger group with four sub-domains. We
captured this as protocol PP-20260914-7387db: the generator handles `/`,
repos preserve hierarchical naming.

An audit of all casehub repos showed no existing code uses hierarchical
domains — every `@McpDomain` annotation is flat. Ledger is the first.
Several candidates emerged for future conversion: the notification
endpoints could group under `notifications/`, the compliance resolvers
under `qhorus/compliance`. But that's separate work.

## The Semantic Gap

We deleted 26 files and 1,201 lines. The old REST resources, GraphQL
resolvers, and all their DTOs — gone. The four SPI interfaces and their
`@DefaultBean` implementations replaced everything.

One thing fell through the cracks: the hand-written `LedgerEntryResource`
threw `LedgerNotFoundException` when an entry didn't exist — HTTP 404.
The generated resource wraps every return in `Response.ok()`. When the
service returns `null`, the client gets 200 with a JSON `null` body.
The compile succeeds, the tests adapt, but the API contract changed
silently. In GraphQL, returning `null` for a missing entry is correct.
In REST, it's wrong. The generated code doesn't know which surface it's
targeting.

The clean fix is a JAX-RS `ContainerResponseFilter` that converts
200-with-null to 404 — cross-cutting, lives alongside the exception
mapper, no service layer contamination. For now, pre-release with no
consumers, the semantic gap sits in the findings log waiting for its
issue.
