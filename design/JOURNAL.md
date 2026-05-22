# Design Journal — issue-94-parity-enforcement

### 2026-05-22 · §Architecture · §Key Design Decisions · §Implementation Tracker

The blocking/reactive tier separation (protocol PP-20260519-39a9a5) was previously
enforced by convention only: the protocol document stated the parity rule, but nothing
in the build verified it. This commit makes enforcement structural via an ArchUnit
bytecode scan in `BlockingReactiveParityTest`. The test auto-discovers all
`Reactive*Service` classes by naming convention — no annotation required on production
code — and enforces bidirectional parity: every public non-static method `foo()` in the
blocking tier must have `fooAsync()` in the reactive tier (and vice versa), with matching
raw parameter types. A minimum-count guard prevents vacuous green on package rename.
ArchUnit was chosen over plain reflection (no generalisation, no overload safety) and
`@BuildStep` (fires at consumer deploy time, not extension test time).

**§Implementation Tracker row to add** (after "Reactive service tier separation"):
`| **Blocking/reactive parity enforcement** | ✅ Done | `BlockingReactiveParityTest` (ArchUnit 1.4.1 bytecode scan); auto-discovers all `Reactive*Service` classes by naming convention; bidirectional method parity with parameter-type checking; minimum-count guard against vacuous pass on package rename; `Uni<T>` return type enforcement. 449 tests. Closes #94. |`
