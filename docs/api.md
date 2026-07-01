# API reference

The public API is intentionally small. All classes are implemented in Rust and exposed
through the native `swarmstate._core` module; full type stubs ship with the package
(`py.typed`).

## `swarmstate.Store`

```python
Store(backend: str = "memory", codec: str = "msgpack", max_history: int | None = None)
```

A framework-agnostic state store with immutable snapshots.

**Properties**

| Property | Type | Description |
| --- | --- | --- |
| `codec` | `str` | serialization codec in use (currently `"msgpack"`) |
| `max_history` | `int \| None` | max retained snapshots (`None` = unlimited) |

**Methods**

| Method | Returns | Description |
| --- | --- | --- |
| `set(namespace, key, value)` | `None` | store `value` under `(namespace, key)` |
| `get(namespace, key, default=None)` | `Any` | value at `(namespace, key)`, or `default` if absent |
| `contains(namespace, key)` | `bool` | whether `(namespace, key)` exists |
| `delete(namespace, key)` | `bool` | delete; `True` if a value was removed |
| `keys(namespace)` | `list[str]` | keys within a namespace |
| `namespaces()` | `list[str]` | all namespaces |
| `clear()` | `None` | remove all entries (keeps snapshot history) |
| `snapshot()` | `Snapshot` | capture a cheap, immutable snapshot |
| `restore(snapshot)` | `None` | roll back to a snapshot |
| `len(store)` | `int` | total number of entries |

Raises `TypeError` for unsupported value types and `ValueError` for an unknown
`backend`/`codec` or out-of-range integers.

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

## `swarmstate.core_version()`

```python
core_version() -> str
```

Returns the version string of the compiled Rust core (matches
`swarmstate.__version__`).
