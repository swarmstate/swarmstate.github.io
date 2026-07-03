# Postgres backend

`PostgresStore` persists state to a PostgreSQL table with the **same interface** as the
in-memory [`Store`](store.md). Values are stored as **msgpack** bytes (the same wire
format as the Rust core), so state is durable, shared across processes and machines, and
readable by any msgpack consumer.

```bash
pip install "swarmstate[postgres]"
# or: uv add "swarmstate[postgres]"
```

## Use it like a `Store`

```python
from swarmstate.backends.postgres import PostgresStore

store = PostgresStore("postgresql://user:pass@host:5432/db")
store.set("workflow", "onboarding", {"step": 3})
store.get("workflow", "onboarding")     # -> {"step": 3}
```

It implements the full interface (`set`, `get`, `contains`, `delete`, `keys`,
`namespaces`, `clear`, `len(store)`, `snapshot()` / `restore()`), plus `close()`.

## Durable, shared LangGraph checkpoints

`PostgresStore` drops straight into [`SwarmStateSaver`](langgraph.md), so LangGraph
checkpoints live in your existing Postgres, survive restarts, and are shared across
workers:

```python
from swarmstate.backends.postgres import PostgresStore
from swarmstate.integrations.langgraph import SwarmStateSaver

saver = SwarmStateSaver(PostgresStore("postgresql://user:pass@host/db"))
graph = builder.compile(checkpointer=saver)
```

## Which backend?

| Backend | Persistent | Shared across processes | Needs a server |
| --- | --- | --- | --- |
| [`Store`](store.md) (memory) | no | no | no |
| [`DiskStore`](disk.md) (SQLite) | yes (a file) | one machine | no |
| [`RedisStore`](redis.md) | yes | yes (networked) | yes (Redis) |
| **`PostgresStore`** | **yes** | **yes (networked)** | yes (Postgres) |

## Layout & parameters

A single table `(ns text, k text, v bytea, primary key (ns, k))` (default name
`swarmstate_kv`); `v` is msgpack bytes.

| Parameter | Default | Description |
| --- | --- | --- |
| `dsn` | `postgresql:///swarmstate` | libpq connection string |
| `conn` | `None` | pass an existing `psycopg` connection instead of a DSN |
| `table` | `"swarmstate_kv"` | table name (validated identifier) |
| `codec` | `"msgpack"` | value serialization (stable, cross-language) |
