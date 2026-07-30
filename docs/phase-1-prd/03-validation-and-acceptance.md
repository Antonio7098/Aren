# Phase 1 PRD — Validation and Acceptance

## 25. Diagnostic CLI Requirements

Phase 1 must provide a development-only CLI for inspecting lifecycle semantics.

Required scenarios:

```text
aren dev run success
aren dev run fail
aren dev run cancel
aren dev run race
```

Recommended additional scenarios:

```text
aren dev run ignore-cancel
aren dev run parent-cancel
aren dev run panic
```

The CLI should display events in sequence order:

```text
000 run.created                 created → created
001 run.started                 created → running
002 run.cancellation_requested running → running
003 run.cancelled               running → cancelled
```

The CLI must:

- exercise the real runtime implementation;
- display run identity, sequence, event type, and outcome;
- distinguish occurrences from state transitions;
- return a useful process exit code;
- make cancellation acceptance and terminal cancellation visibly distinct;
- offer a repeatedly executable race scenario;
- avoid implying persistence or exactly-once delivery.

The CLI is a semantic inspection and smoke-testing tool, not the final Aren CLI product.

## 26. Functional Requirements

### FR-1: Create a run

A caller can start one supervised work function and receive a run handle.

### FR-2: Assign identity

Aren assigns a unique opaque identity before recording the first event.

### FR-3: Record canonical creation

Aren records `run.created` as sequence `0`.

### FR-4: Own transitions

Only Aren-controlled transition code can mutate lifecycle state.

### FR-5: Invoke work

Aren invokes work with a child context derived from the supplied parent.

### FR-6: Preserve successful result

When resolution selects success, Aren returns the work result unchanged in a succeeded outcome.

### FR-7: Preserve work failure

When resolution selects failure from a returned error, Aren preserves the original cause.

### FR-8: Recognise panic

When work panics, Aren records a coherent failed outcome with panic origin preserved.

### FR-9: Accept cancellation

A caller or parent context can request cancellation through one shared acceptance path.

### FR-10: Report cancellation disposition

Cancellation reports `accepted`, `already_requested`, or `already_terminal`.

### FR-11: Make cancellation idempotent

Repeated requests do not corrupt metadata or duplicate events.

### FR-12: Resolve terminal outcome centrally

All completion paths use one deterministic terminal-resolution function.

### FR-13: Commit transitions atomically

State, event, timing, outcome, and waiter release remain coherent.

### FR-14: Retain lifecycle history

Complete lifecycle-event history remains available in memory for the run object’s lifetime.

### FR-15: Support replay

Observers can read events from a requested sequence.

### FR-16: Protect execution from observers

Slow, absent, or abandoned observers do not block the lifecycle.

### FR-17: Wait repeatedly

Multiple callers can retrieve the same immutable outcome.

### FR-18: Reject illegal transitions

Illegal lifecycle transitions are detected as internal invariant violations.

### FR-19: Inspect through CLI

Required lifecycle scenarios can be exercised through the diagnostic CLI.

## 27. Non-Functional Requirements

### Correctness

Correct lifecycle behaviour is more important than API convenience, throughput, or abstraction elegance.

### Determinism

Given the same committed facts, all callers agree on state, event history, cancellation metadata, and outcome. Scheduler behaviour may affect when cancellation is accepted, but not interpretation of an identical fact set.

### Concurrency safety

The implementation passes Go race detection for the required test suite and repeated stress scenarios.

### Leak resistance

A completed or cancelled run must not retain active Aren-owned goroutines solely because nobody reads events, an observer stops, multiple callers wait, or cancellation is repeated.

Work that deliberately never returns is not an Aren lifecycle leak, provided Aren does not create additional leaked goroutines around it.

### Simplicity

Prefer concrete types, explicit control flow, one central transition function, one terminal-resolution function, in-memory state, focused synchronization, and direct tests.

### Diagnosability

Failures and invariant violations identify the run, failure origin, failure kind, terminal state, and violated rule where applicable.

### API stability

No stable public API or semantic-versioning promise is required during foundational phases. Behavioural contracts take priority over initial names and signatures.

## 28. Required Test Matrix

### Normal lifecycle

Verify `created → running → succeeded`, sequence `0` creation, result preservation, one success event, deterministic order, coherent timing, and immediate post-completion waiting.

### Failure lifecycle

Verify `created → running → failed`, inspectable original error, `origin = work`, `kind = returned_error`, one failure event, and no exposed successful result.

### Panic lifecycle

Verify a panic does not leave the run active, produces `failed` with work/panic classification, retains diagnostics, records one terminal event, and releases all waiters.

### Explicit cancellation

Verify first request reports `accepted`, context cancellation reaches work, exactly one request event is recorded, terminal cancellation waits for work return, and first reason is retained.

### Repeated cancellation

Verify later active requests report `already_requested`, post-terminal requests report `already_terminal`, concurrent requests are safe, no duplicate request or terminal events occur, and the reason remains stable.

### Parent-context cancellation

Verify parent cancellation uses the normal acceptance path, does not duplicate an explicit concurrent request, and terminal resolution is source-independent.

### Already-cancelled parent

Verify the normal created/running history is recorded, work receives a cancelled context, no direct created-to-cancelled transition occurs, and normal resolution applies.

### Delayed cancellation acknowledgement

Verify state remains `running`, cancellation metadata is visible, waiters remain blocked, and resolution occurs only after return.

### Ignored cancellation

Verify Aren does not falsely report terminal cancellation, event observation remains usable, no Aren-owned lifecycle goroutine leaks, and diagnostics communicate the limitation.

### Success after cancellation acceptance

Verify accepted cancellation plus a nil-error successful return resolves to `succeeded`, preserves request history, and cannot be overwritten.

### Cancellation error after accepted cancellation

Verify an error matching the run-context cancellation cause resolves to `cancelled` with one cancellation terminal event.

### Unrelated error after accepted cancellation

Verify an unrelated error resolves to `failed` and accepted cancellation does not automatically override it.

### Completion–cancellation race

Run many iterations under the race detector. Every iteration must have exactly one terminal state, one terminal event, one outcome, no race or deadlock, coherent contents, deterministic fact interpretation, and no terminal overwrite.

### Multiple waiters

Verify all waiters unblock, receive the same logical outcome, do not consume it, and post-completion waits return immediately.

### Event replay

Observe before start, after start, after cancellation request, concurrently with terminal commitment, after completion, from sequence `0`, and from a later valid sequence. Every observer reconstructs the correct canonical suffix.

### Slow and abandoned observers

Test no observer, fast observer, slow observer, abandoned observer, and multiple simultaneous observers. None may block execution or another observer.

### Transition atomicity

Concurrent readers must never observe terminal state without event, event without outcome, outcome with nonterminal state, sequence without event, or partial timing.

### Mutable payload isolation

Verify lifecycle payloads cannot be mutated through observer access and one observer cannot change another’s view or retained history. Document the shallow-immutability contract for arbitrary result values.

### Invariant enforcement

Exercise illegal transitions through a test-only boundary. Verify detection, no duplicate outcome, Aren-origin classification, distinction from work failure, and retained diagnostic context.

### Timing

Verify start time exists, terminal time appears only after completion, terminal is not before start, all waiters see identical values, and ordering does not depend on timestamps.

### Concurrency stress

Concurrently run state readers, waiters, observers, explicit cancellation, parent cancellation, and terminal completion under repeated `go test -race` execution.

## 29. Acceptance Criteria

Phase 1 is accepted only when all of the following are true.

### Lifecycle definition

- Every state, legal transition, and illegal transition is defined.
- The atomic transition boundary and ownership are explicit.
- `created` is historical but not externally actionable.
- Outcome coherence rules are documented.

### Terminal resolution

- All candidates pass through one central function.
- Success after cancellation, cancellation-related errors, unrelated errors, and panic are defined.
- Scheduler timing cannot arbitrarily reinterpret identical facts.

### Cancellation

- Requested, accepted, observed, and terminal cancellation are distinct.
- Explicit and parent cancellation use one path.
- Cancellation is idempotent and disposition is observable.
- Delayed, ignored, and post-terminal cancellation follow the contract.

### Events

- Canonical per-run history and `(run_id, sequence)` identity are defined.
- Order is deterministic.
- Exactly one terminal event is recorded.
- Recording and delivery are distinguished.
- Replay, multiple observers, non-blocking observation, and post-completion observation are supported.

### Atomicity

- State, event, timing, outcome, and waiter release commit coherently.
- Concurrent readers cannot observe partial transitions.
- Work execution is explicitly outside lifecycle atomicity.

### Concurrency

- Multiple waiters receive the same outcome.
- Required tests pass under the race detector.
- Stress testing finds no race, deadlock, duplicate terminal outcome, or observer-caused Aren goroutine leak.

### Failures

- Returned errors preserve causes.
- Panics remain recognisable.
- Aren failures are distinguishable from work failures.
- Impossible outcome combinations are prevented.

### Guarantee boundaries

- At-most-one terminal lifecycle commitment is documented.
- Exactly-once work execution is explicitly not promised.
- Work effects are outside lifecycle atomicity.

### Runnable evidence

- CLI implements success, failure, cancellation, and race scenarios.
- Panic, ignored cancellation, and parent cancellation are demonstrated or equivalently tested.
- At least one smoke exercise is performed outside unit tests.
- The phase review records awkward APIs, missing events, redundant behaviour, and simplification opportunities.

### Documentation

- The PRD is current.
- A normative lifecycle contract exists.
- Important semantic decisions are recorded.
- Deferred ideas remain in the scope ledger.

## 30. Exit Gate

Phase 1 is complete only when:

1. all states and transitions are defined;
2. lifecycle transition is established as Aren’s atomic unit;
3. Aren alone owns transition enforcement;
4. state, event, timing, outcome, and waiter release commit coherently;
5. exactly one terminal outcome is guaranteed;
6. terminal resolution is centralized and deterministic;
7. cancellation races are proven under the race detector;
8. requested, accepted, observed, and terminal cancellation are distinct;
9. explicit and parent cancellation share one path;
10. event order is deterministic;
11. exactly one terminal event is recorded;
12. retained per-run replay is implemented;
13. slow or abandoned observers cannot deadlock or leak a run;
14. multiple waiters observe one immutable outcome;
15. work errors, panics, and Aren failures remain distinguishable;
16. lifecycle guarantees are separated from work-side-effect guarantees;
17. runnable diagnostic evidence exists;
18. the lifecycle contract contains no unresolved foundational ambiguity.

Open questions affecting these conditions block completion. Naming, package layout, generic syntax, and later execution types do not.

## 31. Success Measures

Phase 1 success is demonstrated through correctness evidence rather than adoption or performance metrics.

Required measures:

- all lifecycle contract tests pass;
- all tests pass under `go test -race`;
- repeated terminal-resolution races produce zero invariant violations;
- every completed history has exactly one terminal event;
- observer tests cause no lifecycle deadlock;
- multiple waiters never receive inconsistent outcomes;
- transition-atomicity tests never observe partial commits;
- cancellation disposition remains stable under stress;
- the CLI exercises all required scenarios;
- the post-phase review finds no unresolved semantic issue that undermines Phase 2.

Performance benchmarking is not a Phase 1 success criterion unless overhead distorts behaviour or tests.
