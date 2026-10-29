# Roadmap

`swarmstate` is built in verifiable milestones. Status below reflects what is shipped.

| Milestone | Scope | Status |
| --- | --- | --- |
| **M0** | Scaffolding - maturin project, Rust core, Python API, CI | ✅ done |
| **M1** | Rust store - concurrent KV, msgpack codec, O(1) snapshots, incremental diffs, GIL released | ✅ done |
| **M2** | `HandoffGraph` - conditional DAG, safe condition evaluator (no `eval`), deterministic `route()`, cycle detection | ✅ done |
| **M3** | LangGraph adapter - `SwarmStateSaver` implementing `BaseCheckpointSaver` (drop-in for `SqliteSaver`) | ✅ done |
| **M4** | Benchmarks - checkpoint latency (p50/p99) and throughput vs `SqliteSaver` and in-memory dict | ✅ done |
| **M5** | CrewAI adapter + optional Redis backend (state portability across frameworks) | ✅ done |
| **M6** | Docs + cross-platform abi3 wheels + PyPI release via Trusted Publishing (OIDC) | ✅ done |

## Since 0.1

Beyond the initial milestones, these have shipped:

| Area | Scope | Status |
| --- | --- | --- |
| **Backends** | [`DiskStore`](guide/disk.md) (SQLite file) and [`PostgresStore`](guide/postgres.md), alongside [`RedisStore`](guide/redis.md) - all drop-in, all msgpack wire-format | ✅ done |
| **Incremental checkpoints** | opt-in per-version channel dedup in `SwarmStateSaver` (`incremental=True`) for long threads with large, stable channels | ✅ done |
| **Sharded write locking** | namespace-hashed `RwLock`s so writers to different namespaces don't serialize (removes the lock bottleneck for free-threaded builds) | ✅ done |
| **Observability** | opt-in [metrics hooks](guide/observability.md) on `put`/`put_writes`/`get_tuple` with in-memory and OpenTelemetry sinks (`swarmstate[otel]`), plus OpenTelemetry tracing | ✅ done |
| **Batch operations** | [`Store.set_many` / `get_many` / `delete_many` / `delete_namespace`](guide/store.md#batch-reads-and-writes) and on every backend: one GIL release / round-trip per batch | ✅ done |
| **Free-threaded (no-GIL)** | Rust core declares `gil_used(false)`; version-specific [`cp313t` and `cp314t` wheels](architecture.md#free-threaded-no-gil) where the store holds throughput under threads instead of collapsing (~10x the GIL build at 8 threads) | ✅ done |
| **Bounded memory** | [`SwarmStateSaver(max_checkpoints_per_thread=N)`](guide/langgraph.md#retention) trims old checkpoints with their writes and channel blobs; `Store(max_history=...)` governs retained snapshots and defaults to keeping none | ✅ done |
| **One store contract** | [`swarmstate.protocols`](api.md#swarmstateprotocols) writes the store interface down for type checking, and a conformance suite runs the same assertions against every bundled backend | ✅ done |
| **Thread management** | [`copy_thread`, `prune`, `delete_for_runs`](guide/langgraph.md#managing-threads) from LangGraph's checkpointer interface, `DeltaChannel`-aware retention and a direct [`get_delta_channel_history`](guide/langgraph.md#deltachannel) | ✅ done |
| **Routing inside LangGraph** | [`handoff_router`](guide/handoff.md#inside-a-langgraph-graph) for `add_conditional_edges`; conditions gain list literals and `not in`, and route over dataclass / Pydantic states | ✅ done |
| **Portability tools** | [`copy_store`, `dump_store`, `load_store`](tutorials/portability.md#5-move-state-to-another-backend): move state between any two backends, or through a portable file | ✅ done |
| **Native async** | `RedisStore` and `PostgresStore` expose an async API (`AsyncStoreLike`) that `SwarmStateSaver`'s async methods await directly | ✅ done |
| **Developer experience** | strict `mypy` in CI, exposed `dumps`/`loads` codec, runnable [examples](https://github.com/swarmstate/swarmstate/tree/main/examples) | ✅ done |

## Under consideration

- **A namespace index for Redis** — `namespaces()` on `RedisStore` still walks the keyspace
  with `SCAN`; an index set per thread would make `list()` and `delete_thread()` cost the
  thread, not the whole Redis.
- **Docs versioning** (per-release documentation) once a stable line is cut.
- **LangGraph `BaseStore` and `BaseCache`** — long-term memory and node caching on the
  same engine, alongside the checkpointer.
- **More codec types** — `datetime`, `UUID`, `Decimal` and `set` via documented msgpack
  extension types, instead of the current `TypeError`.

## Guiding rules

- **Zero install friction** - `pip install swarmstate` must work with no compiler.
- **Never break the drop-in** - `SwarmStateSaver` stays swappable for `SqliteSaver`.
- **Reproducible benchmarks** - document hardware, versions and cache state.
- **Stable state format** - the msgpack codec won't change incompatibly across minor versions.
- **Safe by construction** - `HandoffGraph` conditions run in a bounded Rust evaluator, never `eval()`.

Track progress on [GitHub](https://github.com/swarmstate/swarmstate).
