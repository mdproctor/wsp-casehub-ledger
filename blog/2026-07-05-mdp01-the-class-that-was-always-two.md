# The class that was always two

`LedgerEntry` lived in two places. The api module had one — a plain abstract
class with all the fields, the supplement helpers, the `canonicalBytes()`
method. The runtime module had another — an `@Entity` with the same fields
plus JPA annotations, `@NamedQuery` declarations, entity listeners, and
agent signing infrastructure. Same name, different packages, zero
inheritance between them.

Nobody used the api version. Every consumer in the platform — engine's
`CaseLedgerEntry`, `WorkerDecisionEntry`, the five duplicate
`NoOpLedgerEntryRepository` test classes across engine submodules — they all
imported from runtime. The api copy was dead code with 201 lines and zero
references.

I'd been looking at this for a while and assumed the fix was to delete the
api copies. They were dead, after all. But when I actually mapped the
dependencies across all 26 repos in the IntelliJ workspace, something else
became clear: the api types weren't dead by design. They were dead because
nobody had finished the job. The intent — consumers programming against a
persistence-agnostic type, with JPA being one implementation — was right. The
execution had stopped halfway.

The fix was to complete the hierarchy, not delete it. Make the api
`LedgerEntry` a `@MappedSuperclass` with `@Column` annotations — the
persistence-agnostic contract. Have the runtime `JpaLedgerEntry` extend it
and add `@Entity`, `@Inheritance(JOINED)`, the named queries, the entity
listeners. One type hierarchy. One truth.

We hit a constraint in the design review that killed our first approach.
I'd originally proposed a three-tier split — plain abstract class in api,
`VerifiedLedgerEntry` in runtime (adding signing and canonical bytes), then
`JpaLedgerEntry` on top. Clean separation of data model, verification, and
persistence. The reviewer caught that JPA can't map fields from a
non-`@MappedSuperclass` ancestor — the middle tier's fields would be
invisible to Hibernate. The three-tier collapsed to two, and the signing
fields moved up to the api `@MappedSuperclass` where they belong. Any
persistence backend that maintains tamper evidence needs `canonicalBytes()`.
It's not a JPA concern — it's a data model concern.

The supplements were the hardest part. They had the same parallel-copy
problem as `LedgerEntry`, but with an additional complication: they used
JOINED inheritance. A shared `ledger_supplement` base table with two join
tables. To make the two-tier pattern work — api `@MappedSuperclass`,
runtime `@Entity` — Java's single inheritance forced a choice. A runtime
supplement can either extend its api counterpart (for `instanceof` type
safety in the `compliance()` and `provenance()` helpers) or share a common
`@Entity` root (for JOINED inheritance). Not both.

Type safety won. JOINED inheritance was eliminated. Each supplement type
became its own independent entity with a self-contained table. The schema
got simpler. The `instanceof` checks in the api helpers kept working.

We also learned something about Hibernate's bytecode enhancer that cost
time: it strips `@Transient` fields from `@MappedSuperclass` classes at
build time. Not "doesn't persist them" — physically removes them from the
enhanced bytecode. A subclass `@Entity` that tries to access the inherited
field gets `NoSuchFieldError` at runtime. Nothing in the documentation says
the field would vanish. That one went into the garden.

The end state: `LedgerEntryRepository` is now in `api/spi/`. Any consumer
can write ledger entries without depending on runtime. A new
`LedgerAppender` SPI provides value-type event recording for consumers like
blocks that don't need entity subclasses. Four dead api model duplicates are
gone, `ScoreType` extracted as a standalone enum. 209 files changed, 7
commits, and the type hierarchy that should have existed from the start
finally does.
