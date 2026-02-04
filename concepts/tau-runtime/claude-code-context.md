# Plan: TAU Agent Runtime Concept Document

## Session Type
Concept Development — `agent-runtime`

## Objective
Write a comprehensive concept document at `~/tau/tau-platform/concepts/tau-runtime/README.md` that establishes the long-term vision for the TAU Agent Runtime ecosystem. The document defines the complete ecosystem target state — including restructuring of existing repositories — the full library hierarchy, dependency graph, prioritized build order, and the iterative development process.

## Deliverable
Concept document: `~/tau/tau-platform/concepts/tau-runtime/README.md`

The document establishes architecture and interfaces at a level sufficient to scope individual library concept sessions. It does NOT contain detailed interface designs (those are deferred to per-library concept sessions). It defines:
- The complete ecosystem target state (existing + new repositories)
- What each library is responsible for and its scope boundary
- The dependency hierarchy between all libraries
- The prioritized build order with rationale
- How each library relates to Anthropic's published guidance
- The iterative development process (concept → build → revise central doc)
- Model compatibility analysis
- Identified problem sets in existing infrastructure (solutions deferred)
- Skill requirements per library (usage + dev)
- Deployment vision (local, container, standard API)

---

## Revised Ecosystem Architecture

### The tau-core Deconstruction

**Problem**: tau-core currently bundles foundational types (protocol.Message, response.ToolCall, config structs) with LLM client machinery (Agent, Client, Provider, Request). New libraries like tau-tools only need type definitions (e.g., response.ToolCall) but would pull in the entire HTTP client and provider system.

**Solution**: Deconstruct tau-core into foundational vocabulary types. Extract LLM client functionality into a new tau-agent library.

**tau-core (restructured)** — Foundational vocabulary of the ecosystem:
- `pkg/protocol/` — Protocol enum, Message type
- `pkg/response/` — Struct definitions only (ChatResponse, ToolsResponse, ToolCall, TokenUsage, StreamingChunk, EmbeddingsResponse, AudioResponse). Parsing functions move to tau-agent.
- `pkg/config/` — Configuration structures (AgentConfig, ProviderConfig, ModelConfig, ClientConfig)
- `pkg/model/` — Model runtime type

**tau-agent (new)** — LLM communication layer:
- `pkg/agent/` — Agent interface + implementation
- `pkg/client/` — HTTP client with retry logic
- `pkg/providers/` — Provider interface, registry, BaseProvider, Ollama, Azure
- `pkg/request/` — Protocol-specific request builders
- `pkg/parse/` — Response parsing functions (ParseTools, ParseStreamChunk, Parse)
- `pkg/mock/` — Test mocks (MockAgent, MockProvider, MockClient)
- Depends on: tau-core

**tau-orchestrate (adjusted)**:
- Depends on: tau-agent (Hub registers Agents, MessageContext holds Agent)
- No internal changes needed beyond import path updates

### Complete Library Topology

| Library | Domain | Depends On |
|---------|--------|------------|
| **tau-core** (restructured) | Foundational types: Protocol, Message, Response structs, Config, Model | uuid |
| **tau-agent** (new) | LLM client: Agent, Client, Provider, Request, Parse, Mock | tau-core |
| **tau-orchestrate** (adjusted) | Coordination: Hub, State, Workflows, Observability, Checkpoint | tau-agent |
| **tau-memory** (new) | Persistent memory: bootstrap loading, working memory, structured notes | (none) |
| **tau-tools** (new) | Tool system: execution interface, registry, permissions, built-in tools | tau-core |
| **tau-session** (new) | Conversation management: message history, context window, compaction | tau-core |
| **tau-skills** (new) | Progressive disclosure: SKILL.md discovery, loading, matching | tau-memory |
| **tau-mcp** (new) | MCP client: transport abstraction, tool discovery, stdio/SSE transports | tau-tools |
| **tau-runtime** (new) | Agent runtime: agentic loop, plan mode, environment/sandbox, composition | all above |

### Dependency Hierarchy

```
                        tau-runtime
                       /    |    \    \
                      /     |     \    \
              tau-mcp  tau-skills   \    tau-orchestrate
                |         |         \        |
           tau-tools  tau-memory  tau-session  tau-agent
                \                    |         /
                 \                   |        /
                  +------ tau-core --------+
```

---

## Prioritized Build Order (Precedence)

### Tier 0 — Restructure (prerequisite)
0. **tau-core deconstruction + tau-agent extraction**
   - Extract LLM client machinery from tau-core into tau-agent
   - Response types stay in tau-core (structs only); parsing moves to tau-agent
   - Update tau-orchestrate imports from tau-core → tau-agent
   - This is a refactoring effort, not a feature addition
   - Note: agent-lab (predecessor project) is NOT part of the TAU ecosystem and is excluded from this scope

### Tier 1 — Foundation (independent libraries, no new inter-dependencies)
1. **tau-memory** — Filesystem-based persistent memory. Zero tau dependencies. Bootstrap loading (CLAUDE.md equivalent), working memory files, structured note-taking.
2. **tau-tools** — Tool execution interface, definition types, registry, permissions, built-in tools (read, write, edit, bash, glob, grep). Depends on tau-core for response.ToolCall type.
3. **tau-session** — Conversation history management, message buffer with role support (including tool call/result roles), token tracking. Depends on tau-core for protocol.Message.

### Tier 2 — Integration (builds on Tier 1)
4. **tau-skills** — SKILL.md progressive disclosure system. Filesystem discovery, 3-level loading (metadata → instructions → resources), trigger matching. Depends on tau-memory.
5. **tau-mcp** — MCP client with stdio and SSE transports. Tool discovery from MCP servers, wrapping discovered tools as tau-tools executors. Depends on tau-tools.

### Tier 3 — Composition
6. **tau-runtime** — Agentic loop composing all libraries. Plan mode with markdown files. Environment/sandbox abstraction (local + container). Standard API surface. Depends on tau-agent + all Tier 1-2 libraries + tau-orchestrate.

---

## Document Sections

### 1. Vision and Problem Statement
- Current state analysis (tau-core, tau-orchestrate)
- agent-lab as reference material only (predecessor project demonstrating patterns and limitations, not a TAU ecosystem member)
- The gap: no dynamic agent runtime
- Target state: modular ecosystem of independent libraries
- Design philosophy: Anthropic's "simple, composable patterns"

### 2. Anthropic Reference Analysis
Key insights mapped to library decisions:
- **Building Effective Agents**: Agentic loop pattern → tau-runtime; ACI/tool design → tau-tools
- **Context Engineering**: Finite resource, compaction, sub-agents → tau-session
- **Long-Running Harnesses**: Shift handoff, progress tracking → tau-memory
- **Tool System**: Client/server tools, lifecycle, parallel calls → tau-tools
- **Agent Skills**: Progressive disclosure, 3-level loading → tau-skills
- **Agent SDK**: Built-in tools, subagents, sessions, permissions, hooks, MCP → tau-runtime
- **Prompt Engineering**: System prompt structure → tau-memory (bootstrap)

### 3. Ecosystem Target State
Full topology table, dependency hierarchy diagram, scope boundaries per library.

### 4. Library Descriptions
For each library (existing restructured + new):
- **Purpose**: Problem it solves
- **Scope boundary**: What it does and does NOT include
- **Key abstractions**: Core types/interfaces (named, not designed in detail)
- **Anthropic alignment**: Which guidance it implements
- **Integration points**: How it connects to other libraries
- **Skill deliverables**: tau-[name] usage skill + tau-[name]-dev contributing skill

### 5. Existing Infrastructure Problem Set
Document limitations without proposing solutions (deferred to per-library concept sessions):
- tau-core: Agent methods create fresh messages, no multi-turn support
- tau-core: Tool type duplication (agent.Tool vs providers.ToolDefinition)
- tau-core: protocol.Message lacks tool call/result role support
- tau-orchestrate: Hub depends on Agent interface (impacts deconstruction)

### 6. Prioritized Build Order
Tiered execution plan with rationale for ordering.

### 7. Model Compatibility
- Ollama: Qwen 3, Llama 3.2/3.3, Mistral, DeepSeek, Granite 3.2, Phi-4-Mini
- Azure AI Foundry: GPT-4o, Claude models, Mistral Large, Llama 3.x
- Tool-calling reliability as critical capability
- Model capability detection as tau-runtime concern

### 8. Iterative Development Process
- Each library gets its own `dev-workflow concept` session
- Concept session → milestones → GitHub issues → task execution sessions
- Completing a library triggers revision of this central concept document
- Updated central doc informs next library's concept session
- Two skills per library:
  - `tau-[name]` in `~/tau/tau-marketplace/plugins/tau/skills/` (usage)
  - `.claude/skills/tau-[name]-dev/` in the library repo (contributing)

### 9. Deployment Vision
- Local mode: CLI agent on host system
- Container/sandbox mode: Agent confined to container
- Standard API: Identical interface regardless of deployment
- Environment abstraction in tau-runtime

### 10. Risk Analysis
- Tool-calling reliability across models
- Context window overflow
- MCP specification stability
- Ecosystem complexity (repo count)
- Token counting accuracy
- Breaking changes during tau-core deconstruction

---

## Critical Source Files

### tau-core (to be restructured)
- `pkg/agent/agent.go` — Agent interface, Tool struct, initMessages() → moves to tau-agent
- `pkg/protocol/message.go` — Message type (stays in tau-core)
- `pkg/protocol/protocol.go` — Protocol enum (stays in tau-core)
- `pkg/response/tools.go` — ToolsResponse, ToolCall structs (stay); ParseTools() moves to tau-agent
- `pkg/response/chat.go` — ChatResponse struct (stays); ParseChat() moves
- `pkg/response/parse.go` — Parse() dispatcher moves to tau-agent
- `pkg/response/streaming.go` — StreamingChunk struct (stays); ParseStreamChunk() moves
- `pkg/providers/provider.go` — Provider interface, ToolDefinition → moves to tau-agent
- `pkg/providers/registry.go` — Registry → moves to tau-agent
- `pkg/providers/base.go` — BaseProvider → moves to tau-agent
- `pkg/providers/ollama.go` — Ollama provider → moves to tau-agent
- `pkg/providers/azure.go` — Azure provider → moves to tau-agent
- `pkg/request/` — All request types → move to tau-agent
- `pkg/client/` — HTTP client → moves to tau-agent
- `pkg/config/` — Config structs (stay in tau-core)
- `pkg/model/` — Model type (stays in tau-core)
- `pkg/mock/` — Mocks → move to tau-agent

### tau-orchestrate (import adjustments)
- `pkg/hub/hub.go` — Uses agent.Agent → update import to tau-agent
- `pkg/observability/observer.go` — Observer interface, EventType (no changes)

### Existing Skill Patterns
- `~/tau/tau-marketplace/plugins/tau/skills/tau-core/SKILL.md` — Usage skill pattern
- `~/tau/tau-core/.claude/skills/tau-core-dev/SKILL.md` — Dev skill pattern
- `~/tau/tau-marketplace/plugins/tau/.claude-plugin/plugin.json` — Plugin metadata

---

## Context Snapshot — 2026-02-04

**Current State**: Concept development session complete. The central concept document has been written and placed at `~/tau/tau-platform/concepts/tau-runtime/README.md`.

**Files Created**:
- `~/tau/tau-platform/concepts/tau-runtime/README.md` — The central concept document (10 sections, ~600 lines)
- `~/tau/tau-platform/concepts/tau-runtime/claude-code-context.md` — This file (plan + context snapshot)

**Research Performed**:
- Full codebase exploration of tau-core (`~/tau/tau-core`) — all packages analyzed
- Full codebase exploration of tau-orchestrate (`~/tau/tau-orchestrate`) — all packages analyzed
- Full codebase exploration of agent-lab (`~/code/agent-lab`) — used as reference material only
- Exploration of tau-marketplace (`~/tau/tau-marketplace`) — skill patterns, plugin structure
- Exploration of tau-platform (`~/tau/tau-platform`) — directory structure, existing drafts
- Web research: 7 Anthropic sources fetched and analyzed (Building Effective Agents, Agent Skills, Agent SDK, Prompt Engineering, Effective Harnesses, Engineering Blog index, Tool Use)

**Key Decisions Made**:
- **Capability-per-library topology**: Each foundational capability is its own independent repository
- **tau-core deconstruction**: Split foundational types (stay) from LLM client machinery (→ tau-agent)
- **Response type split**: Struct definitions stay in tau-core, parsing functions move to tau-agent
- **3-tier build order**: Tier 0 (restructure) → Tier 1 (memory, tools, session) → Tier 2 (skills, mcp) → Tier 3 (runtime)
- **Iterative revision**: Completing each library triggers revision of the central concept document
- **Two skills per library**: Usage skill in marketplace + dev skill in repo
- **agent-lab excluded**: Predecessor project used for reference only, not part of TAU ecosystem
- **Interface design deferred**: tau-core Agent interface refactoring waits until Tier 1-2 libraries establish integration patterns
- **Existing problems documented, not solved**: 4 tau-core/tau-orchestrate limitations recorded for future resolution

**Next Steps**:
1. Start Tier 0: `dev-workflow concept` session for tau-core deconstruction + tau-agent extraction
2. After Tier 0 completion: revise this central concept document
3. Start Tier 1 libraries (can proceed in parallel): tau-memory, tau-tools, tau-session
4. After each Tier 1 completion: revise central concept document
5. Continue through Tier 2 and Tier 3

**Blockers/Questions**:
- None identified. The concept is approved and ready for Tier 0 execution.
