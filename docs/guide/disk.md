# Disk backend

`DiskStore` persists state to a single **SQLite file**: no server, no extra service, just
a path. It has the *exact same interface* as the in-memory [`Store`](store.md), so it
drops in anywhere a store is expected (including the
[LangGraph checkpointer](langgraph.md)) with no other code changes.

Values are serialized with **msgpack** (the same wire format as the Rust core), so state
survives process restarts and can be read by any SQLite + msgpack consumer, in any
language.

```bash
pip install "swarmstate[disk]"
# or: uv add "swarmstate[disk]"
```

(SQLite ships with Python; the extra just pulls in `msgpack`.)

## Use it like a `Store`

```python
from swarmstate.backends.disk import DiskStore

store = DiskStore("state.db")
store.set("workflow", "onboarding", {"step": 3})
store.get("workflow", "onboarding")     # -> {"step": 3}
```

It implements the full interface (`set`, `get`, `contains`, `delete`, `keys`,
`namespaces`, `clear`, `len(store)`, `snapshot()` / `restore()`), plus `close()`.

## Durable checkpoints, no server

Point [`SwarmStateSaver`](langgraph.md) at a `DiskStore` for LangGraph checkpoints that
survive restarts, without running Redis:

```python
from swarmstate.backends.disk import DiskStore
from swarmstate.integrations.langgraph import SwarmStateSaver

saver = SwarmStateSaver(DiskStore("checkpoints.db"))
graph = builder.compile(checkpointer=saver)
# restart the process, open the same file, and every thread resumes
```

## Which backend?

| Backend | Persistent | Shared across processes | Needs a server |
| --- | --- | --- | --- |
| [`Store`](store.md) (memory) | no | no | no |
| **`DiskStore`** | **yes** (a file) | one machine | no |
| [`RedisStore`](redis.md) | yes | yes (networked) | yes (Redis) |
| [`PostgresStore`](postgres.md) | yes | yes (networked) | yes (Postgres) |

## Operational notes

- **WAL + `synchronous=NORMAL`.** Crash-safe — never a corrupt file — but a power loss can
  cost the last transactions. That is a weaker guarantee than SQLite's `FULL` default, and
  it is why the [benchmarks](../benchmarks.md#checkpoint-latency-durable) compare against
  `SqliteSaver` at *both* settings.
- **One connection per thread.** In WAL mode readers run concurrently with a writer, so the
  store does not funnel them through a single mutex. `busy_timeout` (from `timeout=`, 5 s
  by default) makes concurrent writers wait rather than error.
- **Multi-statement writes are transactional.** `set_many` and `restore` run in one
  transaction, so an interrupted restore cannot leave the table half-emptied.
- **`max_key` is index-backed**, answered from the `(ns, k)` primary key — which is why the
  LangGraph saver asks this backend directly for the newest checkpoint instead of keeping a
  pointer.
- Call `close()` when done; it closes every thread's connection.

## Layout & format

A single table `kv(ns, k, v)` keyed by `(ns, k)`, where `v` is msgpack bytes. Because the
encoding is standard msgpack, the file is readable from any language, which is the basis
of [cross-framework state portability](../tutorials/portability.md).

## Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `path` | `"swarmstate.db"` | SQLite file path |
| `codec` | `"msgpack"` | value serialization (stable, cross-language) |
