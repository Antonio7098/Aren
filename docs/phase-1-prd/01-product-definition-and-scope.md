# Phase 1 PRD — Product Definition and Scope

## 1. Summary

Phase 1 establishes the smallest execution lifecycle that Aren owns.

Aren must supervise one in-process occurrence of work from creation through exactly one terminal outcome. It must assign the run an identity, control all lifecycle transitions, propagate cancellation, retain ordered lifecycle events, publish one immutable terminal outcome, and remain correct under concurrency and cancellation races.

The atomic unit owned by Aren in Phase 1 is the **lifecycle transition**. Aren does not make the arbitrary work function transactional. It guarantees that its own state, event, timing, and terminal-outcome bookkeeping are committed coherently.

The phase deliberately excludes model providers, subprocesses, tools, persistence, retries, streaming output, workflows, pause/resume, and remote execution.

The purpose is not to design Aren’s permanent universal executor abstraction. The purpose is to establish a small set of lifecycle semantics against which later execution types can be tested.

## 2. Background

Aren’s long-term direction is to become a general-purpose runtime for supervising autonomous execution independently of any particular model provider, language, agent, transport, or deployment model.

That long-term direction is not the Phase 1 specification.

Previous projects demonstrated a recurring risk: future possibilities can become present requirements, causing abstractions and infrastructure to appear before real usage has justified them. Aren must instead begin with one concrete vertical slice and allow broader abstractions to emerge only from observed pressure.

Phase 1 therefore uses a simple in-process work function as a semantic instrument. It removes provider, transport, operating-system, persistence, and agent-loop concerns so that the execution lifecycle itself can be defined and tested in isolation.

## 3. Problem Statement

Aren cannot reliably supervise model calls, tools, agents, subprocesses, or workflows until it can answer:

- What is a run?
- What is the atomic unit of lifecycle progress?
- Who creates and owns run identity?
- Who is allowed to change lifecycle state?
- Which transitions are legal?
- What constitutes completion?
- How is a successful result distinguished from successful lifecycle completion?
- What happens when cancellation races with completion?
- When may Aren truthfully claim that work has been cancelled?
- How are returned errors caused by cancellation interpreted?
- How can lifecycle behaviour be observed without influencing execution?
- How are failures caused by work distinguished from failures inside Aren?
- How can multiple callers safely wait for the same outcome?
- Which guarantees apply to lifecycle bookkeeping, and which do not apply to work side effects?
- Which invariants must remain true under concurrency?

Without explicit answers, later capabilities would invent their own lifecycle behaviour, making cancellation, retries, persistence, event delivery, recovery, and composition inconsistent.

## 4. Phase Objective

Prove that Aren can define and enforce a coherent lifecycle for one supervised in-process execution without relying on:

- an LLM or model provider;
- a subprocess;
- a network call;
- a persistent store;
- a workflow engine;
- a daemon;
- an external event system.

A successful implementation must remain correct during:

- normal completion;
- returned work failure;
- work panic;
- explicit cancellation;
- parent-context cancellation;
- delayed cancellation acknowledgement;
- ignored cancellation;
- concurrent waiting;
- concurrent event observation;
- observer registration racing with completion;
- absent or abandoned event consumption;
- completion–cancellation races.

## 5. Primary Question

> Can Aren define and enforce the lifecycle of one execution without depending on an LLM, subprocess, network call, or persistent store?

Phase 1 is complete only when this can be answered with runnable evidence, automated tests, race-detector results, and an explicit written lifecycle contract.

## 6. Users and Stakeholders

### Primary user

The initial user is Aren’s developer, who needs to execute controlled in-process work, inspect lifecycle behaviour, observe events, request cancellation, wait for outcomes, reproduce races, and evaluate whether the semantics are suitable foundations.

### Future consumers

Later model executors, tool executors, agent loops, subprocess supervisors, workflow coordinators, persistence components, daemon-hosted execution, and external language clients may depend on this lifecycle.

These future consumers do not create Phase 1 requirements unless their needs are demonstrated by the Phase 1 vertical slice.

## 7. Definitions

### Execution

Aren-supervised work and the lifecycle surrounding it.

### Run

One concrete occurrence of an execution, with its own identity, state, event history, cancellation facts, timing, and outcome.

### Work

The in-process operation supervised by Aren. It receives a Go context, returns a result and error according to Go semantics, and may panic. It cannot directly mutate run state.

### State

The current lifecycle position of a run:

- `created`
- `running`
- `succeeded`
- `failed`
- `cancelled`

### Terminal state

A state from which no further transition is legal: `succeeded`, `failed`, or `cancelled`.

### Atomic lifecycle transition

One Aren-owned operation that validates the current state, determines the destination, commits state and metadata, constructs any terminal outcome, allocates the next event sequence, records the event, publishes the complete transition, and releases terminal waiters where applicable.

Observers must never see a partially committed transition.

### Outcome

The immutable final representation of a completed run, containing identity, terminal state, timing, and the corresponding result, failure, or cancellation information.

### Result

The successful value produced by work. It is distinct from lifecycle state and outcome.

### Failure

Structured information describing origin, kind, cause, and relevant diagnostics.

### Cancellation request

A request asking work to stop cooperatively. It is not proof that work stopped.

### Accepted cancellation request

The first cancellation request received while the run is active. Aren records its reason, cancels the work context, and records at most one cancellation-request event.

### Cancellation observation

The work function’s opportunity to observe cancellation through context.

### Terminal cancellation

The terminal outcome selected after work returns and terminal-resolution rules are applied.

### Event

An immutable record of a committed lifecycle occurrence.

### Canonical event history

The ordered in-memory sequence of events for one run. Event identity is `(run_id, sequence)`.

### Observer

A consumer that reads lifecycle events without authority to mutate the lifecycle.

### Run view

The capability to inspect identity and state, wait, retrieve the outcome, and observe events.

### Run controller

The caller capability to request cancellation.

### Transition authority

The internal Aren-only capability to mutate state, record events, publish outcomes, and release waiters.

## 8. Product Principles

### Aren owns lifecycle state and control flow

Only Aren performs transitions. Work may return, fail, panic, or observe cancellation, but does not choose the lifecycle state.

### The atomic unit is the lifecycle transition

Aren guarantees atomicity only for lifecycle bookkeeping. Work side effects may be non-transactional and cannot automatically be rolled back, deduplicated, or made exactly once.

### Exactly one terminal outcome

Every run resolves to no more than one terminal outcome. Terminal commitment is atomic from all callers’ perspectives.

### Terminal resolution is centralized

Competing paths do not independently write terminal states. All facts pass through one Aren-owned resolution function.

### Cancellation must be truthful

Aren distinguishes request, acceptance, context propagation, and terminal cancellation. It does not report `cancelled` merely because cancellation was requested.

### Events describe committed truth

An event cannot announce a state or outcome that was not committed.

### Event recording and delivery are different

Each event is recorded once in canonical history. Observer delivery may repeat through replay.

### Observation must not control execution

Slow, absent, abandoned, or faulty observers cannot deadlock the run, prevent completion, or affect another observer.

### State, result, failure, and cancellation remain separate

These are distinct concepts and must remain distinct in the design.

### Authority remains separated

Observation, caller control, and internal transition authority are separate capabilities.

### Invariant violations remain visible

Aren does not silently repair or conceal impossible internal combinations. Internal violations remain recognisable as Aren-origin failures.

### The smallest honest semantics win

Phase 1 does not introduce an event bus, executor plugins, persistence abstractions, middleware, telemetry exporters, idempotent run creation, universal serialization, pause/resume, or iteration budgets.

## 9. Scope

### In scope

- one supervised run per invocation;
- Aren-generated opaque identity;
- explicit states and transition guards;
- one in-process work function;
- one atomic transition boundary;
- context propagation;
- explicit and parent-context cancellation;
- cancellation acceptance and disposition;
- deterministic terminal resolution;
- completion–cancellation race handling;
- immutable outcomes;
- structured failures and panic recognition;
- lifecycle timing;
- retained per-run event history;
- stable event sequence identities;
- replay and multiple observers;
- multiple concurrent waiters;
- unit, race, and stress tests;
- diagnostic CLI;
- lifecycle contract;
- post-phase review.

### Explicitly out of scope

- model providers and messages;
- token streaming or structured model output;
- retries, attempts, or repair loops;
- tools or tool calls;
- subprocess execution;
- progress events or partial output;
- cleanup hooks;
- persistence, restart recovery, or replay after process termination;
- pause/resume or human approval;
- global event streams;
- workflow composition;
- multiple execution types;
- remote execution or daemon hosting;
- network APIs or multi-language SDKs;
- authentication, authorisation, budgets, or policy;
- iteration, wall-clock, or stuck-loop limits;
- production telemetry exporters;
- stable public API guarantees;
- exactly-once work execution;
- rollback or compensation of work effects;
- idempotent run creation.

Progress, partial output, cleanup semantics, and a possible executor interface belong to Phase 2.

## 10. Conceptual Lifecycle

```text
created
   ↓
running
   ├──→ succeeded
   ├──→ failed
   └──→ cancelled
```

Cancellation request is an occurrence, not a state:

```text
running
   ↓
cancellation accepted
   ↓
work context cancelled
   ↓
work eventually returns
   ↓
terminal resolution
   ├──→ succeeded
   ├──→ failed
   └──→ cancelled
```

A cancellation request does not predetermine the terminal state. Work may still return success.

## 11. State Requirements

### Legal transitions

- `created → running`
- `running → succeeded`
- `running → failed`
- `running → cancelled`

### Illegal transitions

All other transitions are illegal, including direct `created` to terminal transitions, terminal-to-terminal transitions, repeated terminal commitment, and mutation outside Aren’s transition authority.

### Transition ownership

Work, observers, waiters, and controllers must not receive a mutable lifecycle-state object.

### Created-state semantics

`created` is canonical history but not an externally actionable scheduling state.

- identity exists before `run.created`;
- `run.created` is the first event;
- `created → running` uses the normal transition mechanism;
- the run may already be running or terminal when its handle is returned;
- Phase 1 does not support caller-controlled cancellation before start.

## 12. Run Identity Requirements

Every run has one opaque Aren-generated identifier that exists before the first event, appears in every event and outcome, remains stable, contains no required semantics, does not encode execution type, and is safe for diagnostics.

Phase 1 does not require caller-supplied IDs, idempotency keys, external correlation IDs, or run-creation deduplication.

## 13. Authority Requirements

### Observation authority

May inspect identity and state, wait, inspect the outcome, read history, and subscribe to future events. It cannot mutate lifecycle state.

### Caller control authority

May request cancellation. Phase 1 exposes no other caller command.

### Internal transition authority

Only Aren may validate and commit transitions, allocate sequences, append events, commit outcomes, and release waiters.

### API-shape neutrality

The PRD defines capability boundaries, not the exact Go interface layout. One handle, separate interfaces, or another minimal arrangement are acceptable if authority remains separated.
