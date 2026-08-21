# API reference

The public API is intentionally small - three classes and a helper. This page is the
exact reference; for explanations and examples, see the [Guide](guide/index.md) and
[Tutorials](tutorials/index.md).

The core classes are implemented in Rust and exposed through the native
`swarmstate._core` module; full type stubs ship with the package (`py.typed`), so editors
give you autocomplete and type checks. Adapters (`SwarmStateSaver`, `SwarmStateStorage`)
and the `RedisStore` backend live in `swarmstate.integrations.*` / `swarmstate.backends.*`.

## `swarmstate.Store`

```python
Store(backend: str = "memory", codec: str = "msgpack", max_history: int | None = 0)
```

A framework-agnostic state store with immutable snapshots.

**Properties**

| Property | Type | Description |
| --- | --- | --- |
| `codec` | `str` | serialization codec in use (currently `"msgpack"`) |
| `max_history` | `int \| None` | snapshots the store retains: `0` none (default), `n` the last `n`, `None` unlimited |

**Methods**

| Method | Returns | Description |
| --- | --- | --- |
| `set(namespace, key, value)` | `None` | store `value` under `(namespace, key)` |
| `get(namespace, key, default=None)` | `Any` | value at `(namespace, key)`, or `default` if absent |
| `set_many(items)` | `None` | store many `(namespace, key, value)` triples in one call (one GIL release, one lock per shard) |
| `get_many(pairs)` | `list[Any]` | fetch many `(namespace, key)` pairs in input order; missing entries come back as `None` |
| `contains(namespace, key)` | `bool` | whether `(namespace, key)` exists |
| `delete(namespace, key)` | `bool` | delete; `True` if a value was removed |
| `keys(namespace, prefix=None)` | `list[str]` | keys within a namespace, optionally only those starting with `prefix` |
| `namespaces(prefix=None)` | `list[str]` | all namespaces, optionally only those starting with `prefix` (filtered inside the store, so only matches are copied out) |
| `max_key(namespace)` | `str \| None` | greatest key in a namespace, without building a key list |
| `clear()` | `None` | remove all entries (keeps snapshot history) |
| `snapshot()` | `Snapshot` | capture a cheap, immutable snapshot |
| `restore(snapshot)` | `None` | roll back to a snapshot |
| `history()` | `list[Snapshot]` | snapshots the store retained, oldest first (empty unless `max_history` was set) |
| `clear_history()` | `None` | drop every retained snapshot, releasing the state they pin |
| `len(store)` | `int` | total number of entries |
| `namespace in store` | `bool` | whether a namespace holds at least one key |

Raises `TypeError` for unsupported value types, and `ValueError` for an unknown
`backend`/`codec`, out-of-range integers, or a value nested deeper than 128 levels
(including self-referential ones).

## `swarmstate.Snapshot`

An immutable, point-in-time view of a `Store`. Created via `Store.snapshot()`; not
instantiated directly.

**Properties**

| Property | Type | Description |
| --- | --- | --- |
| `id` | `int` | monotonic id assigned by the originating store |
| `timestamp` | `float` | seconds since the Unix epoch when taken |
| `parent` | `int \| None` | id of the previous snapshot from the same store |
| `size_bytes` | `int` | total serialized size of all values |
| `keys` | `list[tuple[str, str]]` | all `(namespace, key)` pairs present |

**Methods**

| Method | Returns | Description |
| --- | --- | --- |
| `diff(base)` | `dict[str, list[tuple[str, str]]]` | `{"added", "removed", "changed"}` → `(namespace, key)` lists describing how to go from `base` to this snapshot |

## `swarmstate.HandoffGraph`

```python
HandoffGraph(on_cycle: str = "error")
```

A deterministic, LLM-free routing graph over named nodes with conditional edges.
`on_cycle` is `"error"` (default) or `"allow"`. See the [Handoff graph
guide](guide/handoff.md) for the condition mini-language.

**Properties**

| Property | Type | Description |
| --- | --- | --- |
| `on_cycle` | `str` | cycle-detection behaviour (`"error"` / `"allow"`) |

**Methods**

| Method | Returns | Description |
| --- | --- | --- |
| `add_node(name)` | `None` | register a node with no edges |
| `add_edge(from_node, to, when=None)` | `None` | add a directed edge, optionally guarded by a `when` condition |
| `route(node, state=None)` | `str \| None` | first matching outgoing edge's target, or `None` |
| `nodes()` | `list[str]` | all nodes, sorted |
| `edges(node)` | `list[tuple[str, str \| None]]` | outgoing `(to, when)` pairs, insertion order |
| `has_node(node)` | `bool` | whether a node exists |
| `is_dag()` | `bool` | whether the graph is currently acyclic |
| `len(graph)` | `int` | number of nodes |
| `node in graph` | `bool` | membership test |

Raises `ValueError` for an invalid `on_cycle`, an invalid condition, or (when
`on_cycle="error"`) an edge that would create a cycle.

## `swarmstate.dumps` / `swarmstate.loads`

```python
dumps(obj: Any) -> bytes
loads(data: bytes) -> Any
```

Serialize/deserialize with swarmstate's stable **msgpack** codec (the same one the store
uses). The output is standard msgpack, so it round-trips with any msgpack library in any
language:

```python
import swarmstate as ss, msgpack

raw = ss.dumps({"a": 1, "b": [1, 2], "c": b"\xff"})
msgpack.unpackb(raw, raw=False)          # -> {"a": 1, "b": [1, 2], "c": b"\xff"}
ss.loads(msgpack.packb({"x": 1}))        # -> {"x": 1}
```

Supported types: `None`, `bool`, `int` (64-bit), `float`, `str`, `bytes`, `list`,
`tuple` (returns as `list`), `dict`. Unsupported types raise `TypeError`.

## `swarmstate.core_version()`

```python
core_version() -> str
```

Returns the version string of the compiled Rust core (matches
`swarmstate.__version__`).

## `swarmstate.protocols`

The store interface every backend satisfies, written down for type checking. These are
typing constructs, not base classes: a store is conformant by having the methods.

```python
from swarmstate.protocols import StoreLike, SnapshotLike

def build_saver(store: StoreLike) -> SwarmStateSaver:
    return SwarmStateSaver(store)
```

`tests/test_store_conformance.py` in the repository runs the same assertions — key/value
semantics, prefix scans, `max_key`, snapshot metadata, diffs, restore — against every
bundled backend, so "drop-in" is a checked property rather than a claim.

Two documented differences between implementations, both of which callers can act on:
`snapshot()` is O(1) on the Rust store and an O(n) copy on the persistent backends; and a
store whose `max_key` is index-backed advertises `indexed_max_key = True`, which is how
`SwarmStateSaver` decides whether to keep its own latest pointer.

## `swarmstate.integrations.langgraph.SwarmStateSaver`

```python
SwarmStateSaver(
    store: StoreLike | None = None,
    *,
    serde: SerializerProtocol | None = None,
    incremental: bool = False,
    max_checkpoints_per_thread: int | None = None,
    metrics: MetricsSink | None = None,
    tracer: Tracer | None = None,
)
```

A drop-in `BaseCheckpointSaver`. See the [LangGraph guide](guide/langgraph.md).

| Parameter | Default | Description |
| --- | --- | --- |
| `store` | `Store()` | the backing store; shareable across savers and graphs |
| `serde` | LangGraph's default | serializer for checkpoint payloads |
| `incremental` | `False` | store channel values once per version instead of per step |
| `max_checkpoints_per_thread` | `None` | keep only the newest N checkpoints of each thread, dropping older ones with their writes and unreferenced blobs |
| `metrics` | `None` | a [metrics sink](guide/observability.md) for per-operation latency |
| `tracer` | `None` | an OpenTelemetry tracer; each operation becomes a span |

Raises `ValueError` for `max_checkpoints_per_thread < 1`.

## `swarmstate.observability`

Opt-in metrics and tracing on checkpoint operations. Attach them to the checkpointer with
`SwarmStateSaver(metrics=..., tracer=...)`; see the
[Observability guide](guide/observability.md).

```python
class MetricsSink(Protocol):
    def record(self, op: str, duration_s: float, *, thread_id: str, ok: bool) -> None: ...
```

| Sink | Description |
| --- | --- |
| `NullMetrics()` | discards everything (the default when no sink is passed) |
| `InMemoryMetrics()` | accumulates counts + latency; `.summary()` → `{op: {count, errors, mean_ms, p50_ms, p99_ms}}`, `.reset()` |
| `OpenTelemetryMetrics(meter=None)` | emits an OTel histogram + counter (needs `swarmstate[otel]`) |

```python
get_tracer(name: str = "swarmstate") -> Tracer
```

Returns an OpenTelemetry tracer (lazy import; needs `swarmstate[otel]`) for use as
`SwarmStateSaver(tracer=get_tracer())`. Each operation opens a `swarmstate.checkpoint.<op>`
span; on failure the span records the exception and is marked `ERROR`.

`op` is one of `"put"`, `"put_writes"`, `"get_tuple"`. With neither `metrics` nor `tracer`
set, the saver adds no overhead at all.
