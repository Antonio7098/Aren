# Programmatic Tool Calling for Coding Agents

> **Status:** Speculative design and benchmark note  
> **Relationship to roadmap:** This should be treated as a later experiment built on normal tool representation, local tool execution, lifecycle events, cancellation, and the bounded agent loop. It is not part of Aren's foundational semantics.

## Purpose

This document explores whether a coding agent built with Aren should be able to write a short Python or JavaScript program that combines a restricted set of typed tools using loops, conditionals, concurrency, filtering, aggregation, and other ordinary programming constructs.

The proposed model is:

```text
model writes Python or JavaScript
        ↓
program calls a restricted set of typed Aren tools
        ↓
loops, branching, parallelism, filtering, aggregation
        ↓
intermediate results remain inside the program runtime
        ↓
only a reduced result returns to the model
```

The hypothesis is not merely that generated code can call several tools. A shell already provides that capability. The stronger hypothesis is:

> Typed, programmatic composition of semantic repository tools can improve task quality or efficiency enough to justify a dedicated execution surface alongside direct tool calls and shell execution.

This must be demonstrated through a proof of concept and benchmark rather than assumed.

---

# Existing Precedents

There is concrete evidence that this pattern has already been implemented and evaluated.

## Anthropic Programmatic Tool Calling

Anthropic's Programmatic Tool Calling allows Claude to write Python that invokes opted-in tools. Intermediate tool results are processed inside the code-execution environment rather than being inserted into Claude's context one by one. Only the program's final output is returned to the model.

Anthropic presents the capability as useful for:

- dependent tool calls;
- loops and conditionals;
- parallel operations;
- filtering and aggregation;
- processing large intermediate results without polluting model context.

Anthropic reports internal results on complex research tasks in which average usage fell from 43,588 to 27,297 tokens, a 37% reduction. It also reports improvements from 25.6% to 28.5% on an internal knowledge-retrieval evaluation and from 46.5% to 51.2% on GIA. These are vendor-reported results and are not coding-agent-specific, so they should be treated as supporting evidence rather than proof for Aren's use case.

Source: <https://www.anthropic.com/engineering/advanced-tool-use>

## OpenAI Programmatic Tool Calling

OpenAI exposes Programmatic Tool Calling as a hosted JavaScript execution model. Eligible tools can be called from generated JavaScript, with results passed between calls and intermediate outputs processed inside the hosted runtime.

OpenAI recommends it for bounded, tool-heavy stages involving predictable processing such as:

- filtering;
- joining;
- ranking;
- deduplication;
- aggregation;
- validation.

OpenAI explicitly warns that multiple or dependent calls alone do not justify programmatic calling. Direct calls remain preferable when one call is sufficient, intermediate outputs are small, each result may change the model's next semantic decision, approval is required, or native citations and artifacts must be preserved.

Its guidance also recommends benchmarking direct and programmatic calling on representative tasks using task success, answer completeness, required evidence, total tokens, latency, cost, calls, turns, and retries. Lower resource usage should only count as an improvement when the final answer continues to pass the quality bar.

Source: <https://developers.openai.com/api/docs/guides/latest-model>

## Cloudflare Code Mode

Cloudflare Code Mode presents configured tools as typed methods and gives the model one code-execution tool. The model writes JavaScript that calls tools, passes results between calls, applies control flow, filters intermediate data, and returns a focused result.

Cloudflare distinguishes the patterns as follows:

| Pattern | Best suited for |
|---|---|
| Direct tool calls | Simple tasks using a small, known tool set |
| Code Mode | Composed or dependent calls, loops, branching, filtering, result shaping, progressive discovery, or reusable logic |

Cloudflare runs generated code inside an isolated sandbox and supports several integration forms, including server tools, browser-owned tools, MCP tools, and a durable runtime with execution history and approval handling.

Sources:

- <https://developers.cloudflare.com/agents/tools/codemode/>
- <https://developers.cloudflare.com/agents/concepts/tools/>

## CodeAct Research

The 2024 CodeAct paper proposed executable Python as a unified action space for LLM agents instead of constrained JSON or textual actions. The authors evaluated 17 models across API-Bank and a newly constructed benchmark and reported improvements of up to 20 percentage points in success rate over commonly used alternatives.

CodeAct is broader than the proposed Aren experiment because it treats executable code as the agent's general action language. It nevertheless provides independent research evidence that executable code can improve action composition and flexibility.

Sources:

- <https://proceedings.mlr.press/v235/wang24h.html>
- <https://arxiv.org/abs/2402.01030>

---

# The Coding-Agent Caveat

A comparison against only direct `read` calls would be too weak.

> Bash is already a form of programmatic tool calling.

A coding agent with shell access can write loops, pipelines, filters, temporary scripts, and conditional logic:

```bash
for file in $(rg -l 'Executor'); do
    sed -n '1,200p' "$file"
done
```

Or:

```bash
rg -l 'RegisterTool' |
    xargs grep -n 'Permission' |
    head -50
```

It can also create and run a Python or JavaScript script using ordinary command execution.

Programmatic structured tools must therefore compete with a mature environment that is already highly expressive for filesystem traversal, text processing, source inspection, and command composition.

A useful benchmark should include at least three arms:

| Arm | Interface |
|---|---|
| Direct tools | `search_text`, `read_source`, `find_symbol`, and similar tools called one model-mediated step at a time |
| Shell | `bash` plus repository utilities such as `rg`, `find`, `sed`, language tooling, and temporary scripts |
| Programmatic tools | Python or JavaScript calling typed Aren tools inside a restricted runtime |

A later fourth arm could allow the model to choose between direct tools, shell, and programmatic execution.

The likely outcomes are not uniform:

- Programmatic tools may outperform repeated direct calls on wide, mechanical exploration.
- Bash may remain better for simple filesystem and text-search tasks.
- Programmatic tools may outperform Bash when composing richer semantic operations such as symbol resolution, references, implementations, dependency relationships, diagnostics, and versioned source evidence.
- Direct calls may remain better when each observation requires fresh semantic judgment from the model.

The difficult and valuable question is not whether programmatic calling beats naive repeated reads. It is whether it provides enough advantage over competent Bash use to justify another execution surface.

---

# Suitable Task Shapes

## Strong Candidates

Programmatic calling is most promising when a bounded stage requires:

- fan-out over many results;
- predictable filtering or aggregation;
- deduplication;
- dependency traversal;
- repeated application of the same operation;
- parallel independent calls;
- processing a large intermediate dataset into a small evidence set;
- no fresh model judgment between most individual calls.

Example:

> Find every implementation of `ToolExecutor`, inspect the execution paths around each implementation, identify implementations that appear to omit cancellation or permission propagation, and return only the relevant source references.

A JavaScript program might resemble:

```javascript
const implementations = await tools.findImplementations({
  symbol: "ToolExecutor",
});

const findings = [];

for (const implementation of implementations) {
  const references = await tools.findReferences({
    symbol: implementation.symbol,
  });

  const sources = await Promise.all(
    references.map((ref) => tools.readSource(ref)),
  );

  const suspicious = sources.filter((source) =>
    !source.text.includes("context.Context") ||
    !source.text.includes("permission")
  );

  if (suspicious.length > 0) {
    findings.push({
      implementation,
      references: suspicious.map((source) => source.ref),
    });
  }
}

return findings;
```

The model sees the reduced findings rather than every definition, reference, and irrelevant source excerpt.

## Weak Candidates

Programmatic calling is unlikely to help when:

- one tool call is sufficient;
- intermediate outputs are already small;
- each result changes the next semantic decision;
- the task is primarily explanation or judgment;
- an action needs user approval;
- native citations or artifacts would be lost through reduction;
- the model cannot know the tool return shape before writing the program.

Example:

> Read this function and explain the defect.

That task requires a source read followed by model judgment. Adding a code runtime contributes machinery without reducing meaningful work.

---

# Proposed Aren Proof of Concept

The first POC should be read-only, bounded, and deliberately disposable.

It should prove or disprove the approach without committing Aren to a general scripting subsystem.

## Candidate Tool Set

Expose a small set of typed repository tools:

```text
listFiles
searchText
readSource
findSymbol
findReferences
importsOf
```

A seventh tool such as `findImplementations` could be added if it can be implemented reliably for the selected language and materially improves the benchmark tasks.

Do not initially expose:

- arbitrary shell execution;
- direct filesystem APIs;
- network access;
- file mutation;
- package installation;
- process creation;
- environment secrets;
- dynamic tool registration.

## Structured Source Results

Every source-bearing result should retain versioned source identity:

```typescript
type SourceRef = {
  path: string;
  contentHash: string;
  startLine: number;
  endLine: number;
};

type SearchMatch = {
  ref: SourceRef;
  preview: string;
};

type SourceEvidence = {
  ref: SourceRef;
  text: string;
};
```

The model receives a typed programming interface rather than independently invoking raw JSON Schema tools:

```typescript
declare const tools: {
  listFiles(input: ListFilesInput): Promise<FileRef[]>;
  searchText(input: SearchTextInput): Promise<SearchMatch[]>;
  readSource(input: SourceRef): Promise<SourceEvidence>;
  findSymbol(input: FindSymbolInput): Promise<SymbolRef[]>;
  findReferences(input: FindReferencesInput): Promise<SourceRef[]>;
  importsOf(input: ImportsOfInput): Promise<ImportRef[]>;
};
```

The interface should document:

- exact input fields;
- exact return fields;
- failure representation;
- ordering guarantees;
- truncation behaviour;
- result limits;
- whether calls are safe to run concurrently.

If the model cannot determine the return shape before writing the program, direct calling is likely the more appropriate interface.

---

# Execution Semantics

Generated code must not bypass Aren's normal tool execution path.

```text
program calls tools.readSource(...)
        ↓
Aren validates arguments
        ↓
Aren resolves permission
        ↓
Aren emits tool.started
        ↓
implementation executes
        ↓
Aren emits tool.completed, tool.failed, or tool.cancelled
        ↓
result returns to the program runtime
```

The program is another caller of an Aren tool, not a privileged alternative execution mechanism.

Each nested call should retain:

- program execution ID;
- parent caller relationship;
- tool-call ID;
- tool name;
- validated inputs;
- permission decision;
- start and completion timestamps;
- duration;
- result size;
- failure category;
- cancellation state;
- evidence references produced.

The top-level program execution should retain:

- generated source code;
- language and runtime version;
- declared eligible tools;
- execution limits;
- nested call count;
- aggregate bytes processed;
- final output;
- stdout and stderr policy;
- terminal outcome.

This prevents a single visible `program` call from hiding the execution evidence Aren is intended to preserve.

---

# Required Limits

The POC should begin with hard boundaries:

```text
maximum program duration
maximum nested tool calls
maximum concurrency
maximum loop or execution budget
maximum individual tool-result size
maximum aggregate bytes processed
maximum final output size
cancellation
read-only tools
no direct network access
no direct filesystem access
no process creation
```

Additional rules:

- No silent retries initially.
- No recursive program execution.
- No mutation tools.
- No continuation after cancellation.
- No treating a partial program result as success unless the output contract permits it.
- No unbounded collection of tool results in memory.

A generated loop that retries a failed tool indefinitely would otherwise undermine Aren's bounded execution semantics.

---

# Runtime Choice

The POC should use one language, not both.

## JavaScript Advantages

- natural fit with current provider-hosted programmatic calling patterns;
- straightforward async and `Promise.all` composition;
- relatively simple typed declaration generation;
- embeddable runtimes such as QuickJS or isolated V8-based options;
- no need to expose a full Python standard library.

## Python Advantages

- strong model familiarity;
- concise data processing;
- close alignment with Anthropic's existing implementation and CodeAct research;
- convenient iteration, filtering, and aggregation;
- potential future reuse for data-analysis tasks.

## Recommendation

Use the runtime that can be sandboxed most simply and deterministically in the POC environment.

The benchmark is intended to evaluate the action model, not to choose a permanent scripting language. Runtime portability and multi-language support should be out of scope.

---

# Benchmark Design

Begin with approximately 20 to 30 tasks against frozen repository revisions.

Each task should have a known answer or a mechanically checkable evidence set wherever possible.

## Arm A: Direct Tools

The model invokes one structured tool call at a time and receives each result in its context before deciding the next action.

## Arm B: Shell

The model receives a normal shell tool and can use repository utilities, language tools, pipelines, and temporary scripts.

## Arm C: Programmatic Tools

The model writes one or more bounded programs that call the restricted typed repository tools. Intermediate results remain inside the sandbox unless deliberately returned.

## Optional Arm D: Hybrid Routing

The model may choose direct tools, shell, or programmatic execution.

This arm should only be added after the first three establish the strengths and weaknesses of each interface. Otherwise routing quality becomes entangled with execution quality too early.

---

# Task Categories

## 1. Fan-Out and Filtering

Examples:

- Find all implementations of an interface and identify those missing a required pattern.
- Search every command handler for calls to a deprecated API.
- Inspect every matching test and return only those exercising cancellation.
- Find every event emission for a lifecycle event and identify inconsistent payload fields.

Expected advantage: programmatic tools.

## 2. Dependency Traversal

Examples:

- Find transitive dependants of a package up to depth three.
- Locate all callers of a symbol, then all tests touching those callers.
- Determine which packages may be affected by changing an interface.
- Traverse imports from a public API to the concrete execution implementation.

These tasks test:

- loops;
- queues;
- visited-set deduplication;
- depth limits;
- stopping conditions;
- bounded result shaping.

Expected advantage: programmatic tools when semantic dependency functions are reliable; shell may remain competitive for simple package-level traversal.

## 3. Adaptive Exploration

Examples:

- Trace the code path responsible for an observed behaviour.
- Identify the probable cause of an error using a stack trace.
- Explain where a lifecycle invariant can be violated.
- Determine why cancellation fails to reach a specific child execution.

Each observation may materially change what should be inspected next.

Expected advantage: direct tools or shell, because fresh model judgment may be more important than mechanical composition.

## 4. Simple Controls

Examples:

- Read one file.
- Find one exact symbol.
- Search for one literal string.
- Return the imports of one file.

Programmatic calling should probably lose on these tasks. That is useful benchmark evidence rather than a failure.

## 5. Large Intermediate Result Reduction

Examples:

- Search a large repository for a broad pattern, then return only matches satisfying several structural conditions.
- Read many test files and return only tests containing a specific setup and assertion combination.
- Collect diagnostics across packages and return only unique failures grouped by root cause.

Expected advantage: programmatic tools if the sandbox can process large intermediate results without admitting them into model context.

---

# Ground Truth and Verification

The benchmark should evaluate final correctness before efficiency.

Each task should define some combination of:

```text
required paths
required symbols
required source ranges
required relationship edges
forbidden false positives
required final conclusions
required caveats
acceptable alternative evidence
```

Prefer mechanical verification where possible.

For judgment-heavy tasks, use a stable rubric and blind evaluation rather than relying solely on an LLM judge.

A task should fail if the agent returns a plausible conclusion without the required source evidence.

---

# Metrics

## Quality

- task success;
- evidence recall;
- evidence precision;
- unsupported claims;
- answer completeness;
- required caveats preserved;
- false-positive rate.

## Model Use

- number of model turns;
- input tokens;
- output tokens;
- cached-input tokens;
- reasoning tokens where available;
- model invocations.

## Context

- peak context size;
- intermediate bytes shown to the model;
- duplicated source tokens;
- number and size of source excerpts admitted;
- evidence retained in the final answer.

## Execution

- direct tool calls;
- nested program tool calls;
- failed calls;
- invalid arguments;
- retries;
- program syntax failures;
- runtime failures;
- timeouts;
- cancellations;
- maximum observed concurrency.

## Performance

- total wall-clock time;
- model latency;
- tool execution time;
- sandbox startup time;
- program runtime;
- overhead per nested call.

## Cost

- provider cost;
- local execution cost;
- sandbox infrastructure cost;
- cost per successful task.

## Traceability

- percentage of conclusions linked to source evidence;
- completeness of nested-call event history;
- ability to reproduce the final evidence set;
- ability to distinguish program filtering from model judgment.

Run each task several times per arm. A single run per task is likely to measure sampling variance rather than interface quality.

---

# Critical Data-Flow Measurement

Do not compare only total model tokens.

Measure three separate quantities:

```text
bytes returned by underlying tools
bytes processed inside the sandbox
bytes admitted into model context
```

The central efficiency hypothesis may look like:

```text
12 MB read from repository
600 KB processed by program
8 KB returned to model
```

versus:

```text
12 MB read from repository
600 KB returned through direct tool calls
600 KB accumulated in model context
```

That separation is the actual value proposition. Tool execution may process the same amount of repository data in both cases while model-context admission differs dramatically.

---

# POC Success Criteria

The POC should be considered promising only if it demonstrates at least one meaningful advantage over both direct tools and shell.

Possible success criteria:

1. Higher task success on semantic fan-out or dependency tasks at similar cost.
2. Equivalent task success with materially fewer model tokens or turns.
3. Equivalent quality with substantially lower peak context use.
4. Better evidence precision by filtering irrelevant intermediate results before they reach the model.
5. More reliable composition of semantic tools than shell scripts can provide.

The POC should not be considered successful merely because:

- it performs fewer visible model tool calls;
- generated code looks elegant;
- it beats a deliberately weak direct-read baseline;
- it processes many files;
- it uses fewer tokens while omitting required evidence;
- it hides failures inside one top-level program result.

---

# Hypotheses to Test

The experiment should explicitly test these hypotheses:

## H1 — Composition

Models compose semantic repository tools more effectively in generated code than through sequential function calls.

## H2 — Context Reduction

Keeping intermediate results inside the program runtime materially reduces context use without reducing final answer quality.

## H3 — Shell Differentiation

Typed programmatic tools provide enough advantage over Bash and temporary scripts to justify a separate execution surface.

## H4 — Task-Shape Routing

Programmatic calling helps bounded mechanical stages but harms or adds overhead to tasks requiring frequent fresh semantic judgment.

## H5 — Traceability

Nested tool-call provenance can remain complete and understandable even when orchestration occurs inside generated code.

## H6 — Boundedness

Hard execution limits can prevent runaway programs without creating excessive false failures on valid tasks.

---

# Likely Architectural Placement

Programmatic tool calling should initially be implemented as an experimental caller layered on top of existing Aren capabilities:

```text
Tool-call representation
        ↓
Local tool execution
        ↓
Lifecycle events and caller relationships
        ↓
Bounded agent loop
        ↓
Experimental program caller
```

It should not initially define:

- Aren's core tool model;
- the general context model;
- persistence semantics;
- workflow composition;
- a universal sandbox API;
- a permanent plugin system.

If the POC succeeds, it may justify a later execution type with explicit parent-child semantics. If it does not outperform direct tools and shell on representative coding tasks, it should remain an experiment rather than becoming framework surface area.

---

# Current Judgment

There is sufficient precedent to justify a POC.

There is not yet sufficient evidence to conclude that programmatic tool calling belongs in Aren.

The external evidence shows that executable tool composition can reduce context pollution, remove model round trips, and improve results on some tool-heavy tasks. However, those results do not answer the coding-agent-specific comparison against Bash, language servers, repository utilities, and generated temporary scripts.

The decisive benchmark is therefore:

> Can a model using typed, programmatic repository tools produce better or cheaper grounded coding results than the same model using either direct structured tools or a competent shell environment?

Until that is demonstrated, the capability should remain a narrow, read-only, bounded experiment.