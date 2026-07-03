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
### [CrewAI: shared, portable memory](crewai.md)
Durable, portable keyword recall around a CrewAI crew with `SwarmStateStorage`, in a
store shared with your other agents (optionally on Redis/disk).
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

!!! note "Runnable examples"
    Prefer to read code? The repo ships self-contained, offline, deterministic scripts in
    [`examples/`](https://github.com/swarmstate/swarmstate/tree/main/examples):

    - [`support_triage.py`](https://github.com/swarmstate/swarmstate/blob/main/examples/support_triage.py)
      - a LangGraph workflow combining `HandoffGraph` routing, `SwarmStateSaver`
      checkpointing and snapshot/restore time-travel.
    - [`state_portability.py`](https://github.com/swarmstate/swarmstate/blob/main/examples/state_portability.py)
      - state as standard msgpack, read back and cross-checked.
