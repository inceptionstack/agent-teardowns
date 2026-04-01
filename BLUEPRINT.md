### AI Agent Framework: Technical Review Guidelines — v1.0 (April 2026)
### A Systematic Framework for Analyzing AI Agent Architectures
---

## Purpose

This document provides a repeatable framework for producing deep-dive architecture reviews of AI agent frameworks (OpenClaw, Hermes, and similar systems). It was derived from patterns observed across multiple agent codebases and is designed to ensure consistent, thorough coverage.

Use this as a checklist when reviewing a new agent framework's source code.

---

## Review Principles

1. **Source over docs.** Documentation lags behind code. Always verify claims against the actual implementation. Read the actual source files, not just README/docs.
2. **Flag doc-source conflicts.** When documentation claims something the code contradicts (or vice versa), flag it explicitly. This could be a red flag (misleading docs), an undocumented feature (code ahead of docs), or a human mistake (stale docs). All three matter — surface them in the review.
3. **Count things.** Lines of code, number of tools, number of extensions, number of tests — concrete numbers ground the review.
4. **Name the files.** Every architectural claim should reference a specific file path and ideally a function/class name.
5. **Find the undocumented.** The most interesting architectural decisions are often not in the README. Grep the source for features the docs never mention.
6. **Compare to the pattern.** Use this framework to identify both standard patterns and novel approaches.

---

## Source Code vs. Documentation Protocol

**Every section of the review must be verified against actual source code, not just documentation.**

### Mandatory Steps

0. **Record the source.** Every teardown must include a **Source Metadata** header at the top:

```markdown
### Source Metadata
| Field | Value |
|-------|-------|
| **Repo** | `github.com/org/repo` |
| **Commit** | `abc1234` (or tag/release) |
| **Branch** | `main` |
| **Clone date** | YYYY-MM-DD |
| **Version** | (from package.json, pyproject.toml, etc.) |
```

This is non-negotiable. Without it, the review is not reproducible.

1. **Clone the repo.** Always work from a fresh clone, not cached docs or memory.
2. **Read key implementation files.** For each major subsystem, open the actual source file and trace the logic.
3. **Cross-reference doc claims.** For each documented feature, grep the source to confirm it exists and works as described.
4. **Document discrepancies.** Use this format when docs and source disagree:

```
> **Doc-Source Conflict:** The docs claim X supports Y, but the source at `path/to/file.ts:123`
> shows Z instead. This appears to be [stale docs / undocumented change / potential bug].
```

5. **Mark undocumented features.** When source reveals capabilities not mentioned in any docs, mark them:

```
> **Undocumented:** `file.ts` implements [feature] which is not mentioned in any documentation.
```

6. **Mark doc-only features.** When docs describe a feature but the source doesn't implement it:

```
> **Doc-Only (Not Implemented):** The docs describe [feature] but no corresponding implementation
> was found in the source. This may be planned, removed, or behind a feature flag.
```

### Why This Matters

- Agent frameworks evolve fast — docs routinely lag by weeks or months
- Undocumented features can be the most architecturally interesting
- Doc-source conflicts may indicate half-finished migrations
- Security-relevant discrepancies (e.g., "docs say sandboxed but code runs on host") are critical findings
- The review itself becomes a source of truth that's more current than the project's own docs

---

## 1. Project Vitals (Always First)

Start every teardown with source metadata and project vitals:

| Item | What to Find | Importance |
|------|-------------|------------|
| **Repo URL** | Full GitHub/GitLab URL | **Critical** |
| **Commit / tag** | Exact commit hash or release tag reviewed | **Critical** |
| **Branch** | Branch name (usually `main`) | **Critical** |
| **Clone date** | When the source was cloned for this review | **Critical** |
| Language & runtime | Primary language, runtime version requirements | **Critical** |
| Version | Current version from manifest file | **High** |
| Repo structure | Monorepo? Workspaces? Extension architecture? | **High** |
| Package manager & build | npm/pnpm/pip/cargo, build tool, bundler | Medium |
| License | Open source license type | Medium |
| Key dependencies | Major external deps (LLM SDKs, DB drivers, messaging libs) | Medium |

**Where to look:** `package.json`, `pyproject.toml`, `Cargo.toml`, root config files, CI configs.

**Note:** Lines-of-code counts are optional context, not required. Focus on architectural findings over metrics.

---

## 2. Core Architecture (The Big Picture)

### 2.1 Process Model — CRITICAL

Every agent framework has a process model. Identify it:

| Pattern | Description | Examples |
|---------|-------------|---------|
| **Single daemon** | One long-lived process owns all state | OpenClaw Gateway |
| **CLI-spawned** | Fresh process per invocation | Hermes CLI mode |
| **Hybrid** | Daemon + CLI that connects to it | Both (OpenClaw CLI→Gateway, Hermes CLI+Gateway) |

**Key questions:**
- Is there a single writer per session? (Prevents corruption)
- How do multiple entry points share state? (SQLite WAL, file locks, IPC)
- What happens if the process crashes? (Recovery model)

### 2.2 Entry Points

List ALL ways the agent can be invoked:
- CLI (interactive terminal)
- Daemon/service (background)
- Gateway (messaging platforms)
- Editor integration (ACP, LSP, stdio)
- API server (OpenAI-compatible, REST)
- MCP server (Model Context Protocol)
- Cron/scheduler (self-invoked)
- Webhooks (event-driven)

### 2.3 Architecture Diagram

Draw the standard three-tier:
```
Entry Points → Agent Runtime → LLM Providers
                    ↓
            Tool Dispatch + Memory + Sessions
```

---

## 3. The Agent Loop — CRITICAL

This is the heart of any agent framework. Trace the full lifecycle of a single user message.

### 3.1 Message Intake & Routing

| Component | What to Document |
|-----------|-----------------|
| Message normalization | How platform-specific messages become internal envelopes |
| Session resolution | How inbound messages map to sessions (key format, routing rules) |
| Queue/serialization | How concurrent messages are serialized (FIFO? lanes? priorities?) |
| Queue modes | What happens when a new message arrives during an active run? |

**Expected patterns:**
- Lane-based queuing (per-session serial, global concurrent)
- Queue modes: collect/coalesce, steer/interrupt, followup/wait
- Idempotency keys for safe retries

### 3.2 Prompt Assembly — CRITICAL

How the system prompt is built. This is where frameworks diverge most.

**Standard layers to look for:**
| Layer | Description | Typical Priority |
|-------|-------------|-----------------|
| Identity/persona | SOUL.md, agent identity | Top (stable) |
| Safety guardrails | Behavioral constraints | Top (stable) |
| Tool schemas | Available tools + JSON schemas | Top (stable) |
| Skills index | Available skills metadata | Top (stable) |
| Memory snapshot | Persistent memory content | Mid (stable per session) |
| User profile | User preferences/context | Mid (stable per session) |
| Context files | AGENTS.md, project-level instructions | Mid (stable per session) |
| Runtime metadata | Date/time, model, OS, host info | Mid (semi-stable) |
| Ephemeral context | Per-turn injections, recall results | Bottom (volatile) |

**Key questions:**
- Is the prompt split into cached vs. ephemeral layers? (Cost optimization)
- Are bootstrap files truncated? At what limit?
- How is time handled? (Timezone-only for cache stability, or full timestamp?)
- Are tool schemas counted toward context? (They should be)
- Is there prompt caching support? (Anthropic cache_control breakpoints)

### 3.3 Model Inference

| Feature | What to Document |
|---------|-----------------|
| Provider resolution | How model + auth are selected |
| API modes | Chat completions vs. Messages API vs. Responses API |
| Streaming | How deltas are emitted (text, tool calls, thinking) |
| Timeout | Default timeout, configurability |
| Fallback | What happens when the primary model fails? |
| Retry logic | How API errors are handled |
| Budget tracking | Iteration/token/cost budgets |

### 3.4 Tool Execution

| Feature | What to Document |
|---------|-----------------|
| Tool registry | How tools are registered (declarative, self-registering, plugin) |
| Tool dispatch | Sync vs. async, serial vs. parallel |
| Parallel safety | How parallel tool execution avoids conflicts |
| Error handling | How tool errors are presented to the model |
| Tool policy | Allow/deny system, layered policies |
| Agent-loop tools | Tools that need agent-level state (memory, delegation, search) |
| Dangerous command detection | Pattern-based, LLM-assisted, or both? |
| Approval flow | How destructive commands are gated |

**Parallel tool execution deserves special attention** — look for:
- Path-scoped safety analysis (file tools check for overlapping paths)
- Destructive command detection for sequential fallback
- Never-parallel tools (interactive/clarification tools)
- Max concurrent workers

---

## 4. Memory Architecture — CRITICAL

How a stateless model maintains persistent knowledge across sessions.

### 4.1 The Memory Stack

Every serious agent framework implements multiple memory layers:

| Layer | Characteristics | Standard? |
|-------|----------------|-----------|
| **Context window** | Current conversation, ephemeral | Universal |
| **Session transcript** | Full conversation history (JSONL/SQLite) | Universal |
| **Daily logs** | Append-only daily notes | Common |
| **Long-term memory** | Curated persistent notes (MEMORY.md) | Common |
| **User profile** | User preferences/identity (USER.md) | Common |
| **Vector search** | Semantic retrieval over memory corpus | Advanced |
| **Cross-session search** | FTS/keyword search over past conversations | Advanced |
| **External memory** | Third-party memory services (Honcho, etc.) | Rare |

### 4.2 Key Memory Questions

- **Bounded or unbounded?** Hard caps force curation (Hermes: 2,200 chars). Unbounded risks prompt bloat (OpenClaw: no cap on MEMORY.md, but truncated at injection).
- **Frozen or live?** Is the memory snapshot frozen at session start (cache-friendly) or updated mid-session?
- **Capacity awareness?** Does the model know how full memory is? (Hermes shows `67% — 1,474/2,200 chars`)
- **Pre-compaction flush?** Is there a mechanism to save important context before compaction erases it?
- **Security scanning?** Are memory writes scanned for prompt injection?

### 4.3 Vector Search Details

| Component | What to Document |
|-----------|-----------------|
| Embedding providers | Which models? Local vs. remote? Fallback chain? |
| Chunking | Token size, overlap |
| Storage | SQLite, LanceDB, Pinecone, etc. |
| Search type | Pure vector, pure keyword, or hybrid? |
| Hybrid weights | Vector vs. BM25 ratio |
| Reindex triggers | When does the index rebuild? |
| Advanced features | MMR diversity, temporal decay, multimodal |

---

## 5. Session Management — HIGH

### 5.1 Session Lifecycle

| Phase | What to Document |
|-------|-----------------|
| Creation | On first message, explicit, or both? |
| Routing | Session key format, dmScope rules |
| Active use | How messages append, how context grows |
| Compaction | When/how context is compressed |
| Expiry | Daily reset, idle timeout, manual reset |
| Persistence | JSONL files, SQLite, or both? |
| Cleanup | Pruning old sessions, storage limits |

### 5.2 Context Window Management — CRITICAL

| Feature | What to Document |
|---------|-----------------|
| Compaction trigger | Token threshold (% of context window) |
| Compaction method | Freeform summary vs. structured template |
| Summary format | If structured: what sections? (Goal/Progress/Decisions/Files/Next) |
| Protected regions | First N messages, last N messages, tool call groups |
| Tool result pruning | Separate from compaction? Threshold? |
| Iterative compression | How subsequent compressions update prior summaries |
| Dual-layer compression | Safety net at higher threshold? |
| Pressure warnings | Does the model get warned before compaction? |

**Structured summary templates** (Goal/Constraints/Progress/Decisions/Files/Next Steps) produce dramatically better compressions than freeform. Note whether the framework uses them.

---

## 6. Skills System — HIGH

### 6.1 Standard Pattern

Most frameworks use lazy-loaded skills:
1. **Metadata injected** — compact list in system prompt (name + description + path)
2. **Full content on demand** — model reads the SKILL.md when task matches
3. **Gating** — skills filtered by platform, required binaries, env vars, config

### 6.2 Key Questions

- How are skills discovered? (Workspace → managed → bundled priority)
- What's the skill format? (YAML frontmatter + Markdown body)
- Are skills hot-reloaded on filesystem changes?
- Can the agent create new skills autonomously? (Hermes: yes, closed learning loop)
- Can skills self-improve during use?
- Are external skills security-scanned before installation?
- Conditional activation? (Show/hide based on available tools)

---

## 7. Channel / Messaging — HIGH

### 7.1 Standard Bridge Pattern

```
Platform SDK → Channel Bridge → Normalized Envelope → Session Router → Agent
```

### 7.2 Channel Inventory

List ALL supported platforms. Common ones:
- Telegram, Discord, Slack, WhatsApp, Signal (standard)
- iMessage, Matrix, MS Teams, Google Chat (extended)
- Email, SMS, IRC, Nostr (specialized)
- WebChat, API Server (self-hosted)
- Webhooks (event-driven)

### 7.3 Key Questions

- Is the gateway the single holder of messaging sessions? (Important for WhatsApp)
- How many platforms does it actually support vs. claim?
- Are there undocumented platform adapters?
- Can it expose itself as an OpenAI-compatible API?
- Webhook support? (Generic event-driven automation)

---

## 8. Sub-Agent Architecture — MEDIUM

| Feature | What to Document |
|---------|-----------------|
| Isolation | Own session, own context window? |
| Nesting | Can sub-agents spawn sub-agents? (Usually no) |
| Tool restrictions | What tools are denied? |
| Result delivery | How results flow back to parent |
| Budget sharing | Shared iteration budget with parent? |
| Model override | Can sub-agents use cheaper models? |
| Concurrency | Max concurrent sub-agents |
| Auto-cleanup | Session archival after timeout |

---

## 9. Scheduling & Proactive Behavior — MEDIUM

### 9.1 Heartbeat

- Periodic agent turns in main session
- Interval (typically 15-60 min)
- Suppression protocol (HEARTBEAT_OK = nothing to report)
- HEARTBEAT.md checklist file

### 9.2 Cron

- Schedule types: one-shot, recurring, cron expression
- Payload types: inject into session vs. isolated session
- Delivery: where results go
- Recursion guards: can cron jobs create more cron jobs? (Should not)
- Locking: prevent overlapping executions

---

## 10. Security Architecture — CRITICAL

### 10.1 Standard Security Layers

Every agent framework should have these (in order of evaluation):

| Layer | Purpose | Standard? |
|-------|---------|-----------|
| Gateway authentication | Token/password/device auth | Universal |
| Channel allowlists | Who can message the agent | Universal |
| Device pairing & trust | Approved devices | Common |
| Tool policy (allow/deny) | Which tools are available | Universal |
| Exec approvals | Gate destructive commands | Universal |
| Sandbox (container isolation) | Blast radius reduction | Common |
| Send policy (outbound gates) | Control what the agent can send | Advanced |
| Content scanning | Prompt injection in context files | Advanced |
| Memory write scanning | Injection via memory entries | Advanced |
| Path traversal protection | SSRF, directory traversal | Advanced |
| Security audit CLI | Automated security posture check | Advanced |

### 10.2 Key Questions

- What's the trust model? (Single operator vs. multi-tenant)
- Are system prompt guardrails advisory or enforced?
- Is there a YOLO mode? How does it work?
- Are there smart/LLM-assisted approval decisions?
- Can security settings be audited programmatically?

---

## 11. Sandboxing — MEDIUM

### 11.1 Standard Dimensions

| Dimension | Options |
|-----------|---------|
| **Mode** | off / non-main / all |
| **Scope** | per-session / per-agent / shared |
| **Workspace access** | none / read-only / read-write |
| **Backend** | Docker / SSH / managed cloud |
| **Escape hatches** | Elevated tools, host bypass |

### 11.2 Key Questions

- Can the browser also be sandboxed? (Separate container)
- Is there network isolation?
- What's the default? (Should be safe)

---

## 12. Extensibility (Hooks & Plugins) — MEDIUM

### 12.1 Hook Lifecycle Events

Standard hook points to look for:
- **Gateway**: startup, stop
- **Agent**: before-start, end, before-dispatch
- **Session**: start, end, reset
- **Message**: received, sending, sent, before-write
- **Tool**: before-call, after-call, result-persist
- **Compaction**: before, after
- **Model**: before-resolve (model override)
- **Subagent**: spawning, spawned, ended, delivery-target

### 12.2 Plugin Capabilities

What can plugins register?
- New tools
- New RPC methods / HTTP handlers
- New CLI commands
- Background services
- Skills
- Auto-reply commands
- Hook handlers

### 12.3 Key Questions

- Are hooks discovered from multiple directories? (Workspace → managed → bundled)
- Can workspace hooks override bundled hooks? (Security concern)
- Do plugins run in-process or sandboxed?
- Are plugin installs security-scanned?

---

## 13. Provider & Model Management — MEDIUM

| Feature | What to Document |
|---------|-----------------|
| Supported providers | List all (Anthropic, OpenAI, Bedrock, Vertex, Ollama, etc.) |
| Provider resolution | Precedence: explicit → config → env → default |
| API key scoping | Are keys leaked to wrong endpoints? |
| Credential rotation | Multi-key pools, automatic failover |
| Smart model routing | Cheap model for simple messages, strong for complex |
| Auxiliary model routing | Separate model for summaries, embeddings, search |
| Prompt caching | Provider-specific optimizations |
| Cost tracking | Per-session, per-model cost accounting |

---

## 14. Node / Device System — ESOTERIC

Physical world integration (not all frameworks have this):
- Device pairing protocol
- Capabilities: camera, screen, location, system.run, canvas
- WebSocket-based device communication
- Use cases: surveillance, automation, dashboards

---

## 15. Research Infrastructure — ESOTERIC

Only some frameworks include this:
- RL training environment
- Benchmark suites
- Trajectory generation and compression
- Tool call parsers for different model families
- Training data pipelines

---

## 16. Source Code Deep Dive Checklist

After covering the standard sections, dig into the source for undocumented features:

### Things to grep for:
- `TODO`, `FIXME`, `HACK`, `EXPERIMENTAL` — find rough edges
- `deprecated`, `legacy`, `compat` — find evolution traces
- Feature flags, environment variable gates
- Hidden CLI commands or debug endpoints
- Undocumented tool registrations
- Test files that reveal unfinished features

### Patterns that reveal architecture:
- **Largest files** — often the most complex subsystems
- **Most-imported modules** — the real foundations
- **Error handling patterns** — reveals failure modes
- **Configuration schema** — the full surface area of behavior
- **Type definitions** — the canonical data model

### Common undocumented features to hunt for:
- Parallel tool execution safety analysis
- Credential pools / multi-key rotation
- Smart model routing (cheap vs. strong)
- Background memory/skill review loops
- Content security scanning (beyond what docs say)
- Checkpoint/snapshot systems
- Tool call deduplication and repair
- Budget pressure systems
- Deterministic IDs for retry safety

---

## 17. Review Output Format

Each deep dive should follow this structure:

```
1. Executive Summary (what it is, what makes it unique)
2. What's New (for v2+ reviews — delta from previous version)
3. Doc-Source Discrepancy Log (conflicts, undocumented features, doc-only claims)
4. High-Level Architecture Diagram
5. The Agent Loop (message → action lifecycle)
6. Prompt Assembly (system prompt construction)
7. Memory Architecture (persistence layers)
8. Session Management (lifecycle, compaction)
9. Tool System (registry, dispatch, policy)
10. Skills System (discovery, loading, gating)
11. Channel / Messaging (bridges, platforms)
12. Sub-Agent Architecture
13. Scheduling & Proactive Behavior
14. Security Architecture
15. Sandboxing
16. Hooks & Plugins
17. Provider & Model Management
18. [Framework-Specific Sections]
19. Source Code Deep Dive (undocumented findings)
20. Complete Tool Inventory
21. Summary Table (concept → implementation → why it matters)
```

### Doc-Source Discrepancy Log Format

This section should appear early in the review (section 3) so readers know upfront where docs can't be trusted:

```markdown
## Doc-Source Discrepancy Log

| # | Type | Area | Description | Source File | Severity |
|---|------|------|-------------|-------------|----------|
| 1 | Conflict | Memory | Docs say 20K char limit, source shows 30K | memory.ts:45 | Low |
| 2 | Undocumented | Tools | `mcp_bridge` tool exists in source, not in docs | tools/mcp.ts | Medium |
| 3 | Doc-Only | Sandbox | "Kubernetes backend" mentioned in docs, not implemented | — | High |
| 4 | Stale | Channels | Docs list 6 channels, source has 25 adapters | extensions/ | Medium |
```

**Severity guide:**
- **High**: Security-relevant, or feature claimed but not implemented
- **Medium**: Significant capability missing from docs, or misleading description
- **Low**: Minor numeric discrepancy, outdated count, cosmetic

For **v2+ reviews** (updates to existing deep dives), also add:
- **What's New** section at the top listing all changes since last review
- Version number and date in the title
- Delta-focused analysis (what changed, what's new, what was removed)

---

## 18. Importance Tiers

### Tier 1 — CRITICAL (must cover in depth)
- Agent loop (message lifecycle)
- Prompt assembly (system prompt construction)
- Memory architecture (persistence + recall)
- Context window management (compaction)
- Security model (all layers)
- Tool system (registry, dispatch, policy, approvals)

### Tier 2 — HIGH (cover thoroughly)
- Session management (routing, lifecycle, persistence)
- Skills system (discovery, loading, creation)
- Channel/messaging (bridges, platforms supported)
- Provider management (resolution, failback, cost)

### Tier 3 — MEDIUM (cover the essentials)
- Sub-agent architecture
- Scheduling (heartbeat, cron)
- Sandboxing (modes, backends)
- Extensibility (hooks, plugins)

### Tier 4 — ESOTERIC (cover if present, note if absent)
- Node/device system
- Research infrastructure (RL, benchmarks)
- Editor integration (ACP, LSP)
- MCP server/client support
- Anti-detection browser
- Mixture of Agents
- Training data pipelines

---

## 19. Cross-Framework Comparison Dimensions

When comparing two or more agent frameworks, use these dimensions:

| Dimension | Spectrum |
|-----------|----------|
| **Memory model** | Bounded/curated ← → Unbounded/append-only |
| **Persistence** | SQLite (structured) ← → Files (human-readable) |
| **Prompt caching** | Cache-aware split ← → Naive concatenation |
| **Compression** | Structured template ← → Freeform summary |
| **Tool registration** | Self-registering ← → Centrally declared |
| **Skill learning** | Static/manual ← → Self-creating/self-improving |
| **Security posture** | Advisory guardrails ← → Enforced policies + audit |
| **Sandbox model** | Host-only ← → Container-by-default |
| **Provider flexibility** | Single provider ← → Multi-key rotation + smart routing |
| **Research integration** | Product only ← → Product + RL training platform |
| **Channel breadth** | Few channels ← → 15+ platform adapters |
| **Process model** | Single daemon ← → CLI + daemon + editor + API |

---

*Framework v1.0 — Derived from analysis of OpenClaw (v2026.4.1, ~8,700 TypeScript files) and Hermes Agent (278K Python lines)*
*April 2026*
