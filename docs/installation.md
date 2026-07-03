# Installation

```bash
pip install swarmstate
# or
uv add swarmstate
```

`swarmstate` ships prebuilt **`cp39-abi3`** wheels for Linux (x86_64/aarch64),
macOS (x86_64/arm64) and Windows (amd64). One wheel serves Python 3.9+ and **no Rust
compiler is required** to install.

## Optional extras

Integrations use lazy imports and are pulled in via extras:

```bash
pip install "swarmstate[langgraph]"   # LangGraph checkpointer adapter
pip install "swarmstate[crewai]"      # CrewAI state/memory adapter
pip install "swarmstate[redis]"       # Redis backend (persistent, networked)
pip install "swarmstate[disk]"        # SQLite disk backend (persistent, no server)
pip install "swarmstate[postgres]"    # Postgres backend (persistent, networked)
pip install "swarmstate[all]"         # everything above

# with uv
uv add "swarmstate[langgraph]"
uv add "swarmstate[all]"
```

## Requirements

| | |
| --- | --- |
| Python | `>= 3.9` |
| Platforms | Linux, macOS, Windows |
| Compiler | not required (prebuilt wheels) |

## From source (development)

Building from source requires a Rust toolchain and [maturin](https://www.maturin.rs/):

```bash
python -m venv .venv && source .venv/bin/activate
pip install maturin pytest
maturin develop --release     # compile the Rust core and install it locally
cargo test                    # Rust core tests
pytest -q                     # Python API tests
```
