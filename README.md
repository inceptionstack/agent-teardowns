# Agent Teardowns

Deep architectural teardowns of AI agent frameworks — how they *actually* work under the hood.

These aren't surface-level reviews or marketing summaries. Each teardown is based on **direct source code analysis**, cross-referenced against documentation, with every claim backed by specific file paths and function names.

## Teardowns

| Agent | Latest Version | Last Reviewed | Source Repo |
|-------|---------------|---------------|-------------|
| [OpenClaw](teardowns/openclaw/) | v2 (Apr 2026) | 2026-04-01 | [openclaw/openclaw](https://github.com/openclaw/openclaw) |
| [Hermes Agent](teardowns/hermes-agent/) | v1 (Apr 2026) | 2026-04-01 | [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) |

## Comparisons

| Comparison | Date |
|-----------|------|
| [OpenClaw vs Hermes Agent](comparisons/openclaw-vs-hermes.md) | Apr 2026 |

## How These Are Made

Every teardown follows the [BLUEPRINT.md](BLUEPRINT.md) — a systematic framework for analyzing AI agent architectures. It covers:

- **Tier 1 (Critical):** Agent loop, prompt assembly, memory architecture, context management, security, tool system
- **Tier 2 (High):** Sessions, skills, channels, provider management
- **Tier 3 (Medium):** Sub-agents, scheduling, sandboxing, extensibility
- **Tier 4 (Esoteric):** Node/device systems, research infrastructure, editor integrations

Every section is verified against actual source code, not just documentation. Discrepancies between docs and code are flagged explicitly.

## Starting a New Teardown

Use the [template](teardowns/_template/TEMPLATE.md) to start a new teardown. It follows the blueprint structure with placeholder sections.

## What This Is NOT

- Not a "which agent is best" ranking
- Not a getting-started guide for any framework
- Not affiliated with any of the projects reviewed

## Contributing

Found something wrong or outdated? Open an issue or PR. Teardowns are living documents — agent frameworks evolve fast.

## Author

[Roy Osherove](https://github.com/royosherove) — exploring how AI agent systems work at the architecture level.

---

*All teardowns are based on publicly available open-source code.*
