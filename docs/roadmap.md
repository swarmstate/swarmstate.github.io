# Roadmap

`swarmstate` is built in verifiable milestones. Status below reflects what is shipped.

| Milestone | Scope | Status |
| --- | --- | --- |
| **M0** | Scaffolding — maturin project, Rust core, Python API, CI | ✅ done |
| **M1** | Rust store — concurrent KV, msgpack codec, O(1) snapshots, incremental diffs, GIL released | ✅ done |
| **M2** | `HandoffGraph` — conditional DAG, safe condition evaluator (no `eval`), deterministic `route()`, cycle detection | ✅ done |
| **M3** | LangGraph adapter — `SwarmStateSaver` implementing `BaseCheckpointSaver` (drop-in for `SqliteSaver`) | ✅ done |
| **M4** | Benchmarks — checkpoint latency (p50/p99) and throughput vs `SqliteSaver` and in-memory dict | ✅ done |
| **M5** | CrewAI adapter + optional Redis backend (state portability across frameworks) | ✅ done |
| **M6** | Docs + cross-platform abi3 wheels + PyPI release via Trusted Publishing (OIDC) | ✅ done |

## Guiding rules

- **Zero install friction** — `pip install swarmstate` must work with no compiler.
- **Never break the drop-in** — `SwarmStateSaver` stays swappable for `SqliteSaver`.
- **Reproducible benchmarks** — document hardware, versions and cache state.
- **Stable state format** — the msgpack codec won't change incompatibly across minor versions.
- **Safe by construction** — `HandoffGraph` conditions run in a bounded Rust evaluator, never `eval()`.

Track progress on [GitHub](https://github.com/swarmstate/swarmstate).
