# The Store

`Store` is a framework-agnostic key/value store. State is addressed by a
`(namespace, key)` pair and serialized to **msgpack** bytes so it can be read back by
any language or framework.

```python
import swarmstate as ss

store = ss.Store()
store.set("workflow", "onboarding", {"step": 3, "data": {"user": 42}})
store.get("workflow", "onboarding")          # -> {"step": 3, "data": {"user": 42}}
```

## Creating a store

```python
store = ss.Store()                           # defaults: memory backend, msgpack codec
store = ss.Store(max_history=100)            # cap retained snapshots
```

| Parameter | Default | Description |
| --- | --- | --- |
| `backend` | `"memory"` | storage backend (`"memory"`; `"redis"`/`"disk"` planned) |
| `codec` | `"msgpack"` | value serialization (stable across languages) |
| `max_history` | `None` | number of retained snapshots (`None` = unlimited) |

## Reading and writing

```python
store.set("agents", "researcher", {"status": "running", "tokens": 1200})

store.get("agents", "researcher")            # -> {...}
store.get("agents", "missing")               # -> None
store.get("agents", "missing", default={})   # -> {}

store.contains("agents", "researcher")       # -> True
store.delete("agents", "researcher")         # -> True (False if it wasn't there)
```

## Inspecting

```python
store.keys("agents")        # -> ["researcher", ...]  (keys within a namespace)
store.namespaces()          # -> ["agents", "workflow", ...]
len(store)                  # total number of (namespace, key) entries
store.clear()               # remove all entries
```

## Supported value types

The msgpack codec supports the JSON-like core types plus `bytes` and `tuple`:

`None`, `bool`, `int` (64-bit), `float`, `str`, `bytes`, `list`, `tuple`
(decoded back as `list`), and `dict` (arbitrarily nested).

```python
store.set("bin", "blob", b"\x00\x01\xff")    # bytes are preserved exactly
store.get("bin", "blob")                     # -> b"\x00\x01\xff"
```

Unsupported values raise `TypeError`; integers outside the signed/unsigned 64-bit
range raise `ValueError`.

## Concurrency

Every store operation releases the GIL around the actual lock and map work — only
(de)serialization runs under the GIL. This means multiple Python threads can hit the
same `Store` concurrently without serializing on the interpreter lock:

```python
import threading

store = ss.Store()

def worker(tid):
    for i in range(1000):
        store.set(f"t{tid}", str(i), {"tid": tid, "i": i})

threads = [threading.Thread(target=worker, args=(t,)) for t in range(8)]
for t in threads: t.start()
for t in threads: t.join()

len(store)   # -> 8000
```

Next: [Snapshots & diffs](snapshots.md).
