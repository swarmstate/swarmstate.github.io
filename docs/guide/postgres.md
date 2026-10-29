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
`namespaces`, `clear`, `len(store)`, `snapshot()` / `restore()`, and the batch operations
`set_many`, `get_many`, `delete_many`, `delete_namespace`), plus `close()`. A `set_many`
batch is one transaction: all of it lands, or none of it.

## Async

Every key/value operation has an `a`-prefixed async twin (`aget`, `aset_many`, `akeys`,
...). [`SwarmStateSaver`](langgraph.md#async) awaits them from its async methods, so an
async graph never parks a worker thread on Postgres:

```python
store = PostgresStore("postgresql://user:pass@host/db")
await store.aset("workflow", "onboarding", {"step": 3})
await store.aget("workflow", "onboarding")          # -> {"step": 3}
await store.aclose()                                # the async pool of this event loop
```

A store built from a DSN opens an `AsyncConnectionPool` lazily, one per event loop
(async connections belong to the loop that opened them), or a single async connection
without `psycopg_pool`. A store given only `conn=` / `pool=` has no async support unless
you also pass `apool=`; the saver then falls back to a worker thread.

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

## Connections

Connections come from a `psycopg_pool` pool (`max_size=8` by default), so concurrent
workers are not serialized through one connection — which is the point of choosing Postgres
in the first place:

```python
store = PostgresStore("postgresql://user:pass@host/db", max_size=16)
store = PostgresStore(conn=my_connection)   # opt out: single connection, mutex-guarded
```

An install without `psycopg_pool` falls back to the single-connection path automatically.
Like [`DiskStore`](disk.md), `max_key` is answered from the primary-key index, so the
LangGraph saver resolves "newest checkpoint" with an indexed query and writes no pointer
row.

## Layout & parameters

A single table `(ns text, k text, v bytea, primary key (ns, k))` (default name
`swarmstate_kv`); `v` is msgpack bytes. `ns` and `k` use `COLLATE "C"`, so keys order by
code point like every other backend, and the prefix lookups the LangGraph saver runs on
each `list()` and `delete_thread()` seek the primary key (Postgres 15+) instead of
scanning the table.

Tables created by swarmstate 0.11 or earlier keep the database's collation and keep
working as before. To give one the same behaviour:

```sql
ALTER TABLE swarmstate_kv
  ALTER COLUMN ns TYPE text COLLATE "C",
  ALTER COLUMN k  TYPE text COLLATE "C";
```

| Parameter | Default | Description |
| --- | --- | --- |
| `dsn` | `postgresql:///swarmstate` | libpq connection string |
| `conn` | `None` | pass an existing `psycopg` connection instead of a DSN |
| `pool` | `None` | an existing `psycopg_pool.ConnectionPool` for the sync methods |
| `apool` | `None` | an existing `psycopg_pool.AsyncConnectionPool` for the async ones |
| `max_size` | `8` | size of the pools the store opens itself |
| `table` | `"swarmstate_kv"` | table name (validated identifier) |
| `codec` | `"msgpack"` | value serialization (stable, cross-language) |
