# Snapshots & diffs

A **snapshot** is a frozen, read-only copy of the entire store at a moment in time. You
take one, keep working, and can later either compare against it or roll the whole store
back to it - like a savepoint for your agent's state.

What makes this practical is that snapshots are **cheap**. The store is built on a
*persistent data structure*, so taking a snapshot doesn't copy the data - it shares it,
and the store only copies the small pieces that later change (copy-on-write). Taking a
snapshot is **O(1)**: it costs the same whether the store holds ten entries or a million.
(See the [benchmarks](../benchmarks.md) - snapshotting stays flat while a `deepcopy`
grows linearly.)

Because a snapshot is frozen, later writes to the store never change it.

## When you'd use one

- **Undo / rollback** - try something, and revert if it goes wrong.
- **Time-travel & "what-if"** - branch from a known state, explore, discard.
- **Testing** - snapshot a fixture, run a case, restore, repeat.
- **Auditing** - `diff` two snapshots to see exactly what changed.

## Take a snapshot and roll back

```python
import swarmstate as ss

store = ss.Store()
store.set("workflow", "onboarding", {"step": 3})

snap = store.snapshot()                # savepoint

store.set("workflow", "onboarding", {"step": 4})
store.get("workflow", "onboarding")    # -> {"step": 4}

store.restore(snap)                    # go back to the savepoint
store.get("workflow", "onboarding")    # -> {"step": 3}
```

`restore` replaces the whole store with the snapshot's contents - every namespace and
key, not just one.

## What a snapshot knows about itself

```python
snap = store.snapshot()

snap.id          # a monotonic id from the store
snap.timestamp   # when it was taken (seconds since the Unix epoch)
snap.parent      # id of the previous snapshot, or None - snapshots form a chain
snap.size_bytes  # total serialized size of all values at that moment
snap.keys        # the (namespace, key) pairs it contains
```

## See what changed between two snapshots

`later.diff(base)` answers "what happened to get from `base` to `later`?" - as three
lists of `(namespace, key)` pairs:

```python
base = store.snapshot()

store.set("n", "added_key", 1)
store.set("n", "changed_key", 999)     # existed in base with a different value
store.delete("n", "removed_key")       # existed in base

later = store.snapshot()

later.diff(base)
# {
#   "added":   [("n", "added_key")],
#   "removed": [("n", "removed_key")],
#   "changed": [("n", "changed_key")],
# }
```

Diffing is how you persist or transmit **only what changed** between two checkpoints
instead of the whole state - the basis for efficient incremental checkpointing.

## Next

- [Handoff graph](handoff.md) - deterministic routing between agents.
- [LangGraph checkpointer](langgraph.md) - snapshots power whole-DB time-travel across
  every conversation thread at once.
