# Phase 1 PRD — Risks, Decisions, and Handoff

## 32. Risks and Mitigations

### Risk: Designing the final public API too early

The first work function and run-handle design may accidentally be treated as permanent.

**Mitigation:** Specify capability boundaries and behaviour before names. Make no compatibility promise. Review and simplify after the CLI and smoke exercise.

### Risk: Confusing lifecycle atomicity with work atomicity

Aren may appear to promise transactional execution merely because state and events commit atomically.

**Mitigation:** State explicitly that arbitrary work effects are non-transactional and outside Aren’s rollback guarantees.

### Risk: Turning event observation into an event bus

Replay, cursors, and multiple observers could expand into infrastructure design.

**Mitigation:** Retain only a small per-run lifecycle history in memory. Exclude global streams, persistence, external delivery, filtering, routing, and subscriptions across runs.

### Risk: Using misleading exactly-once language

“Emitted exactly once” may be interpreted as a delivery guarantee.

**Mitigation:** Guarantee one canonical event record. Permit replay. Use `(run_id, sequence)` for deduplication.

### Risk: Treating cancellation request as immediate cancellation

This would make Aren claim work has stopped when it has only been asked to stop.

**Mitigation:** Separate request, acceptance, context propagation, and terminal resolution.

### Risk: Letting scheduler timing choose semantic meaning

Independent success, failure, and cancellation writers could produce arbitrary terminal outcomes.

**Mitigation:** Use one terminal-resolution function and one atomic transition commit.

### Risk: Goroutine leaks hidden by happy-path tests

Channel-based observers and waiting mechanisms can leak when callers disappear.

**Mitigation:** Test absent, slow, and abandoned observers; multiple waiters; ignored cancellation; and repeated subscription.

### Risk: Overengineering deterministic time

A clock interface may be introduced before it is useful.

**Mitigation:** Test ordering through sequence numbers and test only timing invariants. Add a clock boundary only if direct testing proves inadequate.

### Risk: Hiding panics as ordinary failures

Recovering a panic into a generic error could erase diagnostic meaning.

**Mitigation:** Preserve panic origin and kind in structured failure data.

### Risk: Mutable payloads undermine event immutability

Observers may mutate referenced data after delivery.

**Mitigation:** Keep lifecycle event payloads Aren-owned and immutable. Document that arbitrary result values are not deep-copied.

### Risk: Phase 2 concepts leak backward

Progress, cleanup, partial output, executor interfaces, and timeouts may appear attractive during implementation.

**Mitigation:** Record them in the scope ledger. Introduce them only when Phase 2 deliberately attacks the lifecycle with richer controlled behaviour.

## 33. Product Decisions Resolved by This PRD

### Decision 1: Atomic unit

The Phase 1 atomic unit is one Aren-owned lifecycle transition.

### Decision 2: Event observation

Phase 1 retains complete per-run lifecycle history in memory and supports multiple replayable observers with independent sequence cursors.

### Decision 3: Cancellation status

Cancellation request is represented as metadata and a lifecycle event, not as a separate lifecycle state.

### Decision 4: Cancellation disposition

Cancellation reports:

- `accepted`;
- `already_requested`;
- `already_terminal`.

### Decision 5: Created-state observability

`created` is canonical history but not an externally actionable scheduling state.

### Decision 6: Terminal race policy

Aren resolves terminal candidates through one deterministic function and commits the selected candidate atomically.

### Decision 7: Result plus error

A work return containing an error is not successful. The result is not exposed as a successful result.

### Decision 8: Panic representation

Panic produces:

```text
terminal state = failed
origin = work
kind = panic
```

### Decision 9: Internal invariant representation

Internal invariant failure uses:

```text
origin = aren
kind = invariant_violation
```

### Decision 10: Event delivery guarantee

Each event is recorded once in canonical history. Observer delivery may repeat through replay.

### Decision 11: Parent-context cancellation

Parent-context cancellation passes through the same acceptance and terminal-resolution path as explicit cancellation.

## 34. Remaining Technical Design Decisions

The following are implementation decisions rather than unresolved product semantics:

- exact Go run-handle shape;
- whether view and controller are separate interfaces;
- result typing and generic strategy;
- synchronization primitive;
- event cursor API;
- notification mechanism for live observers;
- exact cancellation-result type;
- exact error-wrapping structure;
- stack retention format;
- whether state reads return snapshots or copied values;
- test-only access to illegal-transition boundaries;
- whether a clock seam is necessary.

These decisions must preserve the requirements in the PRD.

## 35. Non-Goals

Phase 1 is not intended to prove:

- that one lifecycle fits every future execution type;
- that Aren has found its permanent executor interface;
- that events can be persisted or distributed;
- that arbitrary goroutines can be forcibly stopped;
- that work results are semantically valid;
- that cancellation is equivalent to process termination;
- that external clients can reconnect after restart;
- that execution is exactly once;
- that side effects can be rolled back;
- that executions can be retried or resumed;
- that multiple executions can be composed;
- that Aren is ready for production deployment.

The phase proves only that one local in-process run can be supervised coherently.

## 36. Required Phase Review

After implementation, automated testing, and diagnostic CLI use, conduct a written review covering:

- Was lifecycle transition the correct atomic unit?
- Was the terminal-resolution policy understandable in practice?
- Did success-after-cancellation behaviour feel correct?
- Were cancellation request, acceptance, and terminal cancellation sufficiently distinct?
- Did parent-context cancellation introduce ambiguity?
- Did retained event history remain simpler than channel-only delivery?
- Was supporting multiple observers worthwhile at this stage?
- Did any event prove redundant?
- Is `run.created` useful as history?
- Did the authority split make the API clearer?
- Did mutable result values create practical problems?
- Did any interface appear without real need?
- Are failure origin and kind sufficient for Phase 2?
- Did any test reveal hidden state or unclear ownership?
- Can any package, type, option, event, or cancellation field be removed?
- Does Phase 2 still represent the correct next uncertainty?
- Which lifecycle semantics should now be treated as stable?
- Which remain provisional for future execution types?

The review may remove or simplify Phase 1 capabilities before Phase 2 begins.

## 37. Deliverables

Phase 1 produces:

- this PRD;
- a normative lifecycle contract;
- the in-process lifecycle implementation;
- run identity generation;
- explicit transition enforcement;
- centralized terminal resolution;
- immutable terminal outcome model;
- structured failure representation;
- panic handling;
- cancellation acceptance and disposition;
- parent-context cancellation integration;
- retained per-run event history;
- sequence-based replay;
- multiple observer support;
- concurrent waiting support;
- comprehensive unit tests;
- race-detector and stress tests;
- diagnostic CLI scenarios;
- one real smoke exercise;
- a written phase review;
- updated glossary, scope ledger, and relevant decision records.

## 38. Deferred Questions

The following questions are explicitly deferred:

- Should Aren have a general `Executor` interface?
- How should progress be represented?
- How should cleanup affect terminal outcomes?
- Can failed executions expose partial output?
- How should retry attempts relate to runs?
- How are work side effects made idempotent during retry?
- Which event types are shared by model and tool execution?
- Should results be generic, typed, serialized, or execution-specific?
- Which state must survive process termination?
- How are durable events observed?
- How are pause and resume represented?
- How are human approvals correlated?
- How are runs composed?
- When does Aren require a daemon?
- How are executions controlled from other languages?
- Which additional execution types genuinely fit the lifecycle?
- Which lifecycle rules are universal, and which are executor-specific?

These questions may inform later experiments but must not expand Phase 1 scope.

## Handoff

The next document should be the **normative lifecycle contract**. It should convert this PRD into precise transition tables, terminal-resolution tables, cancellation rules, event invariants, and concurrency guarantees without selecting implementation details prematurely.
