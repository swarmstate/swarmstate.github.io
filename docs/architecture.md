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

The store shards its write locks and releases the GIL on the hot paths, so on a
**free-threaded CPython build** (`cp313t`, [PEP 703](https://peps.python.org/pep-0703/))
it does not serialize behind the interpreter lock. The Rust core declares support with
`m.gil_used(false)`, so importing it does not force the GIL back on.

What free-threading buys here is **not linear multi-core scaling** - a set/get workload
allocates (msgpack buffers, Python result objects), and the allocator is a shared resource,
so throughput does not grow with cores. What it buys is **not collapsing**: under the GIL,
adding threads makes this workload dramatically *slower*; free-threaded holds throughput
roughly flat. Median of 5 runs, set+get, Apple Silicon:

| Build | 1 thread | 8 threads | 8-thread throughput |
| --- | --- | --- | --- |
| CPython (GIL) | ~1.8M ops/s | ~130k ops/s | baseline |
| Free-threaded (`cp313t`) | ~2.2M ops/s | **~1.8M ops/s** | **~14x the GIL build** |

So single-threaded the free-threaded build is roughly 2x (no per-call GIL machinery), and at
8 threads it sustains over **10x** the throughput of the GIL build, which has collapsed.
Version-specific `cp313t` wheels ship alongside the abi3 wheels (free-threaded builds can't
use the stable ABI).

Batch operations ([`set_many`/`get_many`](guide/store.md#batch-reads-and-writes)) help
independently: they release the GIL once and lock each shard once for the whole batch, so on
a free-threaded build at 8 threads `set_many` reaches roughly **3x** the throughput of
individual `set`s.

## Why a Rust core?

Per-node checkpointing backed by SQLite/Postgres becomes a bottleneck at scale, and
many "which agent is next" routing decisions don't need an LLM. Moving serialization,
snapshotting and rule-based routing into a Rust core - with the GIL released on hot
paths - targets both the latency and the token cost that production teams actually pay
for.

See the [Benchmarks](benchmarks.md) page for measured latency and snapshot-cost numbers (vs `SqliteSaver` and a plain in-memory dict).
