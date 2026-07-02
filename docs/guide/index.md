# Guide

`swarmstate` is a **state and checkpointing engine** that sits *underneath* your agent
framework (LangGraph, CrewAI, a custom loop). It doesn't orchestrate agents - it gives
them a fast, portable place to keep state and checkpoints. See
[Why swarmstate](../why.md) for the motivation.

You work with a small set of pieces. This guide covers each in turn.

## The mental model

State lives in a **`Store`**: a `(namespace, key) → value` container, serialized with a
stable msgpack format. Everything else builds on it.

<div class="ss-features" markdown="1">

<div class="ss-card" markdown="1">
### [The Store](store.md)
The foundation - `set`/`get` values under namespaces, backed by the Rust core with the
GIL released on hot paths.
</div>

<div class="ss-card" markdown="1">
### [Snapshots & diffs](snapshots.md)
Cheap, **O(1)** immutable point-in-time copies of the whole store, plus an incremental
"what changed" between them.
</div>

<div class="ss-card" markdown="1">
### [Handoff graph](handoff.md)
Deterministic, LLM-free routing between agents/steps, with a safe condition language
(never `eval`).
</div>

<div class="ss-card" markdown="1">
### [LangGraph checkpointer](langgraph.md)
`SwarmStateSaver` - a drop-in `BaseCheckpointSaver` that stores LangGraph checkpoints in
a `Store`.
</div>

<div class="ss-card" markdown="1">
### [Redis backend](redis.md)
`RedisStore` - the same `Store` interface, but persistent and shareable across
processes.
</div>

</div>

## How the pieces fit

- The **`Store`** is the state container; **snapshots/diffs** operate on it.
- The **`HandoffGraph`** is independent - deterministic routing you can use with or
  without the store.
- The **LangGraph checkpointer** (`SwarmStateSaver`) *uses* a `Store` to hold
  checkpoints; swap the store for a **`RedisStore`** to make those checkpoints
  persistent. Because every backend shares one wire format, state is portable across
  frameworks (see [State portability](../tutorials/portability.md)).

## New here?

1. [Install](../installation.md) it (`pip install swarmstate` / `uv add swarmstate`).
2. Start with [The Store](store.md) - the core you'll use everywhere.
3. Then follow a [tutorial](../tutorials/index.md) end-to-end.
