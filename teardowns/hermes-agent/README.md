# Hermes Agent Teardown

**Source:** [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)
**Language:** Python
**What it is:** Open-source, self-hosted AI agent framework from Nous Research — persistent assistant with a closed learning loop and RL training infrastructure.

## Versions

| Version | Date | Key Findings |
|---------|------|--------------|
| [v1](v1.md) | Apr 2026 | Full teardown — 278K Python lines, 53 tools, 16 gateway platforms, self-improving skills, RL training environment |

## Quick Stats (v1)

- 278,229 lines of Python
- 53 registered tools across 15 toolsets
- 16 gateway platform adapters (9 undocumented)
- 74 bundled skills across 27 categories
- 386 test files
- 6 terminal backends (Local, Docker, SSH, Daytona, Modal, Singularity)
- SQLite WAL + FTS5 for session storage

## What Makes It Interesting

- **Bounded memory** — hard 2,200 char cap forces curation, capacity shown in prompt (`67% -- 1,474/2,200`)
- **Cached/ephemeral prompt split** — deliberate separation enables ~75% Anthropic prompt caching cost reduction
- **Self-improving skills** — agent autonomously creates and refines skills after solving tasks
- **Dual-layer compression** — gateway safety net (85%) + agent compressor (50%) with structured summary templates
- **Research platform** — same codebase runs your assistant AND generates RL training data
- **Smart model routing** — simple messages route to cheap model, complex ones to frontier
- **Path-scoped parallel tool safety** — file tools only parallelize when paths don't overlap
