1. Roadmap Purpose

Aren will be developed from a small, local execution runtime into a broader autonomous execution system.

The long-term vision is:

«A general-purpose runtime that owns and supervises autonomous execution independently of any particular agent, provider, language, transport or deployment model.»

That vision is directional rather than an initial specification.

Aren will not begin by trying to represent coding agents, model calls, shell commands, workflows, approvals and scheduled tasks through one complete universal abstraction. It will begin by proving a small execution model against controlled implementations, then broaden only when real variation provides evidence for new abstractions.

The progression is:

Execution lifecycle
    ↓
Controlled execution
    ↓
Real model invocation
    ↓
Reliable model results
    ↓
Tool execution
    ↓
Bounded agent loop
    ↓
Context management
    ↓
Durable execution
    ↓
Execution composition
    ↓
Policies and resource governance
    ↓
Daemon and remote clients
    ↓
Additional execution types

---

2. Development Rules

These rules apply to every phase.

2.1 One primary uncertainty

Each phase should answer one major question.

The phase may include several implementation tasks, but they must converge on that question.

2.2 A phase must produce something runnable

Types, interfaces and design documents are not sufficient by themselves.

Every phase must expose a working vertical slice through either:

- a Go package,
- a development CLI,
- an executable example,
- or a combination of these.

2.3 Behaviour before abstraction

Start with one concrete implementation.

Introduce a general interface only when one or more of the following is observed:

- a second implementation behaves meaningfully differently,
- conditional logic is spreading,
- testing requires controlled substitution,
- ownership needs to cross a real boundary,
- or a concept remains stable across repeated uses.

2.4 Failure testing is mandatory

Every phase must validate:

- normal operation,
- expected failures,
- cancellation where relevant,
- timing and race conditions,
- cleanup,
- and recovery or honest termination.

2.5 Real use before progression

After implementation and automated testing, the phase capability must be exercised in a small real workflow.

The phase review should ask:

- Is the API awkward?
- Are important events missing?
- Are any events redundant?
- Are failures sufficiently classified?
- Is ownership clear?
- Is hidden state affecting behaviour?
- Did an abstraction appear before it was necessary?
- Should anything be removed before continuing?

2.6 Later phases remain revisable

The early phases establish the foundation.

Later phases are intentionally less prescriptive. Their contents and order should change based on evidence from real Aren usage.

---

3. Phase Overview

Phase| Capability| Primary uncertainty
0| Project foundation| Can Aren maintain strict scope and trustworthy engineering feedback?
1| Execution lifecycle| Can Aren define and enforce a small, coherent execution lifecycle?
2| Controlled executor| Can the lifecycle survive progress, failure, cancellation and races?
3| Real model invocation| Can Aren own one real LLM request without losing lifecycle control?
4| Streaming model execution| Can streamed output remain ordered, cancellable and observable?
5| Structured model results| Can Aren distinguish valid task results from successful provider calls?
6| Retry and repair| Can Aren recover selectively without hiding failures or losing control?
7| Tool request representation| Can model-requested actions be represented independently of execution?
8| Tool execution| Can Aren supervise external actions through the same reliable principles?
9| Bounded agent loop| Can Aren own a minimal model–tool loop with explicit limits?
10| Context management| Can context become deliberate without obscuring history or behaviour?
11| Persistence and recovery| Which execution state genuinely needs to survive process termination?
12| Execution composition| How should several executions be coordinated without premature workflow machinery?
13| Policies and resources| How should budgets, permissions and concurrency be governed?
14| Daemon hosting| When and how should execution ownership move beyond one client process?
15| Multi-language clients| Can other languages control Aren without duplicating runtime behaviour?
16| Broader execution types| Which non-agent execution forms genuinely belong under Aren?
17| Distributed execution| Is there sufficient evidence for remote workers or clustering?

---

4. Phase 0 — Project Foundation and Scope Control

Primary question

«Can Aren establish a development environment and decision process that make incorrect behaviour visible without building product infrastructure prematurely?»

Phase 0 is not an architecture phase. It creates only the minimum repository foundation needed to develop Phase 1 safely.

Goals

- Establish a small Go repository.
- Make builds, tests, static checks and race detection easy to run.
- Define project terminology before APIs proliferate.
- Record explicit scope boundaries.
- Create lightweight architectural decision records.
- Prevent roadmap work from turning into implementation speculation.

In scope

Repository foundation

A minimal structure such as:

aren/
├── cmd/
│   └── aren/
├── internal/
├── docs/
│   ├── decisions/
│   ├── roadmap.md
│   └── glossary.md
├── examples/
├── go.mod
├── Makefile
└── README.md

The exact structure should remain small and may change during Phase 1.

Engineering feedback

Commands for:

- formatting,
- unit tests,
- race-enabled tests,
- static analysis,
- coverage inspection,
- and building the CLI.

Initial glossary

Define only the terms already needed:

- execution,
- run,
- state,
- event,
- result,
- failure,
- cancellation,
- executor.

Definitions may be explicitly marked provisional.

Scope ledger

Maintain a lightweight document containing:

- current phase scope,
- explicit exclusions,
- deferred ideas,
- evidence required to reconsider each deferred idea.

Decision records

Use short decision records only for foundational choices that would otherwise become ambiguous.

Likely initial records:

- Go as the first implementation language.
- Library-first runtime.
- No daemon in the initial phases.
- Local and in-memory operation initially.
- No stable public API promise before foundational semantics settle.

Out of scope

- provider integrations,
- execution persistence,
- databases,
- servers,
- plugin systems,
- workflow definitions,
- SDK generation,
- broad configuration frameworks,
- observability backends,
- production deployment,
- semantic versioning guarantees.

Deliverables

- buildable Go module,
- minimal CLI entry point,
- automated test and race-test commands,
- glossary,
- phase scope ledger,
- decision-record format,
- initial repository README.

Validation

- Clean checkout can build and test with one documented command.
- Race tests run in local development and CI.
- No production runtime abstractions are introduced solely for repository setup.
- Every nontrivial package created in this phase has an immediate purpose.

Exit gate

Phase 0 is complete when the repository reliably supports Phase 1 development without carrying speculative runtime architecture.

---

5. Phase 1 — Execution Lifecycle

Primary question

«Can Aren define and enforce the lifecycle of one execution without depending on an LLM, subprocess, network call or persistent store?»

This is the most foundational phase.

The objective is not to create a universal executor interface. It is to establish the minimum semantics that Aren itself owns.

Conceptual model

A run represents one supervised occurrence of work.

A preliminary lifecycle might be:

created
   ↓
running
   ├──→ succeeded
   ├──→ failed
   └──→ cancelled

A cancellation request may occur while running:

running
   ↓
cancellation requested
   ↓
cancelled

Whether "cancellation_requested" should be a durable state, an event, or both must be resolved through implementation.

The lifecycle should remain smaller than typical workflow-engine state machines.

Goals

- Assign every run a unique identity.
- Define who owns state transitions.
- Define legal and illegal transitions.
- Represent terminal outcomes explicitly.
- Separate lifecycle state from output data.
- Establish cancellation semantics.
- Establish the first canonical event vocabulary.
- Define how consumers observe events.
- Define a minimum failure representation.
- Make illegal internal behaviour fail visibly.

Core concepts to decide

Run identity

Determine:

- identifier type,
- generation ownership,
- whether callers may provide IDs,
- and whether IDs carry semantic information.

Default direction:

- opaque unique IDs,
- generated by Aren,
- no timestamp or execution-type meaning embedded in the ID.

Lifecycle state

The state set should be deliberately small.

Candidate states:

- "created"
- "running"
- "succeeded"
- "failed"
- "cancelled"

Questions to resolve:

- Is "created" externally observable?
- Is cancellation request a state or an event?
- Can a run fail before entering "running"?
- Is rejected execution a failed run or a failure to create a run?
- Can cleanup failure alter an otherwise successful outcome?

Terminal outcome

State alone is insufficient.

A terminal outcome may need to include:

- terminal state,
- result value,
- classified failure,
- start time,
- finish time,
- cancellation metadata.

Result and failure must be mutually coherent.

Examples:

succeeded + result
failed + failure
cancelled + cancellation reason

A run must not become:

succeeded + failure
failed + successful result
running + terminal timestamp

State ownership

Aren, not the executor implementation, should own legal lifecycle transitions.

The executor may report work outcomes, but it should not directly mutate run state.

Cancellation

Cancellation must answer:

- Who may request it?
- Is it idempotent?
- What happens when cancellation races with completion?
- Is cancellation cooperative?
- When is a run considered cancelled?
- Can cancellation fail?
- What happens if underlying work ignores cancellation?
- Does cancellation reason belong in the result?
- Are repeated requests visible as repeated events?

Initial direction:

- cancellation request is idempotent,
- first terminal outcome wins,
- Aren propagates a Go context,
- Aren does not claim work stopped until controlled work returns,
- cancellation request and cancellation completion are distinct,
- a successful result that wins the race may remain successful even if cancellation was requested concurrently.

That direction must be proven through tests rather than accepted only in prose.

Event ordering

The first event vocabulary should describe only lifecycle behaviour.

Candidate events:

- "run.created"
- "run.started"
- "run.cancellation_requested"
- "run.succeeded"
- "run.failed"
- "run.cancelled"

Potential metadata:

- run ID,
- monotonic sequence number,
- event type,
- occurrence time,
- payload.

Questions to resolve:

- Are sequence numbers per run or global?
- Are timestamps authoritative for order?
- Can event delivery lag state transition?
- What happens when a consumer is slow?
- Can consumers miss events?
- Is event publication synchronous with state mutation?
- Are event payloads immutable?

Initial direction:

- ordering is established by per-run sequence number, not timestamps,
- state transition and event creation occur atomically within the runtime,
- event delivery mechanism may lag but must preserve order,
- events are immutable values,
- consumer failure must not corrupt run state.

Event consumption

Avoid building a general event bus.

Phase 1 needs only enough to observe one run.

Potential first API:

run.Events() <-chan Event

or:

subscription := run.Subscribe()

However, a raw channel introduces important semantics:

- buffer size,
- slow consumer handling,
- closure ownership,
- missed events,
- multiple subscribers,
- replay behaviour.

A simpler callback or internal recorder may initially be safer. The implementation should select the smallest model that can be tested honestly.

Failure model

Start with a minimal structured failure.

Likely categories:

- execution failure,
- cancellation,
- internal invariant violation.

Do not create the eventual provider and tool error taxonomy yet.

The representation should preserve:

- machine-readable category,
- human-readable message,
- wrapped cause where applicable,
- and whether the failure is internal or attributable to executed work.

Time

Time is observable state and must be testable.

The runtime needs:

- start time,
- terminal time,
- possibly duration.

A clock abstraction is justified only if deterministic lifecycle tests genuinely require it. A tiny internal clock function may be sufficient; a public "Clock" interface is probably premature.

Proposed first implementation

Implement an in-process run controller around a concrete work function:

type Work func(context.Context) (Result, error)

This is not yet the permanent executor abstraction. It is a narrow instrument for proving lifecycle semantics.

A run controller would:

1. create run identity,
2. emit creation/start events,
3. invoke the work function,
4. observe cancellation,
5. classify the returned outcome,
6. perform one legal terminal transition,
7. expose the result and event history.

Public surface

The public surface should remain minimal and may be explicitly unstable.

Possible shape:

run := aren.Start(ctx, work)

events := run.Events()

result, err := run.Wait()

run.Cancel(reason)

This is illustrative, not a required API. The PRD should specify behaviours before exact method names.

Required tests

Normal lifecycle

- created → running → succeeded
- result preserved exactly
- terminal timestamp recorded
- events ordered correctly
- waiting after completion returns immediately

Failure lifecycle

- created → running → failed
- original cause retained
- failure event emitted once
- no successful result exposed

Cancellation before start

Clarify whether this can occur and test the chosen behaviour.

Cancellation during execution

- cancellation propagated,
- cancellation request emitted once,
- cooperative work exits,
- terminal state becomes cancelled,
- waiters unblock.

Repeated cancellation

- safe,
- no duplicated terminal transitions,
- no panic,
- clear event behaviour.

Completion/cancellation race

Test many iterations under the race detector.

Required invariant:

- exactly one terminal state,
- exactly one terminal event,
- no data race,
- result agrees with terminal state.

Multiple waiters

- all receive the same immutable outcome,
- no waiter consumes the outcome exclusively,
- no deadlock.

Slow event consumer

Test the chosen backpressure behaviour explicitly.

Aren must either:

- block,
- buffer within a documented bound,
- drop according to a documented policy,
- or detach durable history from live observation.

It must not exhibit accidental behaviour.

Consumer abandonment

A run should not leak indefinitely merely because nobody reads its events.

Work panic

Decide whether panic:

- propagates,
- becomes a classified failure,
- or terminates the process.

For a reusable runtime library, converting executor panic into an internal execution failure is probably appropriate, but this should not silently hide programmer errors. The failure should remain clearly marked as a panic or invariant breach.

Invalid transition

Internal tests should prove that illegal state transitions are rejected visibly.

Development CLI

Provide a small diagnostic CLI for exercising lifecycle behaviour:

aren dev run success
aren dev run fail
aren dev run cancel
aren dev run race

The CLI is not the final product interface. It exists to make semantics observable outside unit tests.

Example output:

000 run.created
001 run.started
002 run.cancellation_requested
003 run.cancelled

Deliverables

- lifecycle model,
- run controller,
- terminal outcome model,
- cancellation behaviour,
- first lifecycle event vocabulary,
- deterministic event sequencing,
- minimum failure representation,
- comprehensive unit and race tests,
- diagnostic CLI,
- written lifecycle contract,
- phase review.

Out of scope

- attempts and retries,
- providers,
- model messages,
- token streaming,
- tools,
- subprocess management,
- persistence,
- event replay after process restart,
- multiple execution kinds,
- daemon transport,
- global event stream,
- workflow composition,
- production telemetry export.

Exit gate

Phase 1 is complete only when:

1. all lifecycle transitions are defined,
2. cancellation races are proven under the race detector,
3. event order is deterministic,
4. exactly one terminal outcome is guaranteed,
5. slow or absent consumers cannot accidentally deadlock or leak the run,
6. a small runnable demonstration exists,
7. and the phase review identifies no unresolved foundational semantic ambiguity.

---

6. Phase 2 — Controlled Executor and Failure Laboratory

Primary question

«Does the Phase 1 lifecycle remain coherent when execution exhibits realistic progress, delay, failure, cleanup and cancellation behaviour?»

Phase 1 proves the state machine around a minimal function.

Phase 2 creates a deliberately configurable executor used as a semantic test instrument.

This should not become a generic simulation framework.

Goals

- Exercise the lifecycle through richer controlled behaviour.
- Introduce nonterminal progress events.
- Test delayed and partially cooperative cancellation.
- Test cleanup after success, failure and cancellation.
- Determine whether attempts belong in the foundation or should wait for retries.
- Identify which executor/runtime boundary is genuinely required.
- Produce a reusable conformance suite for future executors.

Controlled behaviours

The test executor should be configurable to:

- succeed immediately,
- succeed after a delay,
- fail immediately,
- fail after progress,
- emit a fixed sequence of progress events,
- block until externally released,
- cooperate with cancellation,
- delay cancellation response,
- ignore cancellation until a safe point,
- panic,
- fail during cleanup,
- race completion against cancellation,
- produce a partial output before failing.

Configuration should remain typed and explicit. Avoid a general scenario language.

Progress events

Introduce one generic progress mechanism only if needed.

Possible event:

execution.progress

Payload might contain:

- stage or label,
- message,
- optional numeric progress,
- executor-defined metadata.

The risk is creating an unbounded universal payload too early.

A better initial direction may be:

type Progress struct {
    Message string
}

Future executor-specific events should not be anticipated prematurely.

Cleanup semantics

Execution often needs cleanup:

- releasing resources,
- closing streams,
- terminating child operations,
- flushing data.

Questions:

- Does cleanup happen before the terminal event?
- Can cleanup change success into failure?
- What if cleanup is cancelled?
- Is cleanup allowed a separate timeout?
- Are cleanup errors joined with execution failures?
- What does cancellation mean while cleanup is running?

Initial direction:

- terminal state is not published until required cleanup completes,
- cleanup errors are not silently discarded,
- an execution that completed its main work but failed mandatory cleanup should not automatically be called successful,
- cleanup should not run indefinitely under an already-cancelled context.

Executor boundary

Phase 2 should test whether a concrete internal abstraction is now justified.

Possible shape:

type Executor interface {
    Execute(context.Context, Reporter) (Result, error)
}

But this interface should not be adopted merely because future executor types are imag
