# OpenClaw Teardown

**Source:** [openclaw/openclaw](https://github.com/openclaw/openclaw)
**Language:** TypeScript (Node.js 22.12+)
**What it is:** Open-source, self-hosted AI agent framework — multi-channel messaging, sandboxed tool execution, persistent memory, 90 extension packages.

## Versions

| Version | Date | Codebase Version | Key Findings |
|---------|------|-----------------|--------------|
| [v1](v1.md) | Feb 2026 | ~early 2026 | Initial teardown — 7 channels, 12 tools, basic architecture |
| [v2](v2.md) | Apr 2026 | 2026.4.1 | Major update — 30+ channels, 22+ tools, MCP server, OpenAI HTTP API, tool loop detection, 90 extensions, 53 skills |

## Quick Stats (v2)

- ~8,700 TypeScript source files
- 90 extension packages
- 53 bundled skills
- 30+ messaging platforms
- 25+ LLM providers
- 5,355-line CHANGELOG
- 149 gateway WebSocket methods
- 20+ hook types
- 10-layer security model with audit CLI

## What Makes It Interesting

- **Single-daemon architecture** — one Gateway process owns all state and messaging sessions
- **Lane-based command queue** with generation tracking for crash recovery
- **File-based memory** (plain Markdown) with hybrid vector+BM25 search
- **Tool loop detection** — 4 detector types prevent stuck agent loops
- **MCP server mode** — any MCP client can control OpenClaw conversations
- **Plugin-first extensibility** — 90 extensions, 20+ hook types, everything is pluggable
