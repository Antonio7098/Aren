# Phase 1 PRD — Lifecycle Requirements

## 14. Work Invocation Requirements

Phase 1 work must be represented by a concrete in-process function equivalent in capability to:

```go
func(context.Context) (Result, error)
```

The exact names, generic strategy, and concrete types are technical-design decisions.

Aren must:

1. generate the run identity;
2. create the run in `created`;
3. record `run.created`;
4. transition the run to `running`;
5. record `run.started`;
6. establish a child context derived from the supplied parent context;
7. invoke work with that context;
8. capture a returned result, returned error, or panic;
9. collect the facts needed for terminal resolution;
10. resolve the terminal candidate through one centralized policy;
11. atomically commit exactly one terminal transition;
12. record exactly one terminal event;
13. publish one immutable outcome;
14. release all waiters after the complete terminal transition is visible.

The Phase 1 work-function shape is a semantic instrument, not a permanent universal executor interface.

## 15. Transition Commit Requirements

Every lifecycle transition must occur through one Aren-owned critical section.

Within the transition operation, Aren must:

1. read the committed source state;
2. validate that the requested transition is legal;
3. determine the destination state;
4. construct transition-specific metadata;
5. construct a terminal outcome where applicable;
6. record transition timing;
7. allocate the next per-run sequence number;
8. append the immutable lifecycle event;
9. make the new state, event, metadata, and outcome visible together;
10. release terminal waiters only after publication is complete.

No supported concurrent operation may observe:

- terminal state without the terminal event;
- terminal event without the terminal outcome;
- terminal outcome while state is nonterminal;
- a new sequence number without its event;
- two terminal outcomes;
- two terminal events;
- an event whose transition disagrees with committed state.

Work execution itself is outside this atomic boundary.

## 16. Outcome Requirements

### General requirements

A terminal outcome must be:

- immutable after publication;
- coherent with the terminal state;
- associated with the correct run identity;
- available to multiple waiters;
- returned consistently to all waiters;
- safe to inspect after completion;
- accompanied by coherent lifecycle timing.

### Successful outcome

A successful outcome contains terminal state `succeeded`, the exact result returned by work, start time, and finish time. It does not contain failure or terminal-cancellation information.

### Failed outcome

A failed outcome contains terminal state `failed`, structured failure information, start time, and finish time. It does not expose a successful result.

### Cancelled outcome

A cancelled outcome contains terminal state `cancelled`, cancellation metadata, start time, and finish time. The first accepted reason is retained and repeated requests cannot mutate it.

### Work returning result and error

If work returns both a non-empty result and an error:

- Aren treats the invocation as failed unless the error is classified as cancellation under terminal resolution;
- the result is not exposed as successful;
- Phase 1 does not define partial-result semantics.

### Invalid combinations

The runtime must prevent:

- `succeeded` with a failure;
- `failed` with a successful result;
- `cancelled` with a successful result;
- a nonterminal state with a terminal outcome;
- a terminal state without terminal timing;
- multiple terminal timestamps;
- an outcome whose run identity differs from the run;
- outcome timing that differs across waiters.

## 17. Failure Requirements

### Failure structure

A failure must distinguish at least:

```text
origin:
  work
  aren

kind:
  returned_error
  panic
  invariant_violation
```

The exact Go representation is a technical-design decision.

### Work-returned errors

When work returns an error:

- the original cause remains inspectable where practical;
- `origin = work`;
- `kind = returned_error`, unless terminal resolution classifies it as cancellation;
- the human-readable message does not erase the cause.

Phase 1 must not introduce provider-specific or tool-specific categories.

### Work panic

When supervised work panics:

- Aren recovers at the work-execution boundary;
- the run does not remain permanently running;
- terminal state is `failed`;
- `origin = work`;
- `kind = panic`;
- sufficient diagnostics for development investigation are retained;
- the panic is not disguised as an ordinary returned error.

Stack information may remain diagnostic/internal and is not required to be a stable public API field.

### Internal invariant violation

An internal lifecycle invariant violation must:

- be machine-distinguishable from work failure;
- use `origin = aren`;
- use `kind = invariant_violation`;
- identify the violated rule;
- never result in two terminal outcomes;
- not be silently converted into an ordinary work failure.

The technical design may panic internally for impossible states during development, but the externally observed run must remain coherent wherever recovery is possible.

## 18. Cancellation Requirements

### Cancellation sources

Cancellation may originate from:

- an explicit run-controller request;
- cancellation of the supplied parent context.

Both sources pass through the same Aren-owned cancellation-acceptance path.

### Cancellation operation result

A cancellation request returns or exposes one of:

- `accepted`;
- `already_requested`;
- `already_terminal`.

The exact type is a technical-design decision.

### Cancellation acceptance

The first request received while the run is active must:

- be accepted exactly once;
- retain the first reason;
- cancel the work context;
- record at most one `run.cancellation_requested` event.

### Repeated cancellation

Repeated requests must:

- be safe and not panic;
- not replace the accepted reason;
- not record duplicate request events;
- not create duplicate terminal outcomes;
- report `already_requested` while active;
- report `already_terminal` after completion.

### Cooperative cancellation

Aren propagates cancellation through Go context. Phase 1 does not forcibly terminate goroutines.

### Request versus terminal cancellation

Requesting or accepting cancellation does not directly force a terminal state. The run remains `running` until work returns and terminal resolution commits an outcome.

### Delayed acknowledgement

If work returns only after a delay:

- the run remains `running`;
- cancellation metadata is observable;
- waiters continue waiting;
- observers may see `run.cancellation_requested`;
- terminal resolution occurs only after work returns.

### Ignored cancellation

If work ignores context cancellation:

- Aren does not claim work has stopped;
- Aren does not commit terminal cancellation while work continues;
- the run remains active until work returns;
- Phase 1 has no force-kill mechanism;
- event non-consumption must not leak an Aren-owned lifecycle goroutine.

### Cancellation after termination

A post-terminal request must:

- be safe;
- return `already_terminal`;
- leave the outcome and cancellation metadata unchanged;
- record neither a request event nor another terminal event.

### Already-cancelled parent context

When started with an already-cancelled parent:

- Aren records the normal `created → running` lifecycle;
- work receives an already-cancelled child context;
- terminal resolution uses the normal rules;
- no `created → cancelled` shortcut is introduced.

## 19. Terminal Resolution Requirements

### Central resolution function

All returned work facts pass through one terminal-resolution function. It considers:

- whether a terminal outcome is already committed;
- whether cancellation was accepted;
- the cancellation cause;
- whether work returned a result;
- whether work returned an error;
- whether the error matches the run-context cancellation cause;
- whether work panicked.

### Phase 1 resolution policy

#### Successful return

A result with nil error produces:

```text
terminal state = succeeded
```

This remains true even if cancellation was accepted before return. A successful return demonstrates that work completed successfully according to its contract.

#### Cancellation-related return

If cancellation was accepted and work returns an error matching the run-context cancellation cause:

```text
terminal state = cancelled
```

#### Unrelated returned error

Any other returned error produces:

```text
terminal state = failed
failure.origin = work
failure.kind = returned_error
```

#### Panic

A panic produces:

```text
terminal state = failed
failure.origin = work
failure.kind = panic
```

#### Existing terminal outcome

If an outcome is already committed, it remains unchanged and later transition attempts are rejected as invalid or redundant according to the internal contract.

### Atomic commitment

After selecting a candidate, Aren commits it through the atomic transition boundary. A committed terminal outcome can never be replaced.

### Scheduler independence

Scheduler timing may determine whether cancellation is accepted before work completes. Given the same committed facts, terminal interpretation must not depend on which goroutine attempts a write first.

## 20. Event Requirements

### Lifecycle vocabulary

Phase 1 supports events equivalent to:

- `run.created`;
- `run.started`;
- `run.cancellation_requested`;
- `run.succeeded`;
- `run.failed`;
- `run.cancelled`.

No progress, token, output-delta, retry, attempt, tool, cleanup, workflow, pause, or persistence events are included.

### Event contents

Every event contains:

- run identity;
- event type;
- per-run sequence number;
- occurrence time;
- source state where applicable;
- destination state where applicable;
- immutable event-specific data where required.

### Sequence identity

Stable event identity is:

```text
(run_id, sequence)
```

Requirements:

- sequence numbers increase monotonically;
- they are unique within a run;
- `run.created` uses sequence `0`;
- later events increment by one;
- order is determined by sequence, not timestamp;
- no separate transition UUID is required.

### Non-transition events

`run.cancellation_requested` is an occurrence rather than a state transition. Its source and destination are both `running`, with accepted-request metadata.

### Canonical recording

Each lifecycle occurrence is recorded once in canonical history. Exactly one terminal event may be recorded. Aren does not claim exactly-once observer delivery.

### Event immutability

Once recorded, an event cannot change. Observers cannot mutate retained history, another observer’s event view, or transition metadata. Lifecycle payloads should use Aren-owned immutable values wherever practical.

### State-event consistency

Consumers must never observe a success event for a failed run, terminal state without its terminal event, terminal event without outcome, two terminal events, or sequence numbers inconsistent with committed order.

## 21. Event Observation Model

### Selected model

Phase 1 uses retained, in-memory, per-run lifecycle-event history. It does not use a global event bus.

### Observer support

Multiple observers are supported and each has an independent cursor.

### Replay

An observer may begin from a requested sequence. Default observation starts at `0`, replaying existing events, delivering future events, and completing after the terminal event.

### Observation after completion

An observer created after completion can immediately read the complete canonical history.

### Slow and abandoned observers

A slow or abandoned observer must not block work, cancellation, transitions, terminal commitment, waiter release, or another observer, and must not leak an Aren-owned producer goroutine indefinitely.

### Event retention

Events remain available while the run object is reachable in memory. Phase 1 makes no guarantee after process termination, run-object collection, or restart.

### Stream completion

Live observation completes only after the terminal transition is fully committed and the terminal event is visible.

### Delivery semantics

Replay or repeated subscription may expose the same canonical event more than once. Consumers can deduplicate using `(run_id, sequence)`.

### Implementation neutrality

The technical design may use snapshots plus notifications, cursors over retained slices, or another safe in-memory mechanism. A raw single-consumer channel is insufficient unless it satisfies replay, multi-observer, and non-blocking requirements.

## 22. Waiting and Concurrency Requirements

### Waiting

Waiting blocks while active, returns only after the complete terminal transition is visible, returns immediately after completion, provides the same logical outcome to every waiter, and does not consume the outcome exclusively.

### Multiple waiters

Concurrent waiters all observe the same terminal state, result or failure, cancellation metadata, timing, and complete outcome.

### Single-writer lifecycle mutation

Lifecycle mutation behaves as a single-writer system. Concurrent readers are supported. Concurrent transition attempts are serialized through Aren’s transition boundary.

### Supported concurrent operations

The implementation remains race-free under concurrent waiting, explicit cancellation, parent cancellation, state inspection, event subscription, event replay, terminal completion, and post-completion outcome inspection.

### Mutable result ownership

Aren does not deep-copy arbitrary result values. The outcome container is immutable, but referenced objects remain governed by their own type and caller ownership.

## 23. Timing Requirements

The runtime records start time and terminal time. Duration may be derived.

- start time is not after terminal time;
- terminal time exists only after terminal commitment;
- all observers see the same times;
- event timestamps are coherent;
- equality is allowed;
- ordering relies on sequence, not timestamps.

A public clock abstraction is introduced only if deterministic testing cannot be achieved cleanly without one.

## 24. Work-Side-Effect Guarantees

### Lifecycle guarantee

Aren guarantees one canonical history per run, at most one terminal lifecycle commitment, one terminal outcome, and coherent state/event/timing/outcome publication.

### Work guarantee

Aren does not guarantee:

- exactly-once work execution;
- transactional work effects;
- rollback or compensation;
- deduplication across separate runs;
- interruption of a goroutine that ignores context;
- that failed or cancelled work produced no external effects.

### Documentation requirement

This distinction must be explicit in the lifecycle contract, public Phase 1 documentation, cancellation examples, and future retry discussions.
