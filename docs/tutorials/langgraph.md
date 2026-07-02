# Tutorial — LangGraph: resumable agents

Build a persistent, resumable LangGraph agent whose checkpoints live in a swarmstate
`Store`, then time-travel across **all** threads with a single store snapshot.

```bash
pip install "swarmstate[langgraph]"
```

## 1. A minimal graph with a checkpointer

We use a tiny deterministic node so the tutorial runs with no API keys — in your app,
replace `respond` with your real LLM/tool node.

```python
import operator
from typing import Annotated, TypedDict

from langgraph.graph import START, END, StateGraph
from swarmstate.integrations.langgraph import SwarmStateSaver


class State(TypedDict):
    messages: Annotated[list, operator.add]   # reducer: appends
    turns: Annotated[int, operator.add]


def respond(state: State):
    last = state["messages"][-1]
    return {"messages": [f"echo: {last}"], "turns": 1}


builder = StateGraph(State)
builder.add_node("respond", respond)
builder.add_edge(START, "respond")
builder.add_edge("respond", END)

saver = SwarmStateSaver()                       # <- was SqliteSaver(...)
graph = builder.compile(checkpointer=saver)
```

## 2. Converse on a thread — state persists

The `thread_id` identifies a conversation; each `invoke` resumes from the last checkpoint.

```python
cfg = {"configurable": {"thread_id": "alice"}}

graph.invoke({"messages": ["hello"], "turns": 0}, cfg)
graph.invoke({"messages": ["again"], "turns": 0}, cfg)

state = graph.get_state(cfg)
print(state.values["turns"])       # -> 2
print(state.values["messages"])    # -> ['hello', 'echo: hello', 'again', 'echo: again']
```

Different `thread_id`s are fully isolated:

```python
graph.invoke({"messages": ["hi"], "turns": 0}, {"configurable": {"thread_id": "bob"}})
graph.get_state({"configurable": {"thread_id": "bob"}}).values["turns"]   # -> 1
```

## 3. Inspect history

`get_state_history` streams every checkpoint on the thread (newest first) — this is
LangGraph's `list()` under the hood:

```python
for snap in graph.get_state_history(cfg):
    print(snap.config["configurable"]["checkpoint_id"], snap.values["turns"])
```

## 4. Time-travel the *whole* checkpoint DB

Because checkpoints live in a `Store`, you can snapshot **every thread at once** in O(1)
and roll back — great for tests, "what-if" branches, or recovery:

```python
snap = saver.store.snapshot()          # freeze all threads

graph.invoke({"messages": ["oops"], "turns": 0}, cfg)
graph.get_state(cfg).values["turns"]   # -> 3

saver.store.restore(snap)              # undo everything since the snapshot
graph.get_state(cfg).values["turns"]   # -> 2
```

## 5. Share one store across graphs / processes

Pass an explicit `Store` to unify checkpoints across multiple compiled graphs (or keep
several independent ones):

```python
import swarmstate as ss

store = ss.Store()
graph_a = builder.compile(checkpointer=SwarmStateSaver(store))
graph_b = builder.compile(checkpointer=SwarmStateSaver(store))
# graph_b.get_state(cfg) sees checkpoints written by graph_a
```

## Async

`ainvoke` / `astream` work out of the box — the saver implements `aput`, `aget_tuple`,
`alist`, and `aput_writes`.

## Recap

- `SwarmStateSaver()` is a **one-line** replacement for `SqliteSaver`/`InMemorySaver`.
- Everything in the LangGraph API (`invoke`, `get_state`, `get_state_history`, resume)
  works unchanged.
- Snapshotting the underlying `Store` gives **atomic time-travel over all threads**.

Next: [state portability across frameworks](portability.md).
