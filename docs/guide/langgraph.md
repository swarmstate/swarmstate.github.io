# LangGraph checkpointer

In LangGraph, a **checkpointer** is what persists a graph's state after every step, so a
conversation (a *thread*) can be paused, resumed, inspected, or rewound. LangGraph ships
`InMemorySaver` (fast but not persistent) and `SqliteSaver` (persistent but slower -
every step commits to disk).

`SwarmStateSaver` is a **drop-in replacement** for either: same
`BaseCheckpointSaver` interface (`put`, `put_writes`, `get_tuple`, `list`,
`delete_thread`, plus the async variants), but backed by a swarmstate [`Store`](store.md)
with a Rust core. What you get is **read latency that does not drift as a thread grows**
(see the [benchmarks](../benchmarks.md)) and - because the state lives in a `Store` - the
ability to snapshot or roll back **every thread at once**. Point it at a persistent backend
([`DiskStore`](disk.md), [`RedisStore`](redis.md) or [`PostgresStore`](postgres.md)) and
those checkpoints survive restarts, too.

Writes are a different story, and worth stating plainly: durable writes land **on par**
with `SqliteSaver` at a matched fsync policy, and in-memory writes are *slower* than
`InMemorySaver`, because state is serialized to msgpack on the way in. That serialization
is what buys the portability and the cheap snapshots.

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

## Why reads stay flat

Resuming a thread means answering "which checkpoint is the newest?". The reference savers
scan the thread's keys for the maximum, so a thread that has run for a week costs more per
resume than a fresh one. `SwarmStateSaver` looks it up instead:

- **in-memory and Redis stores** — the saver publishes a *latest pointer* next to each
  checkpoint, in the same batched write, and reads it in O(1).
- **SQL backends** ([`DiskStore`](disk.md), [`PostgresStore`](postgres.md)) — no pointer at
  all: they answer `max_key` from the `(namespace, key)` primary-key index, which is both
  always current and one row less to write per step.

Either way the answer is `max(checkpoint_id)`, matching `InMemorySaver`, and it comes from
the store — so two savers, or two processes, sharing one backend always agree on which
checkpoint is newest.

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

## Retention

Checkpointers keep every step by default. For a service that never restarts, that is
unbounded growth:

```python
saver = SwarmStateSaver(max_checkpoints_per_thread=8)   # keep the newest N per thread
```

Older checkpoints are dropped along with their pending writes and, in incremental mode,
the channel blobs no surviving checkpoint still references. On a 300-invocation thread
that is **0.5 MB instead of 28 MB**, and the thread still resumes.

Two things to know: trimming happens in batches, so a thread sits slightly above the limit
before it is cut back (the limit is a bound, not an exact length), and time travel is
limited to the retained window — size it to the history you actually want to rewind to.

Graphs with [`DeltaChannel`](#deltachannel) state keep a little more: such a channel is
rebuilt from the checkpoints before the newest one, back to its last snapshot, so those
stay too. Retention still bounds the thread; the bound is just "the newest N plus what
they are rebuilt from".

## Managing threads

LangGraph's checkpointer interface also defines thread-management operations, which
`InMemorySaver` and `SqliteSaver` leave unimplemented. `SwarmStateSaver` implements them
(and their `a`-prefixed async twins):

```python
saver.copy_thread("user-42", "user-42-experiment")   # full history, parent chain included
saver.prune(["user-42"])                             # keep only the latest checkpoint
saver.prune(["user-42"], strategy="delete")          # or remove the thread
saver.delete_for_runs(["run-7f3a"])                  # drop what one run wrote, everywhere
```

- **`copy_thread`** copies every checkpoint, pending write and channel blob, so the copy
  resumes and time-travels like the source and then moves on independently.
- **`prune`** with `"keep_latest"` keeps the newest checkpoint of each checkpoint
  namespace, plus what `DeltaChannel` state is rebuilt from.
- **`delete_for_runs`** matches the `run_id` LangGraph records in each checkpoint's
  metadata, which comes from `config["metadata"]["run_id"]`. Give each run its own
  `run_id`: LangGraph reads a repeated one as re-entering the same run. It reads the
  metadata of every checkpoint, so it is an administrative operation, not a hot path.

## DeltaChannel

LangGraph 1.2 adds `DeltaChannel`, a reducer channel that stores a value only at snapshot
points and rebuilds it in between by replaying the writes of earlier checkpoints, through
the saver's `get_delta_channel_history`. LangGraph's default implementation rebuilds a full
`CheckpointTuple` - every channel value, all metadata, every pending write - for each
ancestor, and the chain can run up to the channel's `snapshot_frequency` (1000 by default)
checkpoints long. `SwarmStateSaver` reads the stored records directly and decodes only the
requested channel: **2.2x faster** with `incremental=True`, 1.3x without, on a chain of 100
runs.

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

`ainvoke` / `astream` and async graphs work out of the box. How the async methods
(`aput`, `aput_writes`, `aget_tuple`, `alist`) reach the store depends on the backend:

- **[`RedisStore`](redis.md) and [`PostgresStore`](postgres.md)** have a native async API,
  and the saver awaits it directly: no worker thread blocks on the network.
- **The in-memory `Store` and [`DiskStore`](disk.md)** run each call on a worker thread
  (`asyncio.to_thread`), so it doesn't block the event loop; the Rust store releases the
  GIL on its hot paths, so that work overlaps with the loop.

Administrative operations (`adelete_thread`, `acopy_thread`, `aprune`,
`adelete_for_runs`) always take the worker-thread route.
