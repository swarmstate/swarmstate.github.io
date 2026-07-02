# Handoff graph

`HandoffGraph` resolves **deterministic, rule-based transitions** between agents/nodes
— the "which agent is next" decisions that don't need an LLM. Edges carry an optional
condition, and `route()` returns the first matching edge, resolved natively in Rust.

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
