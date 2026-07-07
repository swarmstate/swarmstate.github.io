# The Store

The `Store` is where swarmstate keeps state. Think of it as a fast, in-memory
dictionary with two differences that matter for agent systems:

1. **Keys have two parts** - a *namespace* and a *key* - so you can group related state
   (one namespace per workflow, agent, or thread) instead of inventing string prefixes.
2. **Values are stored as [msgpack](https://msgpack.org) bytes**, a compact binary format
   that any language can read. That's what makes state portable across frameworks and
   what lets snapshots and checkpoints be cheap.

Everything else in swarmstate - snapshots, the LangGraph checkpointer, the Redis
backend - is built on top of the `Store`.

```python
import swarmstate as ss

store = ss.Store()

# set(namespace, key, value)
store.set("workflow", "onboarding", {"step": 3, "data": {"user": 42}})

# get(namespace, key)
store.get("workflow", "onboarding")          # -> {"step": 3, "data": {"user": 42}}
```

## Creating a store

`Store()` with no arguments is an in-memory store using the msgpack codec - the right
choice for most cases.

```python
store = ss.Store()                           # in-memory, msgpack (defaults)
store = ss.Store(max_history=100)            # keep at most 100 snapshots
```

| Parameter | Default | Meaning |
| --- | --- | --- |
| `backend` | `"memory"` | where state lives. `"memory"` today; for persistence use [`RedisStore`](redis.md). |
| `codec` | `"msgpack"` | how values are serialized. msgpack is stable and cross-language. |
| `max_history` | `None` | how many snapshots to retain (`None` = unlimited). Cap it in long-running processes to bound memory. |

## Reading and writing

The four everyday operations. Note that `get` returns `None` (or your `default`) for a
missing key rather than raising - handy for "read state if it exists" flows.

```python
store.set("agents", "researcher", {"status": "running", "tokens": 1200})

store.get("agents", "researcher")            # -> {"status": "running", "tokens": 1200}
store.get("agents", "missing")               # -> None
store.get("agents", "missing", default={})   # -> {}   (supply your own fallback)

store.contains("agents", "researcher")       # -> True
store.delete("agents", "researcher")         # -> True  (False if nothing was there)
```

## Batch reads and writes

When you have many entries, `set_many` and `get_many` do the whole batch in one call:
the in-memory core releases the GIL once and locks each shard once for the batch (not once
per item), and the persistent backends turn it into a single round-trip (a Redis pipeline,
one SQL statement). `get_many` preserves input order and returns `None` for missing keys.

```python
store.set_many([
    ("agents", "researcher", {"status": "running"}),
    ("agents", "writer", {"status": "idle"}),
    ("workflow", "main", {"step": 3}),
])

store.get_many([("agents", "researcher"), ("agents", "missing"), ("workflow", "main")])
# -> [{"status": "running"}, None, {"step": 3}]
```

This is worth it for bulk loads and for networked backends, where the per-call round-trip
dominates. On a free-threaded build the win is larger still (see
[Architecture](../architecture.md#free-threaded-no-gil)).

## Inspecting what's stored

```python
store.keys("agents")        # keys within one namespace  -> ["researcher", ...]
store.namespaces()          # every namespace             -> ["agents", "workflow", ...]
len(store)                  # total (namespace, key) entries
store.clear()               # wipe everything (snapshots you already took are kept)
```

## What you can store

Values go through the msgpack codec, which supports the JSON-like types plus `bytes` and
`tuple`:

> `None`, `bool`, `int` (64-bit), `float`, `str`, `bytes`, `list`, `tuple` (comes back as
> a `list`), and `dict` - nested arbitrarily.

```python
store.set("bin", "blob", b"\x00\x01\xff")    # bytes are preserved byte-for-byte
store.get("bin", "blob")                     # -> b"\x00\x01\xff"
```

Anything else (a custom object, a set, a NumPy array) raises `TypeError`, and integers
beyond the 64-bit range raise `ValueError` - so serialization problems surface
immediately at `set()` time, not later.

## Why it's safe to share across threads

Agent systems are often concurrent (parallel tool calls, worker pools). The `Store` is
thread-safe, and - because the hot work happens in Rust with the **GIL released** - many
Python threads can read and write the *same* store genuinely in parallel, instead of
queuing behind the interpreter lock:

```python
import threading

store = ss.Store()

def worker(tid):
    for i in range(1000):
        store.set(f"t{tid}", str(i), {"tid": tid, "i": i})

threads = [threading.Thread(target=worker, args=(t,)) for t in range(8)]
for t in threads:
    t.start()
for t in threads:
    t.join()

len(store)   # -> 8000, with no locking on your side
```

On **standard CPython** the GIL still serializes the Python-side work, so this is about
correctness and not blocking, more than raw multi-core speed. On a **free-threaded
(no-GIL) build** the same code scales across cores: see
[Architecture: free-threaded](../architecture.md#free-threaded-no-gil).

## Where to go next

- [Snapshots & diffs](snapshots.md) - capture and roll back state cheaply.
- [Redis backend](redis.md) - the same API, but persistent and shared across processes.
- [LangGraph checkpointer](langgraph.md) - let LangGraph store its checkpoints here.
