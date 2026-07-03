# Observability

The [LangGraph checkpointer](langgraph.md) can report the **latency and outcome** of every
checkpoint operation (`put`, `put_writes`, `get_tuple`) to a metrics sink. It is opt-in and
has **zero overhead when unused**: with no sink (the default) the saver never even reads the
clock.

```python
from swarmstate.integrations.langgraph import SwarmStateSaver
from swarmstate.observability import InMemoryMetrics

metrics = InMemoryMetrics()
saver = SwarmStateSaver(metrics=metrics)

graph = builder.compile(checkpointer=saver)
graph.invoke(inputs, {"configurable": {"thread_id": "t1"}})

metrics.summary()
# {'put': {'count': 3, 'errors': 0, 'mean_ms': 0.007, 'p50_ms': 0.006, 'p99_ms': 0.011},
#  'get_tuple': {'count': 1, ...}}
```

## Sinks

A sink is anything with a `record` method:

```python
def record(self, op: str, duration_s: float, *, thread_id: str, ok: bool) -> None: ...
```

Three ship with swarmstate:

| Sink | Use |
| --- | --- |
| `NullMetrics` | the explicit no-op (same as passing no sink) |
| `InMemoryMetrics` | accumulate counts + latency percentiles in process (tests, notebooks, quick profiling); thread-safe, with `.summary()` and `.reset()` |
| `OpenTelemetryMetrics` | emit an OpenTelemetry histogram + counter |

Bring your own by implementing `record` (for example, push straight to StatsD or Prometheus).

## OpenTelemetry

```bash
pip install "swarmstate[otel]"
# or: uv add "swarmstate[otel]"
```

```python
from swarmstate.observability import OpenTelemetryMetrics

saver = SwarmStateSaver(metrics=OpenTelemetryMetrics())
```

It records two instruments, tagged with `op` and `ok`:

- `swarmstate.checkpoint.duration` - a histogram in milliseconds.
- `swarmstate.checkpoint.operations` - a counter.

!!! info "Cardinality"
    `thread_id` is passed to your sink but is **not** used as an OpenTelemetry attribute,
    to keep metric cardinality bounded. Use a custom sink if you need per-thread breakdowns.
