# TAU Agent Runtime — Ecosystem Concept

## 1. Vision and Problem Statement

### Current State

The TAU (Tailored Agentic Units) ecosystem provides two layers of capability today:

**tau-core** — A platform and model-agnostic Go library providing LLM interaction primitives. Supports five protocols (Chat, Vision, Tools, Embeddings, Audio) across pluggable providers (Ollama, Azure). Handles single-turn request/response interactions with configuration management, HTTP client retry logic, and streaming support.

**tau-orchestrate** — Go-native agent coordination primitives built on tau-core. Provides Hub-based messaging (point-to-point, request-response, pub/sub), LangGraph-inspired state management with immutable State and StateGraph execution, composable workflow patterns (Chain, Parallel, Conditional), observability via the Observer interface, and checkpoint-based resume capability.

**agent-lab** (reference only) — A predecessor web service built with the libraries that preceded tau-core and tau-orchestrate. It demonstrates how hand-crafted workflows are executed through a web API with an embedded web client. agent-lab is not part of the TAU ecosystem but informed this concept's understanding of existing patterns, production integration challenges, and infrastructure limitations.

### The Gap

The current infrastructure is heavily focused on explicit, code-based workflow definitions that solve a singular use case: statically-defined, compile-time workflow execution. There is no capability for an agent to dynamically organize its own workflow, select and execute tools based on environmental feedback, maintain persistent memory across sessions, discover and load modular skills, or integrate with external systems via standard protocols like MCP.

The gap is an **agent runtime** — the middle layer that transforms a single-turn `Agent.Tools()` call into a sustained, autonomous agentic loop. As Anthropic's guidance states: "Agents are typically just LLMs using tools based on environmental feedback in a loop."

### Target State

A modular ecosystem of independent Go libraries, each encapsulating a distinct capability domain, that compose into a powerful agent runtime. The runtime enables:

- Self-organizing workflow sequences driven by the model's reasoning
- Request for missing, unclear, or contradictory detail clarifications
- Dynamic reaction to injections and environmental feedback
- Plan mode with markdown-based files at configurable locations
- A secure, configurable tool system with permissions
- Persistent, bootstrapped memory loaded into context on start
- Skills for structured capability specifications and commands
- MCP support for integration with external data platforms and APIs
- Deployment flexibility: local environment or container sandbox
- A standard API for interfacing with the runtime regardless of deployment mode

### Design Philosophy

Following Anthropic's core principle: **simple, composable patterns over complex frameworks**. Start with the minimum viable abstraction at each layer. Only increase complexity when it demonstrably improves outcomes. Each library should be independently useful, testable, and comprehensible.

---

## 2. Anthropic Reference Analysis

The following insights are drawn from Anthropic's engineering publications and Claude platform documentation. Each insight is mapped to the specific TAU library it informs.

### 2.1 Building Effective Agents

**Source**: [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)

| Insight | Implication | Library |
|---------|------------|---------|
| Workflows (predefined code paths) vs Agents (LLM-directed processes) | The runtime must support both modes — static orchestration via tau-orchestrate AND dynamic agentic loops | tau-runtime |
| "Agents are typically just LLMs using tools based on environmental feedback in a loop" | The agentic loop is fundamentally simple: prompt → reason → tool calls → execute → observe → iterate | tau-runtime |
| Agent-Computer Interface (ACI) design deserves equal investment to HCI | Tool definitions need absolute paths, example usage, edge cases, clear boundaries. Tool engineering > prompt engineering | tau-tools |
| Start with direct LLM API calls, not frameworks | Each library should be a thin, composable primitive — not a framework. Consumers compose what they need | All |
| Poka-yoke principles for tool arguments | Tools should reduce mistake likelihood through argument design (e.g., absolute paths prevent relative path errors) | tau-tools |
| Patterns: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer | tau-orchestrate already covers these via Chain, Conditional, Parallel, StateGraph. The runtime composes them with the agentic loop | tau-orchestrate, tau-runtime |

### 2.2 Context Engineering

**Source**: Anthropic engineering guidance on context management

| Insight | Implication | Library |
|---------|------------|---------|
| Context is a finite resource; models experience "context rot" as token count increases | Active context window management with compaction strategies | tau-session |
| Compaction: summarize and reinitiate with condensed context | Summarization strategy using a dedicated agent to compress older messages | tau-session |
| Structured note-taking: external memory files the agent reads and writes | Working memory as filesystem-backed scratchpad for current task state | tau-memory |
| Sub-agent architectures: delegate focused subtasks to reduce main context load | The runtime should support spawning sub-agents for focused work | tau-runtime |
| Just-in-time context: lightweight identifiers rather than loading everything upfront | Progressive disclosure — load skill metadata always, instructions on trigger, resources on demand | tau-skills |

### 2.3 Effective Harnesses for Long-Running Agents

**Source**: [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

| Insight | Implication | Library |
|---------|------------|---------|
| Operational discontinuity: each new session begins without memory | Bootstrap memory must persist across sessions and be loaded at startup | tau-memory |
| Progress files and shift handoff documentation | Working memory files that track what was done, what's next, and key decisions | tau-memory |
| Feature list tracking with pass/fail status | Structured note-taking for long-horizon task management | tau-memory |
| Clean state requirement: code at session boundaries should be mergeable | The runtime should enforce clean state at checkpoints | tau-runtime |

### 2.4 Claude Tool System

**Source**: [Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)

| Insight | Implication | Library |
|---------|------------|---------|
| Tool use lifecycle: provide tools → model decides → execute → return results → model uses results | This is the core agentic loop. The runtime orchestrates this cycle | tau-runtime |
| Parallel tool calls: multiple tool_use blocks in single response | Tool registry must support concurrent execution of multiple calls | tau-tools |
| MCP integration: convert MCP tools to standard tool format | MCP client discovers tools from external servers and wraps them as standard executors | tau-mcp |
| Tool result formatting: content field with string or structured data | Tool result type must be flexible (string, JSON, error indication) | tau-tools |

### 2.5 Agent Skills

**Source**: [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)

| Insight | Implication | Library |
|---------|------------|---------|
| Progressive disclosure: 3-level loading (metadata ~100 tokens, instructions <5k, resources unlimited) | Skills system must support lazy loading to minimize context consumption | tau-skills |
| SKILL.md format with YAML frontmatter | Standard skill format: name, description, triggers in frontmatter; instructions in markdown body | tau-skills |
| Filesystem-based architecture: skills as directories with scripts, templates, references | Skills are directories on disk, not database records. Agent reads them via standard file operations | tau-skills |
| Token efficiency: many skills installed with minimal context cost | Only skill metadata is always loaded. Instructions load on trigger. Resources load on demand | tau-skills |

### 2.6 Claude Agent SDK

**Source**: [Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)

| Insight | Implication | Library |
|---------|------------|---------|
| Built-in tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch | The runtime needs a standard set of built-in tools for filesystem and system interaction | tau-tools |
| Subagents: spawn specialized agents for focused subtasks via Task tool | The runtime should support delegating work to sub-agents with constrained tool sets | tau-runtime |
| Sessions: maintain context across multiple exchanges, resume with session ID | Session management with conversation history persistence and resumption | tau-session |
| MCP: connect to external systems via Model Context Protocol | MCP client with transport abstraction (stdio, SSE) for external tool discovery | tau-mcp |
| Permissions: control which tools an agent can use (allowlist/denylist) | Tool registry with permission enforcement before execution | tau-tools |
| Hooks: Pre/Post tool use, Stop, Session events | Runtime hooks for customizing behavior at key execution points | tau-runtime |
| Memory: project context and instructions via CLAUDE.md | Bootstrap memory loaded at startup, injected as system prompt context | tau-memory |
| Skills: filesystem-based SKILL.md with progressive disclosure | Skill discovery, matching, and lazy loading from configured directories | tau-skills |

### 2.7 Prompt Engineering

**Source**: [Prompt Engineering](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)

| Insight | Implication | Library |
|---------|------------|---------|
| System prompts define agent role and behavior | Bootstrap memory compiles into the system prompt. Clear, structured system prompts are critical | tau-memory |
| XML tags for structured prompt sections | Bootstrap content should use structured sections that the model can parse reliably | tau-memory |
| Chain of thought: let the model reason step by step | The runtime should not interfere with the model's reasoning — tool calls emerge from the model's analysis | tau-runtime |
| Long context tips: place important information at boundaries | Context manager should preserve system prompt and recent messages, compacting the middle | tau-session |

---

## 3. Ecosystem Target State

### 3.1 Library Topology

| Library | Domain | Depends On | Status |
|---------|--------|------------|--------|
| **tau-core** | Foundational types: Protocol, Message, Response structs, Config, Model | uuid | Restructure existing |
| **tau-agent** | LLM client: Agent, Client, Provider, Request, Parse, Mock | tau-core | Extract from tau-core |
| **tau-orchestrate** | Coordination: Hub, State, Workflows, Observability, Checkpoint | tau-agent | Adjust imports |
| **tau-memory** | Persistent memory: bootstrap loading, working memory, structured notes | (none) | New |
| **tau-tools** | Tool system: execution interface, registry, permissions, built-in tools | tau-core | New |
| **tau-session** | Conversation management: message history, context window, compaction | tau-core | New |
| **tau-skills** | Progressive disclosure: SKILL.md discovery, loading, matching | tau-memory | New |
| **tau-mcp** | MCP client: transport abstraction, tool discovery, stdio/SSE | tau-tools | New |
| **tau-runtime** | Agent runtime: agentic loop, plan mode, environment, composition | all above | New |

### 3.2 Dependency Hierarchy

```
                         tau-runtime
                       /   |    \     \
                      /    |     \     \
              tau-mcp  tau-skills  \    tau-orchestrate
                |         |        \        |
           tau-tools  tau-memory  tau-session  tau-agent
                \                    |         /
                 \                   |        /
                  +------ tau-core --------+
```

Key properties of this hierarchy:

- **Acyclic**: No circular dependencies at any level
- **Shallow**: Maximum depth of 3 (tau-runtime → tau-mcp → tau-tools → tau-core)
- **Independent foundations**: tau-memory has zero TAU dependencies; tau-tools and tau-session depend only on tau-core types
- **Clean separation**: Each library owns a single domain with no overlap

### 3.3 The tau-core Deconstruction

tau-core currently bundles foundational types with LLM client machinery. Libraries like tau-tools only need type definitions (e.g., `response.ToolCall`) but would pull in the entire HTTP client and provider system as a transitive dependency. This violates the principle of minimal coupling.

**tau-core (restructured)** becomes the foundational vocabulary of the ecosystem:

| Package | Contents | Rationale |
|---------|----------|-----------|
| `pkg/protocol/` | Protocol enum, Message type | Universal message vocabulary |
| `pkg/response/` | Struct definitions only: ChatResponse, ToolsResponse, ToolCall, TokenUsage, StreamingChunk, EmbeddingsResponse, AudioResponse | Response types are vocabulary — they describe what comes back from any LLM interaction. Parsing logic is client machinery |
| `pkg/config/` | AgentConfig, ProviderConfig, ModelConfig, ClientConfig, Duration | Configuration structures used across the ecosystem |
| `pkg/model/` | Model runtime type | Model representation bridging config and runtime |

**tau-agent (extracted)** becomes the LLM communication layer:

| Package | Contents | Rationale |
|---------|----------|-----------|
| `pkg/agent/` | Agent interface + implementation, Tool struct | High-level LLM interaction API |
| `pkg/client/` | HTTP client with retry logic, connection pooling | HTTP execution engine |
| `pkg/providers/` | Provider interface, registry, BaseProvider, Ollama, Azure | Provider abstraction and implementations |
| `pkg/request/` | ChatRequest, VisionRequest, ToolsRequest, etc. | Protocol-specific request construction |
| `pkg/parse/` | ParseTools, ParseChat, ParseStreamChunk, Parse dispatcher | Response parsing (JSON → structs) |
| `pkg/mock/` | MockAgent, MockProvider, MockClient | Test infrastructure |

**tau-orchestrate (adjusted)** updates its dependency from tau-core to tau-agent, since Hub coordination registers Agents and provides Agent access via MessageContext. No internal logic changes are needed — only import paths change.

---

## 4. Library Descriptions

### 4.1 tau-core (restructured)

**Purpose**: Provide the foundational type vocabulary shared across the entire TAU ecosystem. Any library that needs to reference a Protocol, Message, Response type, or Config structure depends on tau-core and nothing else.

**Scope boundary**:
- INCLUDES: Type definitions, enums, config structures, model representation
- EXCLUDES: HTTP clients, provider implementations, request construction, response parsing, agent behavior

**Key abstractions**:
- `protocol.Protocol` — Enum identifying interaction types (Chat, Vision, Tools, Embeddings, Audio)
- `protocol.Message` — Conversation message with Role and Content
- `response.ToolCall` — Structured tool call from model output
- `response.ChatResponse`, `response.ToolsResponse` — Response type definitions
- `response.TokenUsage` — Token accounting
- `config.AgentConfig` — Top-level agent configuration
- `model.Model` — Runtime model representation

**Anthropic alignment**: Foundation layer — no direct Anthropic guidance maps here. This is pure Go type infrastructure.

**Integration points**: Every TAU library depends on tau-core for shared types. tau-core depends on nothing except `github.com/google/uuid`.

**Skill deliverables**:
- `tau-core` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-core/` (update existing)
- `tau-core-dev` contributing skill in `.claude/skills/tau-core-dev/` (update existing)

---

### 4.2 tau-agent (new — extracted from tau-core)

**Purpose**: Provide the LLM communication layer — everything needed to send requests to language models and parse their responses. This is where the Agent interface, HTTP client, provider implementations, and request/response pipeline live.

**Scope boundary**:
- INCLUDES: Agent interface and implementation, HTTP client with retry, provider abstraction (Ollama, Azure), request construction, response parsing, test mocks
- EXCLUDES: Tool execution, conversation history, memory, skills, agentic loops

**Key abstractions**:
- `agent.Agent` — Interface for LLM interactions (Chat, Vision, Tools, Embed, Audio)
- `client.Client` — HTTP client with retry logic and connection pooling
- `providers.Provider` — Interface for LLM service providers
- `providers.Registry` — Thread-safe provider factory registration
- `request.Request` — Interface for protocol-specific request construction
- `parse.Parse` — Response parsing dispatcher

**Anthropic alignment**: Direct API access — Anthropic recommends starting with direct LLM API calls rather than complex frameworks. tau-agent is the thin, direct access layer.

**Integration points**:
- Depends on tau-core for types (Protocol, Message, Response structs, Config)
- Consumed by tau-orchestrate (Hub registers Agents)
- Consumed by tau-runtime (agentic loop calls Agent.Tools())

**Skill deliverables**:
- `tau-agent` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-agent/`
- `tau-agent-dev` contributing skill in `.claude/skills/tau-agent-dev/`

---

### 4.3 tau-orchestrate (adjusted)

**Purpose**: Provide Go-native agent coordination primitives for building multi-agent systems with state management and composable workflow patterns. No functional changes — only dependency adjustment from tau-core to tau-agent.

**Scope boundary**:
- INCLUDES: Hub messaging, StateGraph execution, State management, workflow patterns (Chain, Parallel, Conditional), Observer interface, CheckpointStore
- EXCLUDES: Agentic loops, tool execution, memory, skills

**Key abstractions**: Unchanged from current implementation.

**Anthropic alignment**: Workflow patterns (prompt chaining, routing, parallelization, orchestrator-workers) map directly to ProcessChain, ProcessConditional, ProcessParallel, and StateGraph.

**Integration points**:
- Depends on tau-agent (Hub uses Agent interface)
- Observer interface consumed by tau-runtime for event emission
- StateGraph can orchestrate multiple agent runtimes as nodes

**Skill deliverables**:
- `tau-orchestrate` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-orchestrate/` (update existing)
- `tau-orchestrate-dev` contributing skill in `.claude/skills/tau-orchestrate-dev/` (update existing)

---

### 4.4 tau-memory (new)

**Purpose**: Provide filesystem-based persistent memory for agents. Three tiers: bootstrap memory (persistent identity loaded at startup), working memory (current task scratchpad), and structured notes (organized knowledge artifacts). This is the equivalent of CLAUDE.md and the working memory patterns described in Anthropic's long-running agent harness guidance.

**Scope boundary**:
- INCLUDES: Bootstrap file loading and compilation, working memory read/write/append, structured note organization, filesystem operations for memory management
- EXCLUDES: Conversation history (tau-session), token counting (tau-session), context window management (tau-session), skill loading (tau-skills)

**Key abstractions**:
- Bootstrap — Ordered list of markdown files loaded at startup, compiled into system prompt context
- Working Memory — Filesystem-backed scratchpad the agent reads and writes during task execution
- Structured Notes — Organized filesystem layout for knowledge artifacts (concepts, guides, sessions, reviews)

**Anthropic alignment**:
- Bootstrap memory → CLAUDE.md persistent instructions
- Working memory → Structured note-taking for long-horizon tasks
- Shift handoff documentation from long-running agent harness guidance

**Integration points**:
- Zero TAU dependencies (filesystem-only operations)
- Consumed by tau-skills (skill filesystem patterns build on memory filesystem patterns)
- Consumed by tau-runtime (bootstrap loaded at startup, working memory available during execution)

**Skill deliverables**:
- `tau-memory` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-memory/`
- `tau-memory-dev` contributing skill in `.claude/skills/tau-memory-dev/`

---

### 4.5 tau-tools (new)

**Purpose**: Provide the tool system for agent-driven execution. Defines the interface for executable tools, a registry for tool management with permissions, and a set of built-in tools for filesystem and system interaction. Implements Anthropic's ACI (Agent-Computer Interface) design principles.

**Scope boundary**:
- INCLUDES: Tool executor interface, tool definition types, tool registry with permissions, built-in tools (read, write, edit, bash, glob, grep), parallel tool execution, tool result types
- EXCLUDES: MCP tool discovery (tau-mcp), agentic loop orchestration (tau-runtime), LLM interaction (tau-agent)

**Key abstractions**:
- Tool Executor — Interface for executable tools (Name, Definition, Execute)
- Tool Definition — JSON Schema description of a tool for LLM consumption
- Tool Result — Output of tool execution (content + error indication)
- Tool Registry — Registration, discovery, and access-controlled execution
- Permissions — Allowlist/denylist patterns for tool access control

**Anthropic alignment**:
- ACI design: tool definitions with absolute paths, example usage, edge cases, clear boundaries
- Poka-yoke principles: argument design that reduces mistake likelihood
- Built-in tools: Read, Write, Edit, Bash, Glob, Grep (from Agent SDK)
- Parallel tool calls: concurrent execution of multiple tool calls from single model response
- Permissions: allowlist/denylist patterns (from Agent SDK permissions)

**Integration points**:
- Depends on tau-core for `response.ToolCall` type (dispatching model-requested tool calls)
- Consumed by tau-mcp (MCP-discovered tools wrapped as Tool Executors)
- Consumed by tau-runtime (registry provides tools to the agentic loop)

**Skill deliverables**:
- `tau-tools` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-tools/`
- `tau-tools-dev` contributing skill in `.claude/skills/tau-tools-dev/`

---

### 4.6 tau-session (new)

**Purpose**: Provide conversation history management and context window optimization. Maintains ordered message sequences with support for all roles (user, assistant, system, tool), tracks token usage, and applies compaction strategies when conversations exceed the context window.

**Scope boundary**:
- INCLUDES: Message history buffer, multi-role support (including tool call and tool result messages), token counting interface, context window tracking, compaction strategies (truncation, summarization, sliding window)
- EXCLUDES: Persistent memory (tau-memory), skill loading (tau-skills), HTTP communication (tau-agent)

**Key abstractions**:
- Session — Ordered message buffer with append, truncate, and clear operations
- Token Counter — Interface for estimating token counts per message (model-specific implementations)
- Context Manager — Builds messages within a token budget, applying compaction when needed
- Compaction Strategy — Interface for reducing context (truncation, summarization, sliding window)

**Anthropic alignment**:
- Context as finite resource with active management
- Compaction via summarization (summarize and reinitiate)
- Sliding window with rolling summary
- Preserve system prompt and recent messages, compact the middle
- Long context tips: important information at prompt boundaries

**Integration points**:
- Depends on tau-core for `protocol.Message` type
- Consumed by tau-runtime (session manages conversation state within the agentic loop)

**Skill deliverables**:
- `tau-session` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-session/`
- `tau-session-dev` contributing skill in `.claude/skills/tau-session-dev/`

---

### 4.7 tau-skills (new)

**Purpose**: Provide the progressive disclosure skill system. Discovers SKILL.md files from configured filesystem paths, loads them in three levels (metadata → instructions → resources), and matches relevant skills to prompts. Directly implements the Agent Skills architecture from Claude's platform.

**Scope boundary**:
- INCLUDES: SKILL.md parsing (YAML frontmatter + markdown body), filesystem discovery across configured paths, 3-level progressive loading, trigger-based matching, resource file access
- EXCLUDES: Skill execution (skills provide instructions, the runtime follows them), tool definitions (tau-tools), MCP integration (tau-mcp)

**Key abstractions**:
- Skill — Metadata (name, description, triggers) + lazy-loaded instructions and resources
- Skill Loader — Discovers SKILL.md files from configured directories
- Skill Matcher — Determines which skills are relevant to a given prompt or context
- Level 1/2/3 loading — Metadata always loaded (~100 tokens), instructions on trigger (<5k tokens), resources on demand (unlimited)

**Anthropic alignment**:
- Progressive disclosure architecture from Agent Skills documentation
- SKILL.md format with YAML frontmatter
- Filesystem-based skill directories with scripts, templates, references
- Token-efficient loading: many skills installed with minimal context cost

**Integration points**:
- Depends on tau-memory (filesystem patterns for discovering and reading skill files)
- Consumed by tau-runtime (matched skills injected into context when relevant)

**Skill deliverables**:
- `tau-skills` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-skills/`
- `tau-skills-dev` contributing skill in `.claude/skills/tau-skills-dev/`

---

### 4.8 tau-mcp (new)

**Purpose**: Provide a Model Context Protocol client for discovering and invoking tools from external MCP servers. Abstracts transport mechanisms (stdio, SSE) and wraps discovered MCP tools as standard tau-tools executors so they integrate seamlessly with the tool registry.

**Scope boundary**:
- INCLUDES: MCP client connection management, transport abstraction (stdio, SSE), tool discovery from MCP servers, wrapping MCP tools as tau-tools executors, MCP handshake and protocol lifecycle
- EXCLUDES: MCP server implementation, tool registry management (tau-tools), built-in tools (tau-tools)

**Key abstractions**:
- MCP Client — Connects to an MCP server, discovers tools, and provides them as executors
- Transport — Interface abstracting communication (stdio for subprocess-based servers, SSE for HTTP-based servers)
- MCP Tool Wrapper — Adapts an MCP tool into a tau-tools ToolExecutor

**Anthropic alignment**:
- MCP integration from Agent SDK: connect to databases, browsers, APIs
- MCP tool format conversion to standard tool definitions
- Multiple MCP servers per agent

**Integration points**:
- Depends on tau-tools for the ToolExecutor interface (wraps MCP tools as executors)
- Consumed by tau-runtime (discovered tools registered in the tool registry)

**Skill deliverables**:
- `tau-mcp` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-mcp/`
- `tau-mcp-dev` contributing skill in `.claude/skills/tau-mcp-dev/`

---

### 4.9 tau-runtime (new)

**Purpose**: Compose all TAU ecosystem libraries into an agent runtime capable of autonomous, tool-driven task execution. Implements the agentic loop, plan mode, environment abstraction (local/container), hooks, and the standard API surface for interfacing with the runtime.

**Scope boundary**:
- INCLUDES: Agentic loop (prompt → reason → tool calls → execute → observe → iterate), plan mode (draft → approve → execute), environment/sandbox abstraction (local filesystem, container), hooks (pre/post tool use, iteration, stop), runtime configuration, standard API for external consumers
- EXCLUDES: Individual tool implementations (tau-tools), MCP transport details (tau-mcp), conversation compaction algorithms (tau-session), skill parsing (tau-skills), LLM communication (tau-agent), workflow orchestration (tau-orchestrate)

**Key abstractions**:
- Runtime — Central execution engine composing an Agent, Session, Tool Registry, Memory, Skills, and Observer
- Agentic Loop — The core execution cycle with configurable stop conditions and iteration limits
- Plan Mode — Structured planning with markdown plan files, draft/approve/execute lifecycle
- Environment — Interface abstracting where the agent executes (local host vs container sandbox)
- Hooks — Callbacks at key execution points (pre/post tool use, iteration, stop)
- Runtime Config — MaxIterations, parallel tool calls, stop conditions, context window size

**Anthropic alignment**:
- "LLMs using tools based on environmental feedback in a loop" — the agentic loop
- Agent SDK patterns: built-in tools, subagents, sessions, permissions, hooks, MCP
- Plan mode from Claude Code (.claude/plans/ with configurable directory)
- Sandbox/container execution from Claude Code security model
- Sub-agent delegation for focused subtasks

**Integration points**:
- Depends on tau-agent (calls Agent.Tools() in the loop)
- Depends on tau-tools (tool registry, permissions, built-in tools)
- Depends on tau-session (conversation history, context management)
- Depends on tau-memory (bootstrap loading, working memory)
- Depends on tau-skills (skill discovery and injection)
- Depends on tau-mcp (external tool discovery)
- Depends on tau-orchestrate (Observer for event emission; StateGraph can orchestrate multiple runtimes)

**Skill deliverables**:
- `tau-runtime` usage skill in `~/tau/tau-marketplace/plugins/tau/skills/tau-runtime/`
- `tau-runtime-dev` contributing skill in `.claude/skills/tau-runtime-dev/`

---

## 5. Existing Infrastructure Problem Set

The following limitations have been identified in the current tau-core and tau-orchestrate codebases. Solutions are intentionally deferred to the per-library concept sessions that will be conducted during implementation. Documenting them here ensures they are addressed as the ecosystem evolves.

### 5.1 tau-core: Agent Methods Create Fresh Messages

The current `Agent` interface methods (`Chat`, `Tools`, etc.) accept a `prompt string` parameter and internally call `initMessages()` to create a fresh message list each time. This design is incompatible with multi-turn conversations where the full history of user messages, assistant responses, tool calls, and tool results must be passed to the model.

**Impact**: The agentic loop in tau-runtime needs to pass full conversation history through the Agent interface. The current API cannot support this without bypassing the Agent interface entirely and using lower-level request/client packages directly.

**Deferred to**: tau-agent concept session (the extracted Agent interface must be redesigned to support message history passthrough while maintaining backward compatibility or providing a clean migration path).

### 5.2 tau-core: Tool Type Duplication

Tool definitions exist in two places: `agent.Tool` (used by consumers) and `providers.ToolDefinition` (used internally by providers). The Agent implementation converts between them. With the addition of tau-tools introducing its own canonical tool definition type, this becomes a three-way duplication problem.

**Impact**: Consumers of the tool system need a single, canonical type for defining tools. The current duplication creates confusion about which type to use.

**Deferred to**: tau-tools and tau-agent concept sessions (establish canonical types and conversion patterns).

### 5.3 tau-core: Message Type Lacks Tool Call/Result Support

`protocol.Message` has only `Role string` and `Content any`. The OpenAI-compatible tool calling protocol requires messages with:
- Role `"assistant"` containing `tool_calls` array
- Role `"tool"` containing `tool_call_id` for correlation

The current Message type cannot represent these roles without using the `Content any` field in non-obvious ways.

**Impact**: tau-session needs to store tool call and tool result messages in conversation history. The runtime needs to construct these messages for the agentic loop.

**Deferred to**: tau-core restructuring (the Message type needs to support tool-related roles and content, or a richer message type hierarchy is needed).

### 5.4 tau-orchestrate: Hub Dependency on Agent Interface

The Hub's `RegisterAgent` method and `MessageContext` struct directly reference the `agent.Agent` interface. When tau-core is deconstructed and the Agent interface moves to tau-agent, tau-orchestrate's dependency shifts from tau-core to tau-agent.

**Impact**: tau-orchestrate gains a dependency on the full LLM client layer (tau-agent) rather than just foundational types (tau-core). This is acceptable because the Hub fundamentally coordinates Agents, but it means tau-orchestrate cannot depend solely on tau-core.

**Deferred to**: Tier 0 restructuring (import path updates).

---

## 6. Prioritized Build Order

### Rationale

The build order follows three principles:
1. **Dependencies first**: A library is only built after all its dependencies exist
2. **Foundation before integration**: Libraries with fewer dependencies are built first
3. **Restructure before extend**: Existing infrastructure is adjusted before new capabilities are added

### Tier 0 — Restructure (prerequisite)

**0. tau-core deconstruction + tau-agent extraction**

Extract LLM client machinery from tau-core into a new tau-agent repository. Response struct definitions stay in tau-core; parsing functions move to tau-agent. Update tau-orchestrate imports from tau-core to tau-agent.

This is a refactoring effort, not a feature addition. The goal is to establish clean dependency boundaries before building new libraries on top of them.

agent-lab is a predecessor project, not part of the TAU ecosystem, and is excluded from this scope.

**Deliverables**: Restructured tau-core, new tau-agent repository, adjusted tau-orchestrate imports. Updated usage and dev skills for all three.

### Tier 1 — Foundation (independent libraries)

These libraries can be built in parallel after Tier 0, as they have no dependencies on each other.

**1. tau-memory** — Filesystem-based persistent memory with zero TAU dependencies. Bootstrap loading, working memory, structured notes. This is the simplest new library and can serve as a template for the others.

**2. tau-tools** — Tool execution interface, registry, permissions, built-in tools. Depends on tau-core for `response.ToolCall` type only. The built-in tools (read, write, edit, bash, glob, grep) are self-contained filesystem/system operations.

**3. tau-session** — Conversation history management with token tracking and compaction. Depends on tau-core for `protocol.Message` type only. The compaction strategies (truncation, summarization, sliding window) are independent of each other.

### Tier 2 — Integration (builds on Tier 1)

**4. tau-skills** — Progressive disclosure skill system. Depends on tau-memory for filesystem patterns. This can begin after tau-memory is complete.

**5. tau-mcp** — MCP client with transport abstraction. Depends on tau-tools for the ToolExecutor interface. This can begin after tau-tools is complete.

Tier 2 libraries can also proceed in parallel, as tau-skills depends on tau-memory (not tau-tools) and tau-mcp depends on tau-tools (not tau-memory).

### Tier 3 — Composition

**6. tau-runtime** — Agentic loop composing all libraries. This is the final library and depends on everything above. By the time tau-runtime development begins, all foundational and integration libraries will have established their interfaces, and the patterns for tau-core/tau-agent refactoring will be clear.

### Build Order Diagram

```
Tier 0:  [tau-core restructure + tau-agent extraction]
              |
              v
Tier 1:  [tau-memory]  [tau-tools]  [tau-session]    (parallel)
              |              |
              v              v
Tier 2:  [tau-skills]  [tau-mcp]                     (parallel)
                    \    /
                     v  v
Tier 3:          [tau-runtime]
```

---

## 7. Model Compatibility

The agent runtime must work across the platforms available at all classification levels. Both **tool calling** and **reasoning** are critical model capabilities — tool calling drives the agentic loop, while reasoning enables effective tool selection, result interpretation, error recovery, and multi-step planning. Models that support both capabilities simultaneously are the strongest candidates for agent runtime deployment.

### 7.1 Ollama (Self-Hosted)

Models supporting both tool calling and reasoning are the strongest candidates for local deployment:

| Model | Parameters | Tool Calling | Reasoning | Notes |
|-------|-----------|-------------|-----------|-------|
| **Qwen 3** | 0.6B - 235B | Strong | Strong | Hybrid think/no_think modes. Dense and MoE variants. Recommended starting point for local development |
| **GPT-OSS** (OpenAI) | 20B, 120B | Strong | Strong | OpenAI's open-weight models. Adjustable reasoning effort levels (low/medium/high). 20B runs on 16GB devices |
| **Kimi K2 Thinking** (Moonshot AI) | MoE (~32B active) | Excellent | Excellent | Best-in-class agentic capability; supports 200-300 sequential tool calls. Cloud variant available |
| **DeepSeek-V3.1** | 671B MoE | Strong | Strong | Hybrid thinking/non-thinking modes. Requires significant hardware |
| **Devstral Small 2** (Mistral) | 24B | Strong | Limited | 68% on SWE-Bench Verified. Strong for software engineering agents |
| **Nemotron-3-Nano** (NVIDIA) | 30B | Strong | Strong | Designed as an efficient agentic model |
| **GLM-4.7-Flash** | 30B | Strong | Strong | Lightweight deployment, good balance of performance and efficiency |

Models with strong tool calling but limited reasoning:

| Model | Parameters | Tool Calling | Reasoning | Notes |
|-------|-----------|-------------|-----------|-------|
| **Qwen3-Coder** | 30B, 480B | Excellent | Limited | Purpose-built for agentic coding workflows |
| **Ministral-3** (Mistral) | 3B, 8B, 14B | Strong | Limited | Edge deployment. Multi-modal, multi-lingual. Runs in browser via WebGPU |
| **Granite4** (IBM) | 350M, 1B, 3B | Good | Limited | Ultra-small enterprise models suitable for edge/IoT |

Models with strong reasoning but limited tool calling:

| Model | Parameters | Tool Calling | Reasoning | Notes |
|-------|-----------|-------------|-----------|-------|
| **DeepSeek-R1** | 1.5B - 671B | Poor | Excellent | Performance approaching o3 and Gemini 2.5 Pro. Tool calling is unreliable |
| **Magistral** (Mistral) | 24B | Limited | Strong | Dedicated reasoning model with transparent reasoning traces |
| **Phi-4-Reasoning** (Microsoft) | 14B | Limited | Strong | Rivals much larger models on complex reasoning |

**Recommendation**: Qwen 3 (8B or larger) or GPT-OSS 20B for development and testing. Both support tool calling and reasoning on consumer hardware with good balance of reliability and resource requirements.

### 7.2 Azure AI Foundry (Cloud-Hosted)

Frontier models available through Azure AI Foundry (Microsoft Foundry):

| Model Family | Tool Calling | Reasoning | Notes |
|-------------|-------------|-----------|-------|
| **GPT-5.2** | Excellent | Excellent | Freeform tool calling (no rigid schemas required). 272K context. State-of-the-art reasoning |
| **Claude Opus 4.5 / Sonnet 4.5** | Excellent | Excellent | Full tool suite including computer use, code execution, web search. Model router available |
| **Grok 4** (xAI) | Excellent | Excellent | First-principles reasoning with native tool use. New provider on Azure |
| **DeepSeek-V3.2** | Excellent | Excellent | Thinking Retention Mechanism preserves reasoning across tool calls |
| **o3 / o4-mini** | Good | Excellent | Native function calling within chain-of-thought. No parallel tool calls |
| **Mistral Large 3** | Excellent | Very Good | 673B MoE with 128 experts. Best open-weight option for multi-tool orchestration |
| **Phi-4-reasoning** (Microsoft) | Moderate | Very Good | Best small model option for constrained infrastructure |

### 7.3 Model Capability Detection

Model capability detection is a tau-runtime concern, not a tau-core concern. The runtime should determine model capabilities through configuration rather than runtime probing. The existing `ModelConfig.Capabilities` map (per-protocol options) can be extended to include capability metadata:

- Whether the model supports tool calling at all
- Whether the model supports parallel tool calls
- Whether the model supports reasoning / chain-of-thought (and whether it can be used simultaneously with tool calling)
- Maximum context window size (for tau-session compaction decisions)
- Whether the model supports vision (for multimodal tool results)

---

## 8. Iterative Development Process

### 8.1 Per-Library Concept Sessions

Each library follows the dev-workflow concept development pattern:

1. **Concept session** (`/dev-workflow concept`): Deep design of the library's architecture, interfaces, and scope boundaries. Produces a concept document at the library's `.claude/context/concepts/` directory.

2. **Concept → Milestones**: The concept is decomposed into implementation milestones, each representing a shippable increment of functionality.

3. **Milestones → Issues**: Each milestone is broken into GitHub issues with clear acceptance criteria.

4. **Task execution sessions** (`/dev-workflow task <issue>`): Individual issues are implemented through structured development sessions with plan mode, implementation, and closeout.

5. **Central document revision**: Upon completing a library, this concept document (`tau-runtime.md`) is revised with lessons learned, interface clarifications, and updated guidance for subsequent libraries.

### 8.2 Revision Cycle

```
Central Concept (this document)
         |
         v
  Library Concept Session
         |
         v
  Library Implementation
         |
         v
  Revise Central Concept  -----> Next Library Concept Session
```

Each completed library produces knowledge that refines the central vision:
- Interface patterns that emerged during implementation
- Integration challenges that inform adjacent library design
- Scope adjustments based on practical experience
- Updated dependency requirements

### 8.3 Skill Deliverables

Every library produces two skills:

**Usage skill** — Hosted in the tau plugin at `~/tau/tau-marketplace/plugins/tau/skills/tau-[name]/`
- Covers: installation, getting started, configuration, API usage, examples, error handling
- Audience: consumers of the library
- Format: SKILL.md with YAML frontmatter following existing tau-core/tau-orchestrate skill patterns

**Contributing skill** — Embedded in the library repository at `.claude/skills/tau-[name]-dev/`
- Covers: architecture overview, package hierarchy, extension patterns, testing strategy
- Audience: contributors to the library
- Format: SKILL.md with YAML frontmatter following existing tau-core-dev/tau-orchestrate-dev patterns

Skills are created as part of the library's implementation, not as a separate effort. They encode the knowledge gained during development and serve as living documentation.

### 8.4 Complete Skill Inventory

| Library | Usage Skill | Dev Skill |
|---------|------------|-----------|
| tau-core | `tau-core` (update existing) | `tau-core-dev` (update existing) |
| tau-agent | `tau-agent` (new) | `tau-agent-dev` (new) |
| tau-orchestrate | `tau-orchestrate` (update existing) | `tau-orchestrate-dev` (update existing) |
| tau-memory | `tau-memory` (new) | `tau-memory-dev` (new) |
| tau-tools | `tau-tools` (new) | `tau-tools-dev` (new) |
| tau-session | `tau-session` (new) | `tau-session-dev` (new) |
| tau-skills | `tau-skills` (new) | `tau-skills-dev` (new) |
| tau-mcp | `tau-mcp` (new) | `tau-mcp-dev` (new) |
| tau-runtime | `tau-runtime` (new) | `tau-runtime-dev` (new) |

---

## 9. Deployment Vision

The agent runtime supports two deployment modes with a single, standard API surface. Whether the runtime executes locally or within a container, the interface to the agent is identical.

### 9.1 Local Mode

The agent executes directly on the host system, similar to Claude Code's CLI. It has access to the local filesystem, installed tools, and network. This is the primary mode for development and interactive use.

- Tool execution: direct subprocess/filesystem calls
- Memory files: host filesystem paths
- Skills: discovered from host filesystem directories
- MCP servers: launched as local subprocesses

### 9.2 Container/Sandbox Mode

The agent executes within a container (Docker, Podman, or similar) and can only operate within the container's filesystem and network boundaries. This is the mode for production deployment, untrusted environments, and sandboxed execution.

- Tool execution: confined to container filesystem and processes
- Memory files: container-local paths (mounted or built-in)
- Skills: baked into the container image or mounted at runtime
- MCP servers: either inside the container or accessed via network transport (SSE)

### 9.3 Standard API

The runtime exposes a standard API that is deployment-mode agnostic. Whether the runtime is local or containerized, consumers interact through the same interface:

- Submit prompts
- Receive streaming responses (text and tool execution events)
- Manage sessions (create, resume, list)
- Configure tools and permissions
- Load and manage skills
- Create and approve plans

The environment abstraction is a tau-runtime concern. It provides an `Environment` interface that tools use for all filesystem and process operations, allowing the same tool implementations to work in both local and container modes.

---

## 10. Risk Analysis

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Tool-calling reliability varies across models** | Poor agentic loop behavior on weaker models; loops that never terminate or produce garbage tool calls | Model capability detection via configuration; fallback to simpler patterns (prompt chaining instead of agentic loop) for weaker models; clear minimum model requirements in documentation |
| **Context window overflow in long tasks** | Lost context, degraded responses, conversation amnesia | Compaction strategies in tau-session (truncation, summarization, sliding window); working memory offload in tau-memory; sub-agent delegation in tau-runtime |
| **MCP specification stability** | Breaking changes from upstream MCP protocol evolution | Abstract behind Transport interface; pin to specific MCP protocol version; isolate MCP concerns in tau-mcp so changes don't cascade |
| **Ecosystem complexity** | 9 repositories to maintain; dependency coordination overhead; contributor onboarding friction | Each library is independently useful and testable; clear scope boundaries prevent overlap; iterative build process surfaces issues early; skills provide onboarding documentation |
| **Token counting accuracy** | Incorrect compaction timing — either too aggressive (losing context) or too conservative (exceeding window) | TokenCounter interface allows model-specific implementations; conservative default estimates with configurable safety margin; compaction is a strategy pattern that can be swapped |
| **Breaking changes during tau-core deconstruction** | Existing consumers of tau-core (tau-orchestrate, external users) break when packages move | Semantic versioning with major version bump; clear migration documentation; Tier 0 is specifically scoped as a breaking restructure before new development begins |
| **Go module dependency management** | Diamond dependency problems if multiple tau libraries depend on different tau-core versions | All tau libraries should target the same tau-core version; Go modules handle this well as long as versions are coordinated; the iterative build process ensures each tier is stable before the next begins |

---

## Appendix A: Reference Sources

All insights in this document are derived from the following Anthropic publications:

1. **Building Effective Agents** — https://www.anthropic.com/engineering/building-effective-agents
2. **Effective Harnesses for Long-Running Agents** — https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents
3. **Tool Use Overview** — https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
4. **Agent Skills Overview** — https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
5. **Agent SDK Overview** — https://platform.claude.com/docs/en/agent-sdk/overview
6. **Prompt Engineering Overview** — https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview
7. **Anthropic Engineering Blog** — https://www.anthropic.com/engineering

---

## Appendix B: Revision History

| Date | Revision | Trigger |
|------|----------|---------|
| 2026-02-04 | Initial concept document | Concept development session |
