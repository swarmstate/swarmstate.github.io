# Snapshots & diffs

A **snapshot** is an immutable, point-in-time view of a `Store`. Snapshots are cheap:
the store is backed by a persistent data structure, so taking a snapshot is an **O(1)**
operation that shares structure with the live store (copy-on-write). Later mutations to
the store never affect an existing snapshot.

## Taking and restoring

```python
import swarmstate as ss

store = ss.Store()
store.set("workflow", "onboarding", {"step": 3})

snap = store.snapshot()          # capture

store.set("workflow", "onboarding", {"step": 4})
store.get("workflow", "onboarding")   # -> {"step": 4}

store.restore(snap)              # roll back
store.get("workflow", "onboarding")   # -> {"step": 3}
```

## Snapshot metadata

```python
snap = store.snapshot()

snap.id          # monotonic id assigned by the store
snap.timestamp   # seconds since the Unix epoch
snap.parent      # id of the previous snapshot (or None), for incremental chains
snap.size_bytes  # total serialized size of all values
snap.keys        # list of (namespace, key) pairs present
```

## Incremental diffs

`Snapshot.diff(base)` reports how to go from `base` to the snapshot it is called on:

```python
base = store.snapshot()

store.set("n", "added_key", 1)
store.set("n", "changed_key", 999)   # assume it existed in base
store.delete("n", "removed_key")     # assume it existed in base

now = store.snapshot()

now.diff(base)
# {
#   "added":   [("n", "added_key")],
#   "removed": [("n", "removed_key")],
#   "changed": [("n", "changed_key")],
# }
```

This makes it cheap to persist only what changed between two checkpoints — the
foundation for the incremental checkpointing used by the LangGraph adapter (M3).
