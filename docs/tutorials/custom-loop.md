# Tutorial — Custom multi-agent loop

You don't need an orchestration framework to get deterministic routing and durable state.
This tutorial builds a **support-ticket triage** system with just swarmstate: a
[`HandoffGraph`](../guide/handoff.md) for routing and a [`Store`](../guide/store.md) for
state. No LLM tokens are spent on the routing decisions.

```bash
pip install swarmstate
```

## 1. Define the routing rules

Edges are evaluated in insertion order; the first matching one wins, and an edge with no
condition is the fallback (added last).

```python
import swarmstate as ss

store = ss.Store()
router = ss.HandoffGraph()

router.add_edge("triage", "billing", when="category == 'billing'")
router.add_edge("triage", "tech",    when="category == 'tech' or 'error' in text")
router.add_edge("triage", "human")   # unconditional fallback
```

## 2. Route tickets and record state

`route()` resolves the next agent natively in Rust; we persist each ticket's assignment
in the `Store`.

```python
def triage(ticket: dict) -> str:
    nxt = router.route("triage", ticket)
    store.set("tickets", ticket["id"], {**ticket, "assigned": nxt})
    return nxt

triage({"id": "T1", "category": "billing", "text": "double charge"})   # -> "billing"
triage({"id": "T2", "category": "other",   "text": "error 500"})       # -> "tech"
triage({"id": "T3", "category": "other",   "text": "how to export?"})  # -> "human"
```

The condition mini-language supports dotted paths, `and/or/not`, comparisons and `in` —
see the [Handoff graph guide](../guide/handoff.md). It is parsed and evaluated in Rust,
never with `eval`, so untrusted rules can't execute code.

## 3. Inspect and snapshot state

```python
store.keys("tickets")                 # -> ['T1', 'T2', 'T3']
store.get("tickets", "T2")["assigned"]  # -> 'tech'

checkpoint = store.snapshot()         # cheap, immutable
store.set("tickets", "T3", {"id": "T3", "assigned": "reopened"})
store.restore(checkpoint)             # roll back
store.get("tickets", "T3")["assigned"]  # -> 'human'
```

## 4. Multi-step workflows

Chain handoffs to model a whole pipeline. Here a ticket flows `intake → review → close`
unless it needs escalation:

```python
flow = ss.HandoffGraph()
flow.add_edge("intake", "escalate", when="priority >= 8")
flow.add_edge("intake", "review")            # default
flow.add_edge("review", "escalate", when="approved == false")
flow.add_edge("review", "close")             # default

def run(ticket):
    node, path = "intake", ["intake"]
    while node not in ("close", "escalate"):
        node = flow.route(node, ticket)
        path.append(node)
    return path

run({"priority": 3, "approved": True})    # -> ['intake', 'review', 'close']
run({"priority": 9})                       # -> ['intake', 'escalate']
run({"priority": 2, "approved": False})   # -> ['intake', 'review', 'escalate']
```

Cycles are rejected by default (`on_cycle="error"`), so a misconfigured pipeline can't
loop forever; pass `HandoffGraph(on_cycle="allow")` if you intend retry loops.

## Recap

- `HandoffGraph.route()` gives **deterministic, microsecond, token-free** routing.
- `Store` gives durable, snapshot-able state that any other system can read.
- Together they're a complete, framework-free orchestration substrate.

Next: [share this state with a LangGraph app](portability.md).
