# Tutorial — CrewAI: shared, persistent memory

`SwarmStateStorage` implements CrewAI's storage protocol — `save`, `search`, `reset` —
backed by a swarmstate [`Store`](../guide/store.md). Your crew's memory becomes
**durable, snapshot-able and portable**: the same store can hold your LangGraph
checkpoints and be read by other systems.

```bash
pip install swarmstate crewai
# or: uv add swarmstate crewai
```

!!! note "How it's built"
    `SwarmStateStorage` **implements the protocol** rather than importing CrewAI, so it
    is independent of any CrewAI version — you wire it into your crew wherever CrewAI
    accepts a storage object. Search is **lexical** (token-overlap), deterministic and
    dependency-free; for embedding-based semantic recall, use CrewAI's RAG storage.

## Create the storage

```python
import swarmstate as ss
from swarmstate.integrations.crewai import SwarmStateStorage

store = ss.Store()                                   # in-memory
storage = SwarmStateStorage(store, namespace="crew:research")
```

For **persistence across runs/processes**, back it with Redis — same one line:

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

## Wire it into a crew

Pass the storage where your CrewAI setup accepts a memory/storage object (e.g. an
external-memory slot), then run the crew as usual:

```python
from crewai import Agent, Task, Crew          # your normal setup

# ... define agents & tasks, attach `storage` to the crew's memory ...
# result = crew.kickoff()
```

## Portability: one store, many agents

Because the memory lives in a `Store`, anything else can read it — a LangGraph app, a
dashboard, another language:

```python
key = store.keys("crew:research")[0]
store.get("crew:research", key)   # {"value": "...", "metadata": {...}}
```

See [state portability](portability.md) for the full anti-lock-in story, and the
[Redis backend](../guide/redis.md) to make it persistent.
