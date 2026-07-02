# Redis backend

`RedisStore` is a persistent, shareable store with the **same interface** as the
in-memory [`Store`](store.md). Values are serialized with **msgpack** — the same wire
format as the Rust core — so state is readable by any msgpack consumer, in any language.

```bash
pip install "swarmstate[redis]"
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
`namespaces`, `clear`, `len(store)`, plus `snapshot()` / `restore()` (copy-based, O(n) —
Redis persists rather than offering the Rust store's O(1) structural-sharing snapshots).

## Persistent LangGraph checkpoints

Because `RedisStore` shares the store interface, it drops straight into
[`SwarmStateSaver`](langgraph.md) — giving you **persistent** checkpoints that survive
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
  library — the foundation of [cross-framework state portability](../tutorials/portability.md).

## Parameters

| Parameter | Default | Description |
| --- | --- | --- |
| `url` | `redis://localhost:6379/0` | Redis connection URL |
| `client` | `None` | pass an existing `redis.Redis` client instead of a URL |
| `prefix` | `"swarmstate"` | key prefix namespacing this store |
| `codec` | `"msgpack"` | value serialization (stable, cross-language) |
