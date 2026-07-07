# Architecture

`swarmstate` follows a **Rust compute core + thin Python API** design (via PyO3 +
maturin), distributed as cross-platform `cp39-abi3` wheels.

```
swarmstate/
├─ rust/src/
│  ├─ lib.rs         # #[pymodule] - exports classes to Python
│  ├─ store.rs       # concurrent KV store + immutable snapshots
│  ├─ codec.rs       # msgpack (de)serialization
│  ├─ checkpoint.rs  # LangGraph checkpoint types            (M3)
│  └─ graph.rs       # handoff/dependency graph               (M2)
└─ python/swarmstate/
   ├─ __init__.py       # public API: Store, Snapshot, HandoffGraph, dumps/loads
   ├─ _core.pyi         # type stubs for the Rust module
   ├─ observability.py  # opt-in metrics sinks for checkpoint ops
   ├─ integrations/     # langgraph.py, crewai.py                 (M3, M5)
   └─ backends/         # disk.py, redis.py, postgres.py          (persistent stores)
```

## Design principles

- **Hot logic lives in Rust.** Serialization, snapshot diffs and (later) graph
  traversal run entirely in the Rust core.
- **The GIL is released** (`py.allow_threads`) on any operation that doesn't touch
  Python objects - lock acquisition, map mutation, snapshot cloning.
- **Sharded write locking.** Namespaces are hashed across independent `RwLock`s, so
  concurrent writers to different namespaces don't serialize on one global lock. (On
  standard CPython the GIL still serializes the serialization step; the sharding removes
  the lock bottleneck for free-threaded builds.)
- **Immutable snapshots via structural sharing.** The store is backed by a persistent
  `im::HashMap`; cloning it is O(1) and snapshots are isolated from later writes.
- **Stable, language-neutral state format.** Values serialize to msgpack so state is
  readable from any language or framework - the antidote to state lock-in. The same wire
  format backs every persistent backend ([disk](guide/disk.md), [redis](guide/redis.md),
  [postgres](guide/postgres.md)), so a store swap never re-encodes state.
- **Thin, fully typed Python API.** Ergonomic wrappers over the PyO3 classes with
  complete type hints, `py.typed`, and strict `mypy` in CI.
- **Observability without coupling.** The checkpointer takes an optional
  [metrics sink](guide/observability.md); when none is set it adds no overhead at all.

## Free-threaded (no-GIL)

The sharded locking exists for a reason: on a **free-threaded CPython build** (`cp313t`,
[PEP 703](https://peps.python.org/pep-0703/)) there is no GIL to serialize the Python-side
work, so writers to different namespaces run genuinely in parallel. The Rust core declares
support with `m.gil_used(false)`, so importing it does not force the GIL back on.

The payoff, same set+get workload, 8 threads on Apple Silicon:

| Build | 1 thread | 8 threads | scaling |
| --- | --- | --- | --- |
| CPython 3.14 (GIL) | ~990k ops/s | ~196k ops/s | **0.2x** (slower) |
| Free-threaded 3.13t | ~1.0M ops/s | **~1.9M ops/s** | **~1.9x** (faster) |

At 8 threads the free-threaded build does roughly **10x** the throughput of the GIL build,
which gets *slower* as threads are added. Version-specific `cp313t` wheels ship alongside
the abi3 wheels (free-threaded builds can't use the stable ABI).

Batch operations ([`set_many`/`get_many`](guide/store.md#batch-reads-and-writes)) compound
this: they release the GIL once and lock each shard once for the whole batch, so on a
free-threaded build `set_many` reaches roughly **4x** the throughput of individual `set`s.

## Why a Rust core?

Per-node checkpointing backed by SQLite/Postgres becomes a bottleneck at scale, and
many "which agent is next" routing decisions don't need an LLM. Moving serialization,
snapshotting and rule-based routing into a Rust core - with the GIL released on hot
paths - targets both the latency and the token cost that production teams actually pay
for.

See the [Benchmarks](benchmarks.md) page for measured latency and snapshot-cost numbers (vs `SqliteSaver` and a plain in-memory dict).
