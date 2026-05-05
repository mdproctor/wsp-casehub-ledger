---
layout: post
title: "traceId, Entity Listeners, and a Gap I Shouldn't Have Left"
date: 2026-04-22
type: phase-update
entry_type: note
subtype: diary
projects: [casehub-ledger]
tags: [quarkus, opentelemetry, jpa, cdi]
---

`correlationId` was documented from day one as "OpenTelemetry trace ID — W3C format, 32-char hex string." The name was just wrong.

`correlationId` is an established term in messaging — JMS, AMQP, Kafka all use it for request/reply matching. That's different from a distributed trace ID. If a consumer is doing event-driven workflows and wants to store a message correlation ID, the field name would actively mislead them. `causedByEntryId` already handles causal chain traversal via a direct FK, so there's no gap to fill — just the name to fix.

We renamed it to `traceId` across the entity, migration, index, archiver, PROV serializer, and tests. The migration is rewritten in place — no deployed instances, no backwards compatibility needed.

## Auto-wiring the trace ID at persist time

With the field correctly named, I brought Claude in to wire up automatic population from the active OTel span. The approach: a `@ApplicationScoped` CDI bean registered as a JPA entity listener.

```java
@ApplicationScoped
public class LedgerTraceListener {

    @Inject
    LedgerTraceIdProvider traceIdProvider;

    @PrePersist
    public void prePersist(Object entity) {
        if (!(entity instanceof LedgerEntry entry)) return;
        if (entry.traceId != null) return;
        traceIdProvider.currentTraceId().ifPresent(id -> entry.traceId = id);
    }
}
```

In standard JPA, entity listeners are instantiated by the JPA provider with no injection support. Quarkus's Hibernate ORM extension silently wires CDI into the listener lifecycle — `@ApplicationScoped` on the listener class is all it takes. No `BeanManager` lookups, no boilerplate.

The `LedgerTraceIdProvider` SPI lets consumers override the trace source. The default reads `Span.current().getSpanContext()`. If no OTel SDK is configured, `Span.current()` returns the noop span — `isValid()` returns false, `traceId` stays null. Safe in every configuration.

One compile-cycle gotcha: `@DefaultBean` lives in `io.quarkus.arc`, not `jakarta.enterprise.inject`. The CDI spec doesn't have it. The compiler just says "cannot find symbol" — unhelpful.

## The integration test I skipped

The unit tests covered `OtelTraceIdProvider` in isolation: valid span, invalid span, no span. What they don't say anything about is whether Hibernate actually calls the listener, and whether CDI injection into it works. The Hibernate-CDI entity listener bridge can silently fail to wire — unit tests won't catch that.

I asked why we hadn't written a `@QuarkusTest` for the wire-up. No good reason. We added it.

`Span.wrap(SpanContext).makeCurrent()` turns out to be the right tool here. It creates an active OTel trace context on the current thread using only the OTel API — no SDK, no configured tracer, nothing beyond what's already transitively on the classpath.

```java
try (Scope ignored = Span.wrap(activeSpanContext()).makeCurrent()) {
    repo.save(entry);
}
assertThat(entry.traceId).isEqualTo(TRACE_ID);
```

Three cases: auto-populated from active span, not overwritten when caller sets it, null when no span. 175 tests, all green.
