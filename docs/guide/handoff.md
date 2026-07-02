# Handoff graph

In a multi-agent system, *something* has to decide which agent or step runs next. Often
that decision is a simple rule — "billing tickets go to the billing agent" — but it's
easy to end up paying an LLM call to make it. `HandoffGraph` lets you express those rules
as a graph and resolve them **deterministically, in microseconds, with zero tokens**.

You describe the graph as **nodes** connected by **edges**, where each edge may carry a
`when` condition. `route(node, state)` looks at the outgoing edges of `node`, in the
order you added them, and returns the target of the **first** edge whose condition is
true (an edge with no condition always matches, so it acts as a default). It's all
evaluated in Rust — no `eval`, no network, no model.

Reach for it whenever the "next step" is a function of known state rather than a
judgement call; keep using the LLM for the genuinely open-ended decisions.

```python
import swarmstate as ss

g = ss.HandoffGraph()
g.add_edge("triage", "billing", when="category == 'billing'")
g.add_edge("triage", "support", when="category == 'support'")
g.add_edge("triage", "human")                       # unconditional default (added last)

g.route("triage", {"category": "billing"})          # -> "billing"
g.route("triage", {"category": "support"})           # -> "support"
g.route("triage", {"category": "other"})             # -> "human"
```

## Routing semantics

- Outgoing edges are evaluated **in insertion order**; the **first** whose condition holds
  wins. Ordering is fully deterministic.
- An edge with **no** `when` always matches — use it as a fallback/default, added last.
- If no edge matches, `route()` returns `None`.

## The condition mini-language

Conditions are **not** Python — they are parsed and evaluated by a small, bounded
evaluator in Rust (never `eval()`), so untrusted rules can't execute code.

| Feature | Examples |
| --- | --- |
| Literals | `'billing'`, `"x"`, `42`, `3.14`, `true`, `false`, `null` |
| State access | `category`, `user.tier`, `data.user.role` (dotted paths) |
| Comparison | `==`, `!=`, `<`, `<=`, `>`, `>=` |
| Membership | `tag in tags`, `'urgent' in tags`, `'ad' in text` |
| Logic | `and`, `or`, `not`, parentheses |

```python
g.add_edge("n", "escalate",
           when="user.tier == 'gold' or (priority > 3 and 'urgent' in tags)")
```

**Total evaluation.** Type mismatches and missing keys evaluate to `false` rather than
raising, so routing never crashes on unexpected state. Numbers compare across int/float
(`score == 5` matches `5.0`). Invalid *syntax* is rejected eagerly at `add_edge()` time
with a `ValueError`.

## Cycle detection

By default the graph must stay acyclic; an edge that would close a cycle raises:

```python
g = ss.HandoffGraph()             # on_cycle="error" (default)
g.add_edge("a", "b")
g.add_edge("b", "c")
g.add_edge("c", "a")              # ValueError: would create a cycle

ss.HandoffGraph(on_cycle="allow") # opt in to cycles (e.g. retry loops)
```

## Introspection

```python
g.nodes()            # sorted list of all nodes
g.edges("triage")    # [("billing", "category == 'billing'"), ("human", None)]
g.has_node("triage") # True
"triage" in g        # True
len(g)               # number of nodes
g.is_dag()           # whether the graph is currently acyclic
```
