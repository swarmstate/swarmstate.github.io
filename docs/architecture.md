# Architecture

`swarmstate` follows a **Rust compute core + thin Python API** design (via PyO3 +
maturin), distributed as cross-platform `cp39-abi3` wheels.

```
swarmstate/
├─ rust/src/
│  ├─ lib.rs         # #[pymodule] — exports classes to Python
│  ├─ store.rs       # concurrent KV store + immutable snapshots
│  ├─ codec.rs       # msgpack (de)serialization
│  ├─ checkpoint.rs  # LangGraph checkpoint types            (M3)
│  └─ graph.rs       # handoff/dependency graph               (M2)
└─ python/swarmstate/
   ├─ __init__.py    # public API: Store, Snapshot, ...
   ├─ _core.pyi      # type stubs for the Rust module
   └─ integrations/  # langgraph.py, crewai.py, redis.py      (M3+)
```

## Design principles

- **Hot logic lives in Rust.** Serialization, snapshot diffs and (later) graph
  traversal run entirely in the Rust core.
- **The GIL is released** (`py.allow_threads`) on any operation that doesn't touch
  Python objects — lock acquisition, map mutation, snapshot cloning.
- **Immutable snapshots via structural sharing.** The store is backed by a persistent
  `im::HashMap`; cloning it is O(1) and snapshots are isolated from later writes.
- **Stable, language-neutral state format.** Values serialize to msgpack so state is
  readable from any language or framework — the antidote to state lock-in.
- **Thin, fully typed Python API.** Ergonomic wrappers over the PyO3 classes with
  complete type hints and `py.typed`.

## Why a Rust core?

Per-node checkpointing backed by SQLite/Postgres becomes a bottleneck at scale, and
many "which agent is next" routing decisions don't need an LLM. Moving serialization,
snapshotting and rule-based routing into a Rust core — with the GIL released on hot
paths — targets both the latency and the token cost that production teams actually pay
for.

Benchmarks quantifying this (vs LangGraph's `SqliteSaver` and a plain in-memory dict)
land in **M4**; see the [Roadmap](roadmap.md).
