# Roadmap

`swarmstate` is built in verifiable milestones. Status below reflects what is shipped.

| Milestone | Scope | Status |
| --- | --- | --- |
| **M0** | Scaffolding — maturin project, Rust core, Python API, CI | ✅ done |
| **M1** | Rust store — concurrent KV, msgpack codec, O(1) snapshots, incremental diffs, GIL released | ✅ done |
| **M2** | `HandoffGraph` — conditional DAG, safe condition evaluator (no `eval`), deterministic `route()`, cycle detection | ✅ done |
| **M3** | LangGraph adapter — `SwarmStateSaver` implementing `BaseCheckpointSaver` (drop-in for `SqliteSaver`) | ⏳ next |
| **M4** | Benchmarks — checkpoint latency (p50/p99) and throughput vs `SqliteSaver` and in-memory dict | ⏳ |
| **M5** | CrewAI adapter + optional Redis backend (state portability across frameworks) | ⏳ |
| **M6** | Docs + cross-platform abi3 wheels + PyPI release via Trusted Publishing (OIDC) | 🟡 partial — wheels & PyPI publishing live (`0.1.0`); benchmarks page pending |

## Guiding rules

- **Zero install friction** — `pip install swarmstate` must work with no compiler.
- **Never break the drop-in** — `SwarmStateSaver` stays swappable for `SqliteSaver`.
- **Honest benchmarks** — document hardware, versions and cache state; no inflated numbers.
- **Stable state format** — the msgpack codec won't change incompatibly across minor versions.
- **Safe by construction** — `HandoffGraph` conditions run in a bounded Rust evaluator, never `eval()`.

Track progress on [GitHub](https://github.com/swarmstate/swarmstate).
