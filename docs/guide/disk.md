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

## Layout & format

A single table `kv(ns, k, v)` keyed by `(ns, k)`, where `v` is msgpack bytes. Because the
encoding is standard msgpack, the file is readable from any language, which is the basis
of [cross-framework state portability](../tutorials/portability.md).

## Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `path` | `"swarmstate.db"` | SQLite file path |
| `codec` | `"msgpack"` | value serialization (stable, cross-language) |
