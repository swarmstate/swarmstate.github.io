# Tutorial - CrewAI: shared, portable memory

`SwarmStateStorage` is a small, dependency-free **keyword** memory store (`save`,
`search`, `reset`) backed by a swarmstate [`Store`](../guide/store.md). Its value is
**portability and durability**: crew memories live in the same store as your LangGraph
checkpoints, can be snapshotted, and can be read by any other system.

```bash
pip install swarmstate crewai
# or: uv add swarmstate crewai
```

!!! warning "This is not CrewAI's built-in memory backend"
    As of CrewAI 1.x, the native memory `StorageBackend` is **embedding-based** (it
    stores `MemoryRecord`s and searches by vector similarity) - a vector store, which
    swarmstate is not. `SwarmStateStorage` is a lightweight **lexical** alternative you
    wire in yourself (e.g. from a task callback or your own loop) when you want durable,
    portable, dependency-free recall. For semantic RAG recall, use CrewAI's own storage.
    *(Verified against crewai 1.15.)*

## Create the storage

```python
import swarmstate as ss
from swarmstate.integrations.crewai import SwarmStateStorage

store = ss.Store()                                   # in-memory
storage = SwarmStateStorage(store, namespace="crew:research")
```

For **persistence across runs/processes**, back it with Redis - same one line:

```python
from swarmstate.backends.redis import RedisStore

storage = SwarmStateStorage(RedisStore("redis://localhost:6379/0"), namespace="crew:research")
```

## Save and recall memory

```python
storage.save("The Q2 churn rate was 4.1%", {"agent": "analyst"})
storage.save("Top objection in calls: pricing", {"agent": "sales"})

storage.search("churn rate", limit=3)
# [{"context": "The Q2 churn rate was 4.1%", "metadata": {"agent": "analyst"}, "score": 1.0}]

storage.reset()          # clear this namespace
```

`search` returns `{"context", "metadata", "score"}` entries, ranked by token overlap
then recency.

## Use it around a crew

You drive it yourself - typically from a task callback or right after `kickoff()` -
rather than plugging it into CrewAI's native (embedding-based) memory slot:

```python
from crewai import Agent, Task, Crew          # your normal setup

# ... define agents & tasks ...
result = crew.kickoff()
storage.save(str(result), {"run": "research", "ts": "2026-07-03"})

# next run, recall prior findings before kicking off again
prior = storage.search("research findings", limit=5)
```

This keeps the useful part - **durable, portable, snapshot-able recall in a shared
store** - without pretending to be a vector database.

## Portability: one store, many agents

Because the memory lives in a `Store`, anything else can read it - a LangGraph app, a
dashboard, another language:

```python
key = store.keys("crew:research")[0]
store.get("crew:research", key)   # {"value": "...", "metadata": {...}}
```

See [state portability](portability.md) for the full anti-lock-in story, and the
[Redis backend](../guide/redis.md) to make it persistent.
