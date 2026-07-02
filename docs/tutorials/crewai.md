# Tutorial — CrewAI: shared, persistent memory

!!! info "First-class adapter is coming in M5"
    A dedicated CrewAI adapter (mirroring the [LangGraph one](langgraph.md)) lands in
    **M5** — see the [Roadmap](../roadmap.md). Until then, this tutorial shows the pattern
    that works **today**: use a swarmstate [`Store`](../guide/store.md) as durable,
    shareable state around your crew, using only the stable public `Store` API.

```bash
pip install swarmstate crewai
```

## The idea

CrewAI coordinates agents and tasks; swarmstate gives that crew **persistent,
snapshot-able, framework-agnostic state** that survives process restarts and can be read
by other systems. You wrap task execution with a couple of `Store` calls.

## 1. A store keyed by run

```python
import swarmstate as ss

store = ss.Store()          # or share one across many crews

RUN = "research-2026-07-02"
```

## 2. Persist each task's output

Whatever a CrewAI task returns, record it in the store under the run. This is plain
`Store.set` — nothing CrewAI-specific:

```python
def remember(task_name: str, output) -> None:
    store.set(RUN, task_name, {"output": str(output), "done": True})

def recall(task_name: str):
    return store.get(RUN, task_name)
```

Wire these into your crew — e.g. in a task callback or right after `crew.kickoff()`:

```python
from crewai import Agent, Task, Crew          # your normal CrewAI setup

# ... define agents and tasks as usual ...
# result = crew.kickoff()
# remember("final_report", result)
```

Now `recall("final_report")` returns the output on the next run, even in a fresh process,
and **any other system** (a LangGraph app, a dashboard, another language) can read the
same keys — see [state portability](portability.md).

## 3. Resume: skip tasks already completed

Because state is durable, you can make a crew **idempotent** — skip work that's already
done:

```python
def run_task(task_name, fn):
    cached = recall(task_name)
    if cached and cached["done"]:
        return cached["output"]          # resume: reuse prior result
    output = fn()                        # run the CrewAI task
    remember(task_name, output)
    return output
```

## 4. Snapshot a crew run for "what-if" branches

```python
checkpoint = store.snapshot()
# ... let the crew explore one strategy ...
store.restore(checkpoint)               # discard it, try another
```

## 5. Deterministic hand-offs between agents (optional)

For rule-based "which agent next" decisions inside a crew, drop in a
[`HandoffGraph`](../guide/handoff.md) instead of spending tokens on the choice:

```python
router = ss.HandoffGraph()
router.add_edge("researcher", "writer",   when="findings_ready == true")
router.add_edge("researcher", "reviewer", when="needs_review == true")
next_agent = router.route("researcher", {"findings_ready": True})   # -> "writer"
```

## Recap

- Today: use `Store` as durable, portable memory + `HandoffGraph` for routing around any
  CrewAI crew — pure public API, nothing to invent.
- Soon (**M5**): a first-class CrewAI adapter so this is wired in automatically.
