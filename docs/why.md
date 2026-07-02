# Why swarmstate

Agent frameworks (LangGraph, CrewAI, custom loops) are great at *orchestration* — but the
**state and checkpointing** underneath them is where production teams hit walls.
`swarmstate` is the fast, framework-agnostic engine for exactly that layer.

## Three problems it solves

### 1. State lock-in across frameworks
State written by one framework is trapped in that framework's format. Migrating from
CrewAI to LangGraph (or running both) means losing or re-plumbing accumulated state.

`swarmstate` stores everything in a single [`Store`](guide/store.md) with a **stable,
language-neutral msgpack format** — any agent, any framework, any language reads and
writes the same state. See [State portability](tutorials/portability.md).

### 2. Checkpointing cost and latency
LangGraph's per-node checkpointing backed by SQLite/Postgres becomes a bottleneck at
scale: every step serializes and writes state. `swarmstate` implements LangGraph's
checkpointer interface with a **Rust core** — fast serialization, **O(1) immutable
snapshots** via structural sharing, incremental diffs, and the **GIL released** on hot
paths. It is a [one-line swap](tutorials/langgraph.md) for `SqliteSaver`.

### 3. Deterministic routing paid for in tokens
Many "which agent handles this next" decisions are *rules over a dependency graph*, not
judgement calls — yet they're often paid for with an LLM round-trip. The native
[`HandoffGraph`](guide/handoff.md) resolves those transitions in **microseconds, in
Rust**, with a safe condition language (never `eval`). See
[Custom multi-agent loop](tutorials/custom-loop.md).

## What it is *not*

`swarmstate` does not compete with visible agent frameworks; it acts as low-level
infrastructure, much like engines such as DuckDB, ClickHouse, Arrow, or Polars can sit
underneath data applications without replacing them. You keep your framework — swarmstate
makes its state layer fast and portable.

## Who it's for

- Teams with a **"checkpoint latency"** ticket or an orchestration bill that doesn't add up.
- Anyone running **multiple frameworks** and needing shared/portable state.
- Systems that need **deterministic, auditable routing** without spending tokens on it.

## Where to start

<div class="ss-features" markdown="1">
<div class="ss-card" markdown="1">
### LangGraph, faster
[Drop-in checkpointer →](tutorials/langgraph.md) — swap `SqliteSaver`, keep everything else.
</div>
<div class="ss-card" markdown="1">
### Custom orchestration
[Build a multi-agent loop →](tutorials/custom-loop.md) — routing + state, no framework.
</div>
<div class="ss-card" markdown="1">
### No lock-in
[Share state across frameworks →](tutorials/portability.md) — one store, many agents.
</div>
</div>

!!! note "Benchmarks"
    On the LangGraph interface, `SwarmStateSaver.put` is **~12.8× faster than
    `SqliteSaver`**, and `Store.snapshot()` is **O(1)** (hundreds of thousands of times
    faster than deep-copying a large state). Full methodology, tables and charts —
    reproducible, hardware/versions documented — are on the
    [Benchmarks](benchmarks.md) page. We publish numbers, not adjectives.
