# Tutorial — State portability across frameworks

The `Store` serializes to a **stable, language-neutral msgpack format**. State written by
one system is readable by any other — no lock-in. This tutorial writes state from a
LangGraph app and reads it from a plain custom loop (and vice versa).

```bash
pip install "swarmstate[langgraph]"
```

## The shared store

Everything hinges on one idea: **point every system at the same `Store`.**

```python
import swarmstate as ss

store = ss.Store()          # one store, shared by every framework/agent
```

## 1. A LangGraph app writes state

```python
from swarmstate.integrations.langgraph import SwarmStateSaver

saver = SwarmStateSaver(store)          # checkpoints go into `store`

# ... build & compile a graph with checkpointer=saver, then run a thread ...
# graph.invoke({...}, {"configurable": {"thread_id": "order-42"}})
```

Alongside checkpoints, your nodes can also write plain, framework-agnostic facts to the
same store:

```python
store.set("orders", "order-42", {"status": "awaiting_payment", "total": 79.0})
```

## 2. A different system reads it — no LangGraph needed

A separate service, a CrewAI crew, or a custom loop opens the same store (or the same
backend) and reads the state directly:

```python
import swarmstate as ss

store = ss.Store()          # same backing store / backend
order = store.get("orders", "order-42")
# {'status': 'awaiting_payment', 'total': 79.0}
```

Because the format is msgpack (not a Python pickle or a framework-specific blob), the
same bytes are readable from **any language** — not just Python.

## 3. Migrate between frameworks without losing state

Switching orchestration frameworks normally means abandoning accumulated state. With a
shared store, the state simply stays put:

```python
# Framework A wrote it:
store.set("workflow", "onboarding", {"step": 3, "collected": {"email": "a@b.co"}})

# Framework B (a different orchestrator) picks up exactly where A left off:
wf = store.get("workflow", "onboarding")
assert wf["step"] == 3
```

## 4. Audit and diff what changed

Snapshots make it trivial to see what a run changed — useful for debugging cross-system
workflows:

```python
before = store.snapshot()
# ... some framework does work ...
store.set("workflow", "onboarding", {"step": 4, "collected": {"email": "a@b.co"}})
after = store.snapshot()

after.diff(before)
# {'added': [], 'removed': [], 'changed': [('workflow', 'onboarding')]}
```

## Recap

- One `Store`, many frameworks — **no state lock-in**.
- The msgpack format is stable and **cross-language**.
- `snapshot().diff()` gives you an audit trail of exactly what changed.

Next: [use the store as CrewAI memory](crewai.md).
