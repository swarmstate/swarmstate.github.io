# LangGraph checkpointer

In LangGraph, a **checkpointer** is what persists a graph's state after every step, so a
conversation (a *thread*) can be paused, resumed, inspected, or rewound. LangGraph ships
`InMemorySaver` (fast but not persistent) and `SqliteSaver` (persistent but slower -
every step commits to disk).

`SwarmStateSaver` is a **drop-in replacement** for either: same
`BaseCheckpointSaver` interface (`put`, `put_writes`, `get_tuple`, `list`, plus the async
variants), but backed by a swarmstate [`Store`](store.md) with a Rust core. You get
faster checkpoints than SQLite (see the [benchmarks](../benchmarks.md)), and - because the
state lives in a `Store` - the ability to snapshot or roll back **every thread at once**.
Point it at a persistent backend ([`DiskStore`](disk.md), [`RedisStore`](redis.md) or
[`PostgresStore`](postgres.md)) and those checkpoints survive restarts, too.

```bash
pip install "swarmstate[langgraph]"
# or: uv add "swarmstate[langgraph]"
```

## One-line swap

```python
from swarmstate.integrations.langgraph import SwarmStateSaver

graph = builder.compile(checkpointer=SwarmStateSaver())   # was SqliteSaver(...)
```

Everything else - `invoke`, `stream`, `get_state`, `get_state_history`, resuming a
thread - works unchanged.

```python
config = {"configurable": {"thread_id": "user-42"}}
graph.invoke({"messages": [("user", "hi")]}, config)
graph.get_state(config)          # resumes from the persisted checkpoint
```

## Share one store across graphs

Pass a `Store` explicitly to share checkpoints across savers/graphs (or to keep several
independent ones):

```python
import swarmstate as ss
from swarmstate.integrations.langgraph import SwarmStateSaver

store = ss.Store()
saver_a = SwarmStateSaver(store)
saver_b = SwarmStateSaver(store)   # same underlying checkpoint DB
```

## Snapshot / roll back the whole checkpoint DB

Because checkpoints live in a `Store`, you get cheap, atomic snapshots of **every
thread at once** - useful for tests, "what-if" branches, or recovery:

```python
saver = SwarmStateSaver()
graph = builder.compile(checkpointer=saver)

snap = saver.store.snapshot()     # O(1) snapshot of all threads
# ... run more turns across many threads ...
saver.store.restore(snap)         # roll everything back
```

## Serialization

By default the saver uses LangGraph's `JsonPlusSerializer` (so it handles the same
objects LangGraph does). Pass your own via `serde=...`:

```python
SwarmStateSaver(serde=my_serializer)
```

## Incremental storage (opt-in)

For long threads with large, mostly-stable channels, pass `incremental=True`. Each
channel value is then stored once per version (deduplicated) instead of writing the
whole checkpoint every step, saving storage and serialization:

```python
SwarmStateSaver(store, incremental=True)
```

Trade-off: `get_tuple` then does one read per channel to reassemble state, versus a
single read in the default mode. Leave it off unless channels are large and change
rarely.

## Durable, shared checkpoints

The saver holds checkpoints in whatever `Store` you give it, so making them persistent is
a backend swap, nothing else changes:

```python
from swarmstate.backends.disk import DiskStore          # a SQLite file, no server
from swarmstate.integrations.langgraph import SwarmStateSaver

saver = SwarmStateSaver(DiskStore("checkpoints.db"))     # or RedisStore / PostgresStore
graph = builder.compile(checkpointer=saver)
```

See the [Disk](disk.md), [Redis](redis.md) and [Postgres](postgres.md) guides for the
"which backend?" trade-offs.

## Observability

Pass a metrics sink to measure the latency and outcome of each checkpoint operation, with
zero overhead when unused:

```python
from swarmstate.observability import InMemoryMetrics     # or OpenTelemetryMetrics

metrics = InMemoryMetrics()
saver = SwarmStateSaver(metrics=metrics)
# ... run the graph ...
metrics.summary()   # {"put": {"count": ..., "p50_ms": ...}, "get_tuple": {...}}
```

Full details in the [Observability guide](observability.md).

## Async

The async methods (`aput`, `aput_writes`, `aget_tuple`, `alist`) run the store work on a
worker thread (`asyncio.to_thread`), so they don't block the event loop; the store
releases the GIL on its hot paths, so that work overlaps with the loop. `ainvoke` /
`astream` and async graphs work out of the box.
