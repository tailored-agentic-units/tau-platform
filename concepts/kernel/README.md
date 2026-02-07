# TAU Kernel — Ecosystem Concept

## 1. Vision and Problem Statement

### Current State

The TAU (Tailored Agentic Units) ecosystem provides two layers of capability today:

**core** — A platform and model-agnostic Go package providing LLM interaction primitives. Supports five protocols (Chat, Vision, Tools, Embeddings, Audio) across pluggable providers (Ollama, Azure). Handles single-turn request/response interactions with configuration management, HTTP client retry logic, and streaming support.

**orchestrate** — Go-native agent coordination primitives built on core types and the agent interface. Provides Hub-based messaging (point-to-point, request-response, pub/sub), LangGraph-inspired state management with immutable State and StateGraph execution, composable workflow patterns (Chain, Parallel, Conditional), observability via the Observer interface, and checkpoint-based resume capability.

**agent-lab** (reference only) — A predecessor web service built with the libraries that preceded the kernel subsystems. It demonstrates how hand-crafted workflows are executed through a web API with an embedded web client. agent-lab is not part of the TAU ecosystem but informed this concept's understanding of existing patterns, production integration challenges, and infrastructure limitations.

### The Gap

The current infrastructure is heavily focused on explicit, code-based workflow definitions that solve a singular use case: statically-defined, compile-time workflow execution. There is no capability for an agent to dynamically organize its own workflow, select and execute tools based on environmental feedback, maintain persistent memory across sessions, discover and load modular skills, or integrate with external systems via standard protocols like MCP.

The gap is an **agent runtime** — the middle layer that transforms a single-turn `Agent.Tools()` call into a sustained, autonomous agentic loop. As Anthropic's guidance states: "Agents are typically just LLMs using tools based on environmental feedback in a loop."

### Target State

A kernel with integrated subsystems in a single Go module, each encapsulating a distinct capability domain, that compose into a powerful agent runtime. The runtime enables:

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

Following Anthropic's core principle: **simple, composable patterns over complex frameworks**. Start with the minimum viable abstraction at each layer. Only increase complexity when it demonstrably improves outcomes. Each subsystem should be independently useful, testable, and comprehensible.

### Runtime Boundary Principle

The runtime is a closed-loop input/output processing system. Inside the runtime, the only concern is managing receipt of context, processing the workflow with additional inputs/outputs as needed, and returning the result.

The runtime has **zero awareness of extensions** — it does not know or care what connects to it. The runtime *interface* provides the extensibility points for external infrastructure:

- **Session bootstrapping** — context provided from external persistence sources
- **Context exchange** — sending and receiving additional context in various formats
- **Metadata output** — structured metadata emitted during processing
- **Administration** — diagnostic, validation, and administrative features

Any concerns beyond processing are outside the runtime's boundary: data persistence, integration with IAM, containerization standards, and platform-specific infrastructure. These are layers of complexity that reside outside the runtime, connecting through its interface.

This is analogous to the relationship between the Linux kernel and its ecosystem. The kernel establishes standards and provides a syscall boundary. Extensions are userspace services that connect through that boundary. The kernel doesn't reach out to extensions — they connect to it. This enables the same kernel to serve as the foundation for an embedded device OS, a desktop OS, a server OS, or any other operating system. Similarly, the TAU kernel serves as the foundation for embedded agents, desktop agents, server agents, or any other deployment model.

### Extension Ecosystem

Each component external to the kernel is an extension, analogous to an operating system userspace service:

- **Persistence** — session state storage, memory file management
- **IAM** — authentication, authorization
- **Container/Sandbox** — execution environment management
- **MCP Gateway** — proxies MCP servers for external tool access
- **Observability** — metrics, tracing, logging export
- **UI** — web interface, CLI, or other user-facing interfaces

Extensions connect to the kernel through its interface — the kernel never reaches out to extensions. For local development, the kernel composes with locally-run extensions (e.g., via Docker Compose). For production, extensions transition to cloud or self-hosted services. The kernel binary remains the same in both cases.

### Proof-of-Concept Validation

The claude-classify-docs implementation (see `~/tau/tau-platform/archive/tau-runtime/claude-classify-docs/`) validates the core design thesis. It reimplements a Go web service (~11,000 lines backed by PostgreSQL and blob storage) as a Claude Code project (~600 lines of markdown configuration). Key findings that inform the kernel architecture:

- **Three primitives suffice**: Memory (bootstrap context), skills (capability specifications), and subagents (parallel task delegation) compose into a functional agent runtime. The kernel formalizes exactly these: memory, skills, and the kernel runtime (subagent orchestration).
- **Component mapping is direct**: Every agent-lab component (StateGraph, State nodes, ProcessParallel, system prompts, checkpoints) maps cleanly to a Claude Code feature (orchestrating skill, individual skills, concurrent Tasks, skill content, working directory files). This validates that the kernel subsystem interfaces capture the essential abstractions.
- **Interfaces are the product**: The classify-docs workflow runs identically on any runtime that implements the tools, skills, and memory interfaces. The workflow specification (prompts, graph topology, scoring factors) remains constant — only the execution machinery changes.
- **Filesystem as state**: Working directory files provide implicit checkpointing and state management without a database. This validates the memory subsystem's filesystem-first approach.
- **Skill orchestration replaces code**: The kernel's value is enabling complex, multi-stage workflows through standardized, composable primitives rather than bespoke infrastructure.

---

## 2. Anthropic Reference Analysis

The following insights are drawn from Anthropic's engineering publications and Claude platform documentation. Each insight is mapped to the specific kernel subsystem it informs.

### 2.1 Building Effective Agents

**Source**: [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents)

| Insight | Implication | Subsystem |
|---------|------------|-----------|
| Workflows (predefined code paths) vs Agents (LLM-directed processes) | The kernel must support both modes — static orchestration via orchestrate AND dynamic agentic loops | kernel |
| "Agents are typically just LLMs using tools based on environmental feedback in a loop" | The agentic loop is fundamentally simple: prompt → reason → tool calls → execute → observe → iterate | kernel |
| Agent-Computer Interface (ACI) design deserves equal investment to HCI | Tool definitions need absolute paths, example usage, edge cases, clear boundaries. Tool engineering > prompt engineering | tools |
| Start with direct LLM API calls, not frameworks | Each subsystem should be a thin, composable primitive — not a framework. Consumers compose what they need | All |
| Poka-yoke principles for tool arguments | Tools should reduce mistake likelihood through argument design (e.g., absolute paths prevent relative path errors) | tools |
| Patterns: prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer | orchestrate already covers these via Chain, Conditional, Parallel, StateGraph. The kernel composes them with the agentic loop | orchestrate, kernel |

### 2.2 Context Engineering

**Source**: [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

| Insight | Implication | Subsystem |
|---------|------------|-----------|
| Context is a finite resource; models experience "context rot" as token count increases | Active context window management with compaction strategies | session |
| Compaction: summarize and reinitiate with condensed context | Summarization strategy using a dedicated agent to compress older messages | session |
| Structured note-taking: external memory files the agent reads and writes | Working memory as filesystem-backed scratchpad for current task state | memory |
| Sub-agent architectures: delegate focused subtasks to reduce main context load | The kernel should support spawning sub-agents for focused work | kernel |
| Just-in-time context: lightweight identifiers rather than loading everything upfront | Progressive disclosure — load skill metadata always, instructions on trigger, resources on demand | skills |

### 2.3 Effective Harnesses for Long-Running Agents

**Source**: [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)

| Insight | Implication | Subsystem |
|---------|------------|-----------|
| Operational discontinuity: each new session begins without memory | Bootstrap memory must persist across sessions and be loaded at startup | memory |
| Progress files and shift handoff documentation | Working memory files that track what was done, what's next, and key decisions | memory |
| Feature list tracking with pass/fail status | Structured note-taking for long-horizon task management | memory |
| Clean state requirement: code at session boundaries should be mergeable | The kernel should enforce clean state at checkpoints | kernel |

### 2.4 Claude Tool System

**Source**: [Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)

| Insight | Implication | Subsystem |
|---------|------------|-----------|
| Tool use lifecycle: provide tools → model decides → execute → return results → model uses results | This is the core agentic loop. The kernel orchestrates this cycle | kernel |
| Parallel tool calls: multiple tool_use blocks in single response | Tool registry must support concurrent execution of multiple calls | tools |
| MCP integration: convert MCP tools to standard tool format | MCP client discovers tools from external servers and wraps them as standard executors | mcp |
| Tool result formatting: content field with string or structured data | Tool result type must be flexible (string, JSON, error indication) | tools |

### 2.5 Agent Skills

**Source**: [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)

| Insight | Implication | Subsystem |
|---------|------------|-----------|
| Progressive disclosure: 3-level loading (metadata ~100 tokens, instructions <5k, resources unlimited) | Skills system must support lazy loading to minimize context consumption | skills |
| SKILL.md format with YAML frontmatter | Standard skill format: name, description, triggers in frontmatter; instructions in markdown body | skills |
| Filesystem-based architecture: skills as directories with scripts, templates, references | Skills are directories on disk, not database records. Agent reads them via standard file operations | skills |
| Token efficiency: many skills installed with minimal context cost | Only skill metadata is always loaded. Instructions load on trigger. Resources load on demand | skills |

### 2.6 Claude Agent SDK

**Source**: [Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview)

| Insight | Implication | Subsystem |
|---------|------------|-----------|
| Built-in tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch | The kernel needs a standard set of built-in tools for filesystem and system interaction | tools |
| Subagents: spawn specialized agents for focused subtasks via Task tool | The kernel should support delegating work to sub-agents with constrained tool sets | kernel |
| Sessions: maintain context across multiple exchanges, resume with session ID | Session management with conversation history persistence and resumption | session |
| MCP: connect to external systems via Model Context Protocol | MCP client with transport abstraction (stdio, SSE) for external tool discovery | mcp |
| Permissions: control which tools an agent can use (allowlist/denylist) | Tool registry with permission enforcement before execution | tools |
| Hooks: Pre/Post tool use, Stop, Session events | Kernel hooks for customizing behavior at key execution points | kernel |
| Memory: project context and instructions via CLAUDE.md | Bootstrap memory loaded at startup, injected as system prompt context | memory |
| Skills: filesystem-based SKILL.md with progressive disclosure | Skill discovery, matching, and lazy loading from configured directories | skills |

### 2.7 Prompt Engineering

**Source**: [Prompt Engineering](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)

| Insight | Implication | Subsystem |
|---------|------------|-----------|
| System prompts define agent role and behavior | Bootstrap memory compiles into the system prompt. Clear, structured system prompts are critical | memory |
| XML tags for structured prompt sections | Bootstrap content should use structured sections that the model can parse reliably | memory |
| Chain of thought: let the model reason step by step | The kernel should not interfere with the model's reasoning — tool calls emerge from the model's analysis | kernel |
| Long context tips: place important information at boundaries | Context manager should preserve system prompt and recent messages, compacting the middle | session |

---

## 3. Kernel Architecture

### 3.1 Subsystem Topology

| Subsystem | Domain | Depends On | Status |
|-----------|--------|------------|--------|
| **core** | Foundational types: Protocol, Message, Response structs, Config, Model | uuid | Complete |
| **agent** | LLM client: Agent, Client, Provider, Request, Mock | core | Complete |
| **orchestrate** | Coordination: Hub, State, Workflows, Observability, Checkpoint | agent | Complete |
| **memory** | Persistent memory: bootstrap loading, working memory, structured notes | (none) | Skeleton |
| **tools** | Tool system: execution interface, registry, permissions, built-in tools | core | Skeleton |
| **session** | Conversation management: message history, context window, compaction | core | Skeleton |
| **skills** | Progressive disclosure: SKILL.md discovery, loading, matching | memory | Skeleton |
| **mcp** | MCP client: transport abstraction, tool discovery, stdio/SSE | tools | Skeleton |
| **kernel** | Agent runtime: agentic loop, plan mode, environment, composition | all above | Skeleton |

### 3.2 Dependency Hierarchy

```
                           kernel
                       /   |    \     \
                      /    |     \     \
                    mcp  skills   \  orchestrate
                     |     |       \           |
                   tools  memory   session   agent
                      \              |         /
                       \             |        /
                        +------- core -------+
```

Key properties of this hierarchy:

- **Acyclic**: No circular dependencies at any level
- **Shallow**: Maximum depth of 3 (kernel → mcp → tools → core)
- **Independent foundations**: memory has zero internal dependencies; tools and session depend only on core types
- **Clean separation**: Each subsystem owns a single domain with no overlap
- **Enforced by Go**: Import rules and the type system enforce boundaries within the single module

### 3.3 Repository Strategy

The kernel is a single repository with a single Go module:

| Property | Value |
|----------|-------|
| **Repository** | `tailored-agentic-units/kernel` |
| **Module path** | `github.com/tailored-agentic-units/kernel` |
| **Go version** | 1.25.7 |
| **Versioning** | Single version for all subsystems |

```
kernel/
├── go.mod                  # Single module
├── core/                   # Foundational types
│   ├── config/
│   ├── model/
│   ├── protocol/
│   └── response/
├── agent/                  # LLM communication
│   ├── client/
│   ├── mock/
│   ├── providers/
│   └── request/
├── orchestrate/            # Multi-agent coordination
│   ├── config/
│   ├── hub/
│   ├── messaging/
│   ├── observability/
│   ├── state/
│   └── workflows/
├── memory/                 # Persistent memory (skeleton)
├── tools/                  # Tool execution (skeleton)
├── session/                # Conversation management (skeleton)
├── skills/                 # Progressive disclosure (skeleton)
├── mcp/                    # MCP client (skeleton)
├── kernel/                 # Runtime loop + ConnectRPC composition (skeleton)
├── rpc/                    # ConnectRPC infrastructure (proto, buf, generated code)
├── cmd/                    # Entry points (kernel, prompt-agent)
├── tests/                  # Kernel-wide integration tests only
└── .claude/                # Project configuration, skills, context documents
```

**Key conventions**:
- Tests co-located with source (`*_test.go` alongside `.go` files in each package)
- Black-box testing using `package_test` suffix
- Top-level `tests/` reserved for kernel-wide integration tests only
- No `pkg/` prefix — packages live directly under their subsystem root
- Package boundaries enforced by Go's type system and import rules

### 3.4 The core Deconstruction (Complete)

The original tau-core bundled foundational types with LLM client machinery. This deconstruction has been completed as part of the monorepo migration.

**core** is now the foundational vocabulary of the kernel:

| Package | Contents | Rationale |
|---------|----------|-----------|
| `core/protocol/` | Protocol enum, Message type | Universal message vocabulary |
| `core/response/` | Struct definitions only: ChatResponse, ToolsResponse, ToolCall, TokenUsage, StreamingChunk, EmbeddingsResponse, AudioResponse | Response types are vocabulary — they describe what comes back from any LLM interaction |
| `core/config/` | AgentConfig, ProviderConfig, ModelConfig, ClientConfig, Duration | Configuration structures used across the kernel |
| `core/model/` | Model runtime type | Model representation bridging config and runtime |

**agent** is the extracted LLM communication layer:

| Package | Contents | Rationale |
|---------|----------|-----------|
| `agent/` | Agent interface + implementation, Tool struct | High-level LLM interaction API |
| `agent/client/` | HTTP client with retry logic, connection pooling | HTTP execution engine |
| `agent/providers/` | Provider interface, registry, BaseProvider, Ollama, Azure | Provider abstraction and implementations |
| `agent/request/` | ChatRequest, VisionRequest, ToolsRequest, etc. | Protocol-specific request construction |
| `agent/mock/` | MockAgent, MockProvider, MockClient | Test infrastructure |

**orchestrate** depends on agent (Hub coordinates Agents). Within the single module, this is simply an intra-package import with no version coordination concerns.

---

## 4. Subsystem Descriptions

### 4.1 core (complete)

**Purpose**: Provide the foundational type vocabulary shared across the entire kernel. Any subsystem that needs to reference a Protocol, Message, Response type, or Config structure depends on core and nothing else.

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

**Integration points**: Every kernel subsystem depends on core for shared types. core depends on nothing except `github.com/google/uuid`.

---

### 4.2 agent (complete)

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

**Anthropic alignment**: Direct API access — Anthropic recommends starting with direct LLM API calls rather than complex frameworks. agent is the thin, direct access layer.

**Integration points**:
- Depends on core for types (Protocol, Message, Response structs, Config)
- Consumed by orchestrate (Hub registers Agents)
- Consumed by the kernel runtime (agentic loop calls Agent.Tools())

---

### 4.3 orchestrate (complete)

**Purpose**: Provide Go-native agent coordination primitives for building multi-agent systems with state management and composable workflow patterns.

**Scope boundary**:
- INCLUDES: Hub messaging, StateGraph execution, State management, workflow patterns (Chain, Parallel, Conditional), Observer interface, CheckpointStore
- EXCLUDES: Agentic loops, tool execution, memory, skills

**Key abstractions**: Unchanged from current implementation.

**Anthropic alignment**: Workflow patterns (prompt chaining, routing, parallelization, orchestrator-workers) map directly to ProcessChain, ProcessConditional, ProcessParallel, and StateGraph.

**Integration points**:
- Depends on agent (Hub uses Agent interface)
- Observer interface consumed by the kernel runtime for event emission
- StateGraph can orchestrate multiple kernel runtimes as nodes

---

### 4.4 memory (skeleton)

**Purpose**: Provide filesystem-based persistent memory for agents. Three tiers: bootstrap memory (persistent identity loaded at startup), working memory (current task scratchpad), and structured notes (organized knowledge artifacts). This is the equivalent of CLAUDE.md and the working memory patterns described in Anthropic's long-running agent harness guidance.

**Scope boundary**:
- INCLUDES: Bootstrap file loading and compilation, working memory read/write/append, structured note organization, filesystem operations for memory management
- EXCLUDES: Conversation history (session), token counting (session), context window management (session), skill loading (skills)

**Key abstractions**:
- Bootstrap — Ordered list of markdown files loaded at startup, compiled into system prompt context
- Working Memory — Filesystem-backed scratchpad the agent reads and writes during task execution
- Structured Notes — Organized filesystem layout for knowledge artifacts (concepts, guides, sessions, reviews)

**Anthropic alignment**:
- Bootstrap memory → CLAUDE.md persistent instructions
- Working memory → Structured note-taking for long-horizon tasks
- Shift handoff documentation from long-running agent harness guidance

**Integration points**:
- Zero internal dependencies (filesystem-only operations)
- Consumed by skills (skill filesystem patterns build on memory filesystem patterns)
- Consumed by the kernel runtime (bootstrap loaded at startup, working memory available during execution)

---

### 4.5 tools (skeleton)

**Purpose**: Provide the tool system for agent-driven execution. Defines the interface for executable tools, a registry for tool management with permissions, and a set of built-in tools for filesystem and system interaction. Implements Anthropic's ACI (Agent-Computer Interface) design principles.

**Scope boundary**:
- INCLUDES: Tool executor interface, tool definition types, tool registry with permissions, built-in tools (read, write, edit, bash, glob, grep), parallel tool execution, tool result types
- EXCLUDES: MCP tool discovery (mcp), agentic loop orchestration (kernel), LLM interaction (agent)

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
- Depends on core for `response.ToolCall` type (dispatching model-requested tool calls)
- Consumed by mcp (MCP-discovered tools wrapped as Tool Executors)
- Consumed by the kernel runtime (registry provides tools to the agentic loop)

---

### 4.6 session (skeleton)

**Purpose**: Provide conversation history management and context window optimization. Maintains ordered message sequences with support for all roles (user, assistant, system, tool), tracks token usage, and applies compaction strategies when conversations exceed the context window.

**Scope boundary**:
- INCLUDES: Message history buffer, multi-role support (including tool call and tool result messages), token counting interface, context window tracking, compaction strategies (truncation, summarization, sliding window)
- EXCLUDES: Persistent memory (memory), skill loading (skills), HTTP communication (agent)

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
- Depends on core for `protocol.Message` type
- Consumed by the kernel runtime (session manages conversation state within the agentic loop)

---

### 4.7 skills (skeleton)

**Purpose**: Provide the progressive disclosure skill system. Discovers SKILL.md files from configured filesystem paths, loads them in three levels (metadata → instructions → resources), and matches relevant skills to prompts. Directly implements the Agent Skills architecture from Claude's platform.

**Scope boundary**:
- INCLUDES: SKILL.md parsing (YAML frontmatter + markdown body), filesystem discovery across configured paths, 3-level progressive loading, trigger-based matching, resource file access
- EXCLUDES: Skill execution (skills provide instructions, the runtime follows them), tool definitions (tools), MCP integration (mcp)

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
- Depends on memory (filesystem patterns for discovering and reading skill files)
- Consumed by the kernel runtime (matched skills injected into context when relevant)

---

### 4.8 mcp (skeleton)

**Purpose**: Provide a Model Context Protocol client for discovering and invoking tools from external MCP servers. Abstracts transport mechanisms (stdio, SSE) and wraps discovered MCP tools as standard tools executors so they integrate seamlessly with the tool registry.

**Scope boundary**:
- INCLUDES: MCP client connection management, transport abstraction (stdio, SSE), tool discovery from MCP servers, wrapping MCP tools as tools executors, MCP handshake and protocol lifecycle
- EXCLUDES: MCP server implementation, tool registry management (tools), built-in tools (tools)

**Key abstractions**:
- MCP Client — Connects to an MCP server, discovers tools, and provides them as executors
- Transport — Interface abstracting communication (stdio for subprocess-based servers, SSE for HTTP-based servers)
- MCP Tool Wrapper — Adapts an MCP tool into a tools ToolExecutor

**Anthropic alignment**:
- MCP integration from Agent SDK: connect to databases, browsers, APIs
- MCP tool format conversion to standard tool definitions
- Multiple MCP servers per agent

**Integration points**:
- Depends on tools for the ToolExecutor interface (wraps MCP tools as executors)
- Consumed by the kernel runtime (discovered tools registered in the tool registry)

---

### 4.9 kernel (skeleton)

**Purpose**: Compose all kernel subsystems into an agent runtime capable of autonomous, tool-driven task execution. Implements the agentic loop, plan mode, environment abstraction (local/container), hooks, and the standard API surface for interfacing with the kernel.

**Scope boundary**:
- INCLUDES: Agentic loop (prompt → reason → tool calls → execute → observe → iterate), plan mode (draft → approve → execute), environment/sandbox abstraction (local filesystem, container), hooks (pre/post tool use, iteration, stop), runtime configuration, standard API for external consumers
- EXCLUDES: Individual tool implementations (tools), MCP transport details (mcp), conversation compaction algorithms (session), skill parsing (skills), LLM communication (agent), workflow orchestration (orchestrate)

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
- Depends on agent (calls Agent.Tools() in the loop)
- Depends on tools (tool registry, permissions, built-in tools)
- Depends on session (conversation history, context management)
- Depends on memory (bootstrap loading, working memory)
- Depends on skills (skill discovery and injection)
- Depends on mcp (external tool discovery)
- Depends on orchestrate (Observer for event emission; StateGraph can orchestrate multiple runtimes)

---

## 5. Existing Infrastructure Problem Set

The following limitations have been identified in the current core and agent codebases. Solutions are intentionally deferred to per-subsystem concept sessions that will be conducted during implementation. Documenting them here ensures they are addressed as the kernel evolves.

### 5.1 core: Agent Methods Create Fresh Messages

The current `Agent` interface methods (`Chat`, `Tools`, etc.) accept a `prompt string` parameter and internally call `initMessages()` to create a fresh message list each time. This design is incompatible with multi-turn conversations where the full history of user messages, assistant responses, tool calls, and tool results must be passed to the model.

**Impact**: The agentic loop in the kernel runtime needs to pass full conversation history through the Agent interface. The current API cannot support this without bypassing the Agent interface entirely and using lower-level request/client packages directly.

**Deferred to**: agent subsystem concept session (the Agent interface must be redesigned to support message history passthrough while maintaining backward compatibility or providing a clean migration path).

### 5.2 core: Tool Type Duplication

Tool definitions exist in two places: `agent.Tool` (used by consumers) and `providers.ToolDefinition` (used internally by providers). The Agent implementation converts between them. With the addition of tools introducing its own canonical tool definition type, this becomes a three-way duplication problem.

**Impact**: Consumers of the tool system need a single, canonical type for defining tools. The current duplication creates confusion about which type to use.

**Deferred to**: tools and agent subsystem concept sessions (establish canonical types and conversion patterns).

### 5.3 core: Message Type Lacks Tool Call/Result Support

`protocol.Message` has only `Role string` and `Content any`. The OpenAI-compatible tool calling protocol requires messages with:
- Role `"assistant"` containing `tool_calls` array
- Role `"tool"` containing `tool_call_id` for correlation

The current Message type cannot represent these roles without using the `Content any` field in non-obvious ways.

**Impact**: session needs to store tool call and tool result messages in conversation history. The kernel runtime needs to construct these messages for the agentic loop.

**Deferred to**: core restructuring (the Message type needs to support tool-related roles and content, or a richer message type hierarchy is needed).

---

## 6. Prioritized Build Order

### Rationale

The build order follows three principles:
1. **Dependencies first**: A subsystem is only built after all its dependencies exist
2. **Foundation before integration**: Subsystems with fewer dependencies are built first
3. **Complete subsystems are available immediately**: core, agent, and orchestrate are already functional within the kernel

### Phase 0 — Complete

**core, agent, orchestrate** — Migrated into the kernel monorepo with co-located tests, import paths updated, all tests passing. The core deconstruction (separating foundational types from LLM client machinery) is complete.

### Phase 1 — Foundation (independent subsystems)

These subsystems can be built in parallel, as they have no dependencies on each other.

**1. memory** — Filesystem-based persistent memory with zero internal dependencies. Bootstrap loading, working memory, structured notes. This is the simplest new subsystem and can serve as a template for the others.

**2. tools** — Tool execution interface, registry, permissions, built-in tools. Depends on core for `response.ToolCall` type only. The built-in tools (read, write, edit, bash, glob, grep) are self-contained filesystem/system operations.

**3. session** — Conversation history management with token tracking and compaction. Depends on core for `protocol.Message` type only. The compaction strategies (truncation, summarization, sliding window) are independent of each other.

### Phase 2 — Integration (builds on Phase 1)

**4. skills** — Progressive disclosure skill system. Depends on memory for filesystem patterns. This can begin after memory is complete.

**5. mcp** — MCP client with transport abstraction. Depends on tools for the ToolExecutor interface. This can begin after tools is complete.

Phase 2 subsystems can also proceed in parallel, as skills depends on memory (not tools) and mcp depends on tools (not memory).

### Phase 3 — Composition

**6. kernel** — Agentic loop composing all subsystems. This is the final subsystem and depends on everything above. By the time kernel development begins, all foundational and integration subsystems will have established their interfaces.

### Build Order Diagram

```
Phase 0:  [core + agent + orchestrate]          (complete)
              |
              v
Phase 1:  [memory]  [tools]  [session]          (parallel)
              |          |
              v          v
Phase 2:  [skills]    [mcp]                     (parallel)
                  \    /
                   v  v
Phase 3:          [kernel]
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

Model capability detection is a kernel runtime concern, not a core concern. The kernel should determine model capabilities through configuration rather than runtime probing. The existing `ModelConfig.Capabilities` map (per-protocol options) can be extended to include capability metadata:

- Whether the model supports tool calling at all
- Whether the model supports parallel tool calls
- Whether the model supports reasoning / chain-of-thought (and whether it can be used simultaneously with tool calling)
- Maximum context window size (for session compaction decisions)
- Whether the model supports vision (for multimodal tool results)

---

## 8. Iterative Development Process

### 8.1 Per-Subsystem Concept Sessions

Each subsystem follows the dev-workflow concept development pattern:

1. **Concept session** (`/dev-workflow concept`): Deep design of the subsystem's architecture, interfaces, and scope boundaries. Produces a concept document at `.claude/context/concepts/`.

2. **Concept → Objectives**: The concept is decomposed into Objectives, each representing a cohesive set of requirements within the phase.

3. **Objectives → Issues**: Each Objective is broken into sub-issues with clear acceptance criteria, each mapping to one branch and one PR.

4. **Task execution sessions** (`/dev-workflow task <issue>`): Individual issues are implemented through structured development sessions with plan mode, implementation, and closeout.

5. **Central document revision**: Upon completing a subsystem, this concept document is revised with lessons learned, interface clarifications, and updated guidance for subsequent subsystems.

### 8.2 Revision Cycle

```
Central Concept (this document)
         |
         v
  Subsystem Concept Session
         |
         v
  Subsystem Implementation
         |
         v
  Revise Central Concept  -----> Next Subsystem Concept Session
```

Each completed subsystem produces knowledge that refines the central vision:
- Interface patterns that emerged during implementation
- Integration challenges that inform adjacent subsystem design
- Scope adjustments based on practical experience
- Updated dependency requirements

### 8.3 Skill Deliverables

The kernel produces two skills:

**Usage skill** — Hosted in the tau plugin at `~/tau/tau-marketplace/plugins/tau/skills/kernel/`
- Covers: subsystem reference documentation (core, agent, orchestrate, memory, tools, session, skills, mcp, kernel)
- Audience: consumers building applications with the kernel
- Format: SKILL.md with references/ directory containing per-subsystem reference files

**Contributing skill** — Embedded in the kernel repository at `.claude/skills/kernel-dev/`
- Covers: architecture overview, package hierarchy, extension patterns, testing strategy
- Audience: contributors to the kernel
- Format: SKILL.md with references/ directory containing per-subsystem architecture documentation

Skills are updated as part of each subsystem's implementation. Reference files are added to both skills as subsystems move from skeleton to complete.

---

## 9. Deployment Vision

The kernel is a closed-loop processing system. External services connect to it through its interface — the kernel has no awareness of what connects to it. Deployment modes differ only in how extensions are configured and where they run; the kernel binary remains the same.

### 9.1 Local Mode

The agent executes directly on the host system. Extensions run locally or in Docker containers alongside the kernel.

- Tool execution: direct subprocess/filesystem calls
- Memory files: host filesystem paths
- Skills: discovered from host filesystem directories
- MCP servers: launched as local subprocesses
- Extensions: local services (e.g., Ollama for LLM, filesystem for persistence)

### 9.2 Container/Sandbox Mode

The agent executes within a container. Extensions connect over the network through the kernel's interface.

- Tool execution: confined to container filesystem and processes
- Memory files: container-local paths (mounted or built-in)
- Skills: baked into the container image or mounted at runtime
- MCP servers: accessed via network transport (SSE)
- Extensions: cloud or self-hosted services connecting through the kernel interface

### 9.3 ConnectRPC Interface

The kernel exposes its interface via **ConnectRPC** (`connectrpc.com/connect`), providing the extensibility boundary through which all external services connect. The kernel has no knowledge of what connects to it.

**Why ConnectRPC**:
- **Go-native**: Uses standard `net/http.Handler` and `http.ServeMux` — no custom transport infrastructure
- **Multi-protocol**: A single service definition serves gRPC, gRPC-Web, and Connect protocol (HTTP/JSON with Protobuf) simultaneously. Browser clients, gRPC clients, and simple HTTP clients all work against the same server
- **Streaming**: Server streaming for token-by-token response output, client streaming for context injection, bidirectional streaming for interactive sessions
- **Interceptors**: Middleware at the interface boundary for auth, validation, logging, metrics — these are extension points, not kernel concerns
- **Protobuf schemas**: Type-safe code generation via `buf generate`; schemas serve as the interface contract
- **Protovalidate**: Request validation via protobuf annotations

**Why ConnectRPC over alternatives**:
- vs gRPC: ConnectRPC uses standard `net/http`, avoiding the separate HTTP/2 transport layer required by `grpc-go`
- vs REST/HTTP: Protobuf schemas provide type safety, native streaming, and code generation that OpenAPI cannot match
- vs Go interfaces alone: Go interfaces define the in-process API between subsystems; ConnectRPC defines the network API for the kernel's external boundary

### 9.4 Extension Architecture

```
External services connect THROUGH the interface.
The kernel has zero awareness of extensions.

    Extensions (external services)
         ├── Persistence    (session state, memory files)
         ├── IAM            (authentication, authorization)
         ├── Container      (sandbox management)
         ├── MCP Gateway    (proxies MCP servers)
         ├── Observability  (metrics, tracing export)
         └── UI             (web interface, CLI)
              |
              |--- ConnectRPC Interface (extensibility boundary)
              |
         TAU Kernel (closed-loop processing)
```

Extensions are analogous to operating system userspace services. The kernel is the core; the ConnectRPC interface is the syscall boundary; extensions are userspace services. This decoupling enables the same kernel to serve as the foundation for embedded agents, desktop agents, server agents, or any other deployment model.

### 9.5 Local Development Environment

For local development, the kernel runs as a Go binary on the host and extensions run as local services:

- **Kernel**: `go run ./cmd/kernel` on the host
- **LLM provider**: Ollama running locally or in Docker
- **Persistence**: Filesystem-backed (no external service needed initially)
- **Docker Compose**: Orchestrates extension services when needed
- **Transition**: Same kernel binary deploys to containers with cloud-hosted extensions in production

---

## 10. Risk Analysis

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Tool-calling reliability varies across models** | Poor agentic loop behavior on weaker models; loops that never terminate or produce garbage tool calls | Model capability detection via configuration; fallback to simpler patterns (prompt chaining instead of agentic loop) for weaker models; clear minimum model requirements in documentation |
| **Context window overflow in long tasks** | Lost context, degraded responses, conversation amnesia | Compaction strategies in session (truncation, summarization, sliding window); working memory offload in memory; sub-agent delegation in kernel runtime |
| **MCP specification stability** | Breaking changes from upstream MCP protocol evolution | Abstract behind Transport interface; pin to specific MCP protocol version; isolate MCP concerns in mcp subsystem so changes don't cascade |
| **Token counting accuracy** | Incorrect compaction timing — either too aggressive (losing context) or too conservative (exceeding window) | TokenCounter interface allows model-specific implementations; conservative default estimates with configurable safety margin; compaction is a strategy pattern that can be swapped |
| **Breaking changes during core evolution** | Existing subsystems break when core types change | All subsystems share a single module and are tested together; `go test ./...` catches breakage immediately; no version coordination needed |
| **CI build times at scale** | Monorepo CI runs all tests even when only one subsystem changed | Go's test caching handles unchanged packages efficiently; CI can be configured with path-based triggers for subsystem-specific workflows if needed |
| **Package boundary erosion** | Without repository walls, packages may develop inappropriate cross-dependencies | Go's import rules enforce acyclic dependencies; `go vet` catches issues; the dependency hierarchy is documented and can be validated with tooling |
| **Version granularity** | Single version means a core-only fix bumps the version for all subsystems | Acceptable tradeoff — the kernel is a single product with a single release. Subsystem-level versioning would reintroduce the cascade complexity this migration eliminated |

---

## 11. Retrospective — Why Monorepo Over Individual Libraries

### What We Tried

9 separate repositories with independent Go modules and versions, coordinated by a `go.work` workspace for local development. Each library had its own CI pipeline, release workflow, CHANGELOG, label taxonomy, CLAUDE.md, settings.json, and versioning scheme. The marketplace plugin maintained 4 separate library skills that all referenced the same architectural patterns.

### What We Learned

**Version cascade friction**: A change to tau-core required cascading through tau-agent and tau-orchestrate — 3+ commits and 3+ tags per atomic change. The release workflow documented multi-step procedures for identifying changed modules, determining per-module dev numbers, tagging in dependency order, cascading go.mod updates, and cleaning up old dev releases per module.

**Diamond dependency risk**: Multiple libraries depending on different versions of tau-core created real risk of version skew during active development. Go modules handle this well in stable ecosystems, but during rapid iteration the coordination overhead was disproportionate.

**Organizational overhead**: 9 repositories meant 9 sets of CI configs, 9 CLAUDE.md files, 9 settings.json files, 9 release workflows, 9 label taxonomies to keep in sync. Each new repository required bootstrapping all of this infrastructure before any code could be written.

**The Go team's own guidance**: "Packages that change together should live together." The Go standard library is a monorepo because those packages have deep interdependencies and need atomic updates. Our packages have the same property — the dependency hierarchy (core → agent → orchestrate → kernel) means changes frequently span multiple layers.

**The kernel analogy proves the point**: If the design inspiration is the Linux kernel (a single codebase with internal subsystems), the implementation should match. Distributing kernel subsystems across separate repositories contradicts the architectural metaphor that inspired the design.

**Only 3 of 9 repos had code**: The timing was ideal for consolidation — 5 repos were skeleton-only, and the pre-release status meant zero external consumers to migrate.

### What the Monorepo Solves

- **Atomic commits**: Changes that span layers (core → agent → orchestrate) are a single commit
- **Single version**: One tag, one release, no cascade — `v<target>-dev.<objective>.<issue>` for dev releases
- **One CI pipeline**: A single workflow validates the entire kernel
- **One CHANGELOG**: Organized by subsystem sections, no cross-repo synchronization
- **One release process**: Tag and push — no dependency ordering, no go.mod cascade updates
- **Package boundaries enforced by Go**: Import rules and the type system prevent inappropriate dependencies — repository walls are unnecessary for enforcement
- **Contributors work in one codebase**: Full context without switching repositories
- **Skills consolidate**: One marketplace skill (`tau:kernel`) with per-subsystem reference files replaces 4 separate library skills

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
| 2026-02-04 | Initial concept document (as tau-runtime) | Concept development session |
| 2026-02-06 | Runtime boundary principle, extension ecosystem, ConnectRPC interface, separate repos with go.work, proof-of-concept validation | Phase 1 preparation session |
| 2026-02-07 | Monorepo migration — rebranded as TAU Kernel, consolidated 9 repos into single module, added retrospective section, updated all terminology and architecture sections | Kernel migration session |
