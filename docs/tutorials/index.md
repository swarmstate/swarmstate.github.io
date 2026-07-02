# Tutorials

End-to-end walkthroughs that show swarmstate in real workflows. Each is runnable and
builds on the [Guide](../guide/store.md).

<div class="ss-features" markdown="1">

<div class="ss-card" markdown="1">
### [LangGraph: resumable agents](langgraph.md)
Swap in `SwarmStateSaver`, persist and resume threads, and time-travel across the whole
checkpoint DB with store snapshots.
</div>

<div class="ss-card" markdown="1">
### [Custom multi-agent loop](custom-loop.md)
Build a triage system with deterministic `HandoffGraph` routing and a shared `Store` -
no orchestration framework required.
</div>

<div class="ss-card" markdown="1">
### [State portability across frameworks](portability.md)
Write state from one system, read it from another - the anti-lock-in story, using the
stable msgpack format.
</div>

<div class="ss-card" markdown="1">
### [CrewAI: shared, persistent memory](crewai.md)
Back a CrewAI crew's memory with `SwarmStateStorage` - durable, shareable and portable,
optionally persisted to Redis.
</div>

</div>

!!! tip "Prerequisites"
    ```bash
    pip install swarmstate                 # core (Store, HandoffGraph)
    pip install "swarmstate[langgraph]"    # + LangGraph checkpointer

    # or with uv
    uv add swarmstate
    uv add "swarmstate[langgraph]"
    ```
