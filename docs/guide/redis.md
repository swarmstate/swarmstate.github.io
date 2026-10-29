# Redis backend

The default [`Store`](store.md) is in-memory: fast, but its state is gone when the
process exits and can't be shared with other processes. When you need state to
**survive restarts** or be **shared across workers/services**, use `RedisStore` - it has
the *exact same interface* as `Store`, so it drops in anywhere a store is expected
(including the [LangGraph checkpointer](langgraph.md)) with no other code changes.

Values are serialized with **msgpack** - the same wire format as the Rust core - so
anything written here is readable by any msgpack consumer, in any language.

```bash
pip install "swarmstate[redis]"
# or: uv add "swarmstate[redis]"
```

## Use it like a `Store`

```python
from swarmstate.backends.redis import RedisStore

store = RedisStore("redis://localhost:6379/0")
store.set("workflow", "onboarding", {"step": 3})
store.get("workflow", "onboarding")     # -> {"step": 3}
store.keys("workflow")                  # -> ["onboarding"]
store.namespaces()                      # -> ["workflow"]
```

It implements the full interface: `set`, `get`, `contains`, `delete`, `keys`,
`namespaces`, `clear`, `len(store)`, the batch operations (`set_many`, `get_many`,
`delete_many`, `delete_namespace`), plus `snapshot()` / `restore()` (copy-based, O(n) -
Redis persists rather than offering the Rust store's O(1) structural-sharing snapshots).

Batched writes and deletes, and `restore()`, run as one `MULTI`/`EXEC` transaction: a
connection dropped halfway applies none of the batch, so a checkpoint never lands without
the latest pointer written beside it, and an interrupted restore leaves the data as it was.

## Async

Every key/value operation has an `a`-prefixed async twin (`aget`, `aset_many`, `akeys`,
...) on a `redis.asyncio` client. [`SwarmStateSaver`](langgraph.md#async) awaits them from
its async methods, so an async graph never parks a worker thread on Redis:

```python
store = RedisStore("redis://localhost:6379/0")
await store.aset("workflow", "onboarding", {"step": 3})
await store.aget("workflow", "onboarding")          # -> {"step": 3}
await store.aclose()                                # the async client of this event loop
```

A store built from a URL opens its async client lazily, one per event loop (Redis async
connections belong to the loop that opened them). A store given only `client=` has no
async support (`store.supports_async` is `False`) unless you also pass `aclient=`; the
saver then falls back to a worker thread.

## Persistent LangGraph checkpoints

Because `RedisStore` shares the store interface, it drops straight into
[`SwarmStateSaver`](langgraph.md) - giving you **persistent** checkpoints that survive
process restarts and are shared across workers:

```python
from swarmstate.backends.redis import RedisStore
from swarmstate.integrations.langgraph import SwarmStateSaver

saver = SwarmStateSaver(RedisStore("redis://localhost:6379/0"))
graph = builder.compile(checkpointer=saver)
# a fresh process pointed at the same Redis resumes every thread
```

## Layout & format

- Each namespace is a Redis **hash** at `{prefix}:{namespace}` (default prefix
  `swarmstate`); fields are keys, values are msgpack bytes.
- The msgpack encoding is standard, so a value written here decodes with any msgpack
  library - the foundation of [cross-framework state portability](../tutorials/portability.md).

## Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `url` | `redis://localhost:6379/0` | Redis connection URL |
| `client` | `None` | pass an existing `redis.Redis` client instead of a URL |
| `aclient` | `None` | an existing `redis.asyncio.Redis` client for the async methods |
| `prefix` | `"swarmstate"` | key prefix namespacing this store |
| `codec` | `"msgpack"` | value serialization (stable, cross-language) |
