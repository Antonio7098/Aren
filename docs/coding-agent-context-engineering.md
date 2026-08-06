# Coding-Agent Exploration and Context Engineering

> **Status:** Speculative design note  
> **Relationship to roadmap:** Potentially informs Phase 9 — Minimal Bounded Agent Loop and Phase 10 — Context Engineering. This document does not commit Aren to the complete design described here.

## Purpose

This document explores how a coding agent built with Aren might inspect a repository efficiently while managing repeated source reads, context growth, provenance, freshness, and provider prompt caching.

The central idea is:

> Aren should not prevent repeated reads. It should prevent unnecessary repeated context injection while preserving the meaning of the read request.

Reading the same code twice can mean several different things. A correct design must distinguish accidental duplication from freshness checks, deliberate re-presentation, and restoration after context reduction.

---

## The Duplication Problem

A repeated code read can produce four different kinds of duplication.

### 1. Filesystem duplication

Aren reads the same bytes from disk again.

This is usually inexpensive and may be necessary to verify that the source has not changed.

### 2. Evidence duplication

Aren stores another internal copy of the same source span and source version.

This is normally unnecessary when identical evidence has already been acquired.

### 3. Prompt duplication

Aren appends the same source code to the logical conversation again, leaving multiple copies visible to the model.

This grows the context and may degrade clarity even when the provider discounts cached input.

### 4. Provider reprocessing

The next model request contains previously sent input and the provider processes or bills it again.

Provider prompt caching may reduce this cost, but it does not eliminate logical context growth or make duplicated source harmless.

These are related but separate concerns. A coding agent should be able to verify the workspace without automatically duplicating source in the model context.

---

## Important Tension: Context Minimisation Versus Cache Stability

Provider prompt caching generally benefits from a stable prompt prefix. Aggressively rebuilding, reordering, or regenerating the supposedly optimal context on every turn may reduce the nominal token count while destroying cache reuse.

The desired system is therefore not simply:

> Produce the smallest possible prompt on every turn.

It is closer to:

> Preserve a stable, append-friendly prefix; avoid unnecessary duplication; and introduce explicit context-reduction boundaries only when growth creates real pressure.

Aren should model semantic context independently from provider caching. Provider adapters can then project that context into the most appropriate request shape for each provider.

---

# Proposed Conceptual Model

A coding agent may need four conceptually separate layers.

## 1. Workspace State

The workspace layer represents what currently exists in the repository.

```text
workspace
├── repository revision
├── working-tree generation
├── files
├── file content hashes
└── language and index capabilities
```

A Git commit is not sufficient because a coding agent commonly works with uncommitted modifications. Source identity should therefore include the actual content version.

A minimal source reference might look like:

```go
type SourceRef struct {
    Path        string
    ContentHash string
    StartLine   int
    EndLine     int
}
```

The content hash is more trustworthy than modification time. If a file changes and later returns to identical content, its evidence identity should reflect the bytes the agent actually observed.

A symbol-specific reference might additionally contain:

```go
type SymbolRef struct {
    SourceRef
    Language      string
    QualifiedName string
}
```

These symbol semantics belong to the coding-agent layer. Aren's foundational runtime should not need to understand Go symbols, Python modules, TypeScript imports, or other language-specific concepts.

---

## 2. Evidence Store

Every meaningful exploration result can be represented as versioned evidence.

```go
type Evidence struct {
    ID             EvidenceID
    Source         SourceRef
    Representation Representation
    ContentHash    string
    Content        []byte
}
```

Possible representations include:

- raw source;
- source excerpt;
- symbol definition;
- reference result;
- dependency result;
- diagnostic;
- diff;
- derived summary.

The evidence store answers:

> Has this exact information already been acquired from this exact source version?

It does not answer:

> Is this information still visible and usable in the model's current context?

That belongs to the logical context ledger.

---

## 3. Logical Context Ledger

The context ledger records what the model has actually been shown and what happened to it later.

```text
Evidence E17
Source: internal/executor.go, lines 80–146
Version: sha256:...
Representation: raw source
Introduced: turn 3
State: active
```

Possible states might eventually include:

- `active`;
- `compacted`;
- `superseded`;
- `summarised`;
- `excluded`.

Evidence may still exist in Aren's local store after it has been removed from the active model context. After compaction, Aren may need to re-present existing evidence even though it does not need to reread the source from disk.

This distinction is essential:

> Evidence availability and context presence are not the same thing.

---

## 4. Provider Request Projection

The provider projection turns the logical context into a concrete model request.

This layer may decide:

- ordering;
- token budgeting;
- stable prefix boundaries;
- context reduction;
- provider-specific cache controls;
- whether stored evidence needs reinjection;
- which dynamic metadata belongs near the end of the request.

A useful division of responsibility is:

```text
Coding agent
    owns code-specific exploration and evidence policy

Aren context layer
    owns provenance, presence, reduction, and projection

Provider adapter
    owns provider request format, cache controls, and cache metrics
```

Provider-managed conversation state or cache objects can be useful transport optimisations, but they should not become Aren's only source of truth.

---

# Repeated-Read Semantics

A repeated read should not have one universal response.

| Situation | Likely behaviour |
|---|---|
| Exact source version and range are active in context | Return a short evidence reference without appending the code again |
| Evidence exists locally but was compacted out | Reinject the stored evidence |
| File content has changed | Read and inject the new version, superseding the old evidence |
| Requested range partially overlaps existing evidence | Initially return the complete requested range; optimise overlap only after real pressure appears |
| Agent intentionally wants the source brought back into focus | Re-present it deliberately |

The final case matters. A model may request a reread because it wants the source close to the current turn, not because it has forgotten that the source exists.

A future read operation might distinguish intentions such as:

```json
{
  "path": "internal/executor.go",
  "start_line": 80,
  "end_line": 146,
  "presentation": "ensure"
}
```

Possible modes could later include:

- `ensure`: ensure the evidence is available, reusing it when possible;
- `refresh`: verify the workspace and present the current version;
- `represent`: deliberately show the evidence again.

These modes should not be added until real agent runs demonstrate that the distinction is useful.

---

## Example: First Read

```text
tool: read_source
path: internal/executor.go
lines: 80-146
```

Result:

```text
Evidence: E17
Source: internal/executor.go@7b91…#L80-L146

<source code>
```

## Example: Exact Repeated Read

```text
Evidence E17 already covers the requested range.
Source version is unchanged.
The evidence remains active in context.
```

The source code is not appended a second time.

## Example: Source Changed

```text
Evidence: E28
Supersedes: E17
Source: internal/executor.go@19af…#L80-L151

<new source code>
```

## Example: Evidence Was Compacted

```text
Evidence E17 has been restored from the evidence store.

<source code>
```

The evidence is re-presented without pretending it was newly discovered.

---

# Coding-Agent Exploration Tools

A coding agent will likely need more precise tools than ordinary file reading and text search.

A possible initial set is:

```text
list_tree
find_files
search_text
read_source
find_symbol
find_references
imports_of
importers_of
git_status
git_diff
apply_patch
run_command
```

This is only a candidate set. Each tool must earn its place through a real workflow.

## Avoid a Vague Dependency Tool

A single `find_dependencies` tool would hide several different questions:

- Which packages does this file import?
- Which files import this file or package?
- Which symbols reference this symbol?
- Which functions call this function?
- Which functions does it call?
- Which types implement this interface?
- Which tests exercise this code?
- Which configuration values affect this path?

These should initially be separate tools or explicit query modes with clear result semantics.

---

## Prefer Location-First Exploration

Exploration tools should usually return locations before returning large amounts of content.

```text
find_symbol("Execute")
    → candidate symbol references

find_references(symbol_ref)
    → bounded source references

read_source(source_ref)
    → selected source evidence
```

This avoids injecting complete definitions and every reference when the agent only needs to choose where to look next.

It also separates:

- discovery;
- selection;
- source acquisition;
- context presentation.

For the first Go coding agent, concrete Go-aware implementations such as `gopls`, `go list`, and textual search may be sufficient. Aren should not begin by designing a universal language-independent code graph.

---

# Cache-Aware Request Shape

A provider request might broadly follow this shape:

```text
Stable prefix
├── tool definitions
├── core agent instructions
├── stable repository instructions
└── earlier append-only conversation

Growing suffix
├── new model output
├── new tool calls
├── new evidence
└── current instruction

Highly dynamic tail
├── current budget
├── current workspace status
└── turn-specific metadata
```

The important rule is:

> Do not rebuild or reorder prior context merely to make it aesthetically cleaner.

Provider caching still does not make an indefinitely growing context safe. Cached input remains part of the model's usable context and can still create attention and capacity problems.

Aren will therefore eventually need both:

- stable request prefixes that permit provider caching;
- explicit compaction boundaries for long-running work.

A possible progression is:

```text
Turns 1–14: append-only, highly cacheable
Turn 15: explicit context checkpoint and reduction
Turns 16 onward: new append-only segment
```

This creates natural cache epochs without continuously rewriting history.

---

# Recommended Incremental Progression

The complete evidence and context architecture should not be implemented before Aren has a working bounded agent loop.

## Phase 9: Observe the Problem

For the initial bounded agent, Aren could:

1. give source-bearing tool results stable, human-readable references;
2. maintain an append-only conversation;
3. detect exact repeated source results;
4. record duplication metrics while behaving conservatively;
5. capture provider-reported cached-input usage where available;
6. run real repository-inspection and coding tasks.

Useful measurements include:

```text
exact rereads
overlapping rereads
rereads after mutation
duplicate source tokens appended
source tokens currently active
provider cached-input tokens
context size per turn
reads requested after compaction
```

This should reveal whether repeated reads are common enough to justify dedicated semantics.

## Phase 10: First Earned Optimisation

The first optimisation should remain deliberately narrow:

> Exact-version, exact-range evidence reuse within one run.

The deterministic rule is:

```text
same source
same content hash
same range
still active
→ append a reference instead of another source copy
```

Do not begin with:

- overlap merging;
- semantic equivalence detection;
- repository-wide retrieval-augmented generation;
- universal context policies;
- a language-independent code graph;
- cross-run evidence reuse.

Those capabilities should follow demonstrated pressure rather than anticipation.

---

# Architectural Boundary

The strongest framing is not a smart file-read cache.

It is:

> A versioned evidence ledger combined with a cache-aware provider projection.

That framing handles several concerns without conflating them:

- source freshness;
- context presence;
- evidence provenance;
- compaction;
- re-presentation;
- provider prompt caching;
- coding-language specificity.

Aren should own general execution and context semantics. A coding agent built on Aren should own source exploration, symbol meaning, dependency queries, and code-specific evidence policy.

---

# Open Questions

These questions should remain unresolved until real runs provide evidence:

1. Does a repeated read usually indicate accidental duplication or deliberate refocusing?
2. Is exact range reuse sufficient, or do overlapping reads create meaningful waste?
3. How often does source mutate between reads during a coding task?
4. Should evidence references be visible to the model, internal to Aren, or both?
5. What minimum provenance must survive context reduction?
6. Can provider cache metrics be normalised meaningfully across adapters?
7. When should a source excerpt be considered superseded rather than merely historical?
8. Does a coding agent benefit more from language-server queries or from a persistent repository index?
9. Which context transformations need lifecycle events?
10. At what point does an evidence store become necessary rather than an in-memory optimisation?

These should be answered experimentally through the bounded coding-agent workflow rather than settled in advance.
