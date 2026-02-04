A comprehensive concept document has been developed for the **TAU Agent Runtime** -- a vision for expanding the TAU ecosystem from static workflow execution into a full agent runtime capable of autonomous, tool-driven task execution.

> **[Read the full concept document here.](https://github.com/tailored-agentic-units/tau-platform/blob/f42ca0cfdc69ce872bbee607ca6885bc65b2316d/concepts/tau-runtime/README.md)**

## Background

This concept was developed through a deep research and planning session that included:

- **Full codebase analysis** of tau-core and tau-orchestrate to understand current capabilities and limitations
- **Reference analysis of agent-lab** (a predecessor web service) to understand production integration patterns and infrastructure gaps
- **Comprehensive review of 7 Anthropic engineering publications**, including [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents), [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents), the [Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview), and [Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) documentation, as well as [Prompt Engineering](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview) guidance
- **Model compatibility research** across Ollama (self-hosted) and Azure AI Foundry (cloud-hosted) platforms, evaluating tool calling and reasoning capabilities

Each Anthropic insight is mapped to the specific TAU library it informs, ensuring the architecture is grounded in proven patterns rather than speculation.

---

## Vision

The TAU ecosystem currently provides two layers of capability:

- **tau-core** -- A platform and model-agnostic Go library providing LLM interaction primitives across five protocols (Chat, Vision, Tools, Embeddings, Audio) with pluggable providers (Ollama, Azure).
- **tau-orchestrate** -- Go-native agent coordination primitives built on tau-core, providing Hub-based messaging, LangGraph-inspired state management, composable workflow patterns (Chain, Parallel, Conditional), and checkpoint-based resume.

These libraries are powerful for **static, compile-time workflow execution** -- but there is no capability for an agent to dynamically organize its own workflow, select and execute tools based on environmental feedback, maintain persistent memory, discover modular skills, or integrate with external systems via MCP.

The gap is an **agent runtime** -- the middle layer that transforms a single-turn `Agent.Tools()` call into a sustained, autonomous agentic loop. As Anthropic's guidance states: *"Agents are typically just LLMs using tools based on environmental feedback in a loop."*

### Design Philosophy

> **Simple, composable patterns over complex frameworks.**

Each library is independently useful, testable, and comprehensible. Start with the minimum viable abstraction at each layer. Only increase complexity when it demonstrably improves outcomes.

---

## Ecosystem Architecture

The target state is a modular ecosystem of **9 independent Go libraries**, each encapsulating a distinct capability domain:

| Library | Domain | Depends On |
|---------|--------|------------|
| **tau-core** (restructured) | Foundational types: Protocol, Message, Response structs, Config, Model | uuid |
| **tau-agent** (new, extracted) | LLM client: Agent, Client, Provider, Request, Parse, Mock | tau-core |
| **tau-orchestrate** (adjusted) | Coordination: Hub, State, Workflows, Observability, Checkpoint | tau-agent |
| **tau-memory** (new) | Persistent memory: bootstrap loading, working memory, structured notes | (none) |
| **tau-tools** (new) | Tool system: execution interface, registry, permissions, built-in tools | tau-core |
| **tau-session** (new) | Conversation management: message history, context window, compaction | tau-core |
| **tau-skills** (new) | Progressive disclosure: SKILL.md discovery, loading, matching | tau-memory |
| **tau-mcp** (new) | MCP client: transport abstraction, tool discovery, stdio/SSE | tau-tools |
| **tau-runtime** (new) | Agent runtime: agentic loop, plan mode, environment, composition | all above |

Key properties: **acyclic** dependencies, **shallow** hierarchy (max depth 3), **independent foundations** (tau-memory has zero TAU dependencies), and **clean separation** (each library owns a single domain).

### tau-core Deconstruction

tau-core currently bundles foundational types with LLM client machinery. Libraries like tau-tools only need type definitions (e.g., `response.ToolCall`) but would pull in the entire HTTP client and provider system. The restructuring splits tau-core into foundational vocabulary types and extracts LLM client functionality into a new **tau-agent** library.

---

## Key Capabilities

The runtime enables:

- **Agentic Loop** -- Self-organizing workflow sequences driven by the model's reasoning (prompt, reason, tool calls, execute, observe, iterate)
- **Plan Mode** -- Structured planning with markdown plan files and a draft/approve/execute lifecycle
- **Tool System** -- Secure, configurable tools with permissions (allowlist/denylist), built-in tools (read, write, edit, bash, glob, grep), and parallel tool execution
- **Persistent Memory** -- Bootstrap memory loaded into context at startup, working memory as a filesystem-backed scratchpad, and structured notes for knowledge artifacts
- **Skills** -- Progressive disclosure skill system with 3-level loading (metadata ~100 tokens, instructions <5k, resources unlimited) for token-efficient capability extension
- **MCP Integration** -- Model Context Protocol client for discovering and invoking tools from external MCP servers via stdio and SSE transports
- **Deployment Flexibility** -- Local environment or container sandbox, with a standard API surface that is deployment-mode agnostic

---

## Prioritized Build Order

The build follows three principles: dependencies first, foundation before integration, restructure before extend.

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

- **Tier 0** -- Restructure existing infrastructure. Extract LLM client machinery from tau-core into tau-agent. Update tau-orchestrate imports.
- **Tier 1** -- Build foundation libraries in parallel (no inter-dependencies): tau-memory, tau-tools, tau-session.
- **Tier 2** -- Build integration libraries in parallel: tau-skills (depends on tau-memory), tau-mcp (depends on tau-tools).
- **Tier 3** -- Compose everything into tau-runtime: agentic loop, plan mode, environment abstraction, standard API.

---

## Anthropic Reference Alignment

The architecture is grounded in Anthropic's published engineering guidance:

| Source | Key Insight | TAU Library |
|--------|------------|-------------|
| [Building Effective Agents](https://www.anthropic.com/engineering/building-effective-agents) | Agentic loop pattern; Agent-Computer Interface (ACI) design principles | tau-runtime, tau-tools |
| [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | Shift handoff documentation; progress tracking | tau-memory |
| Context Engineering guidance | Finite context resource; compaction strategies | tau-session |
| [Tool Use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) | Tool lifecycle; parallel tool calls | tau-tools |
| [Agent Skills](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) | Progressive disclosure; SKILL.md format | tau-skills |
| [Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) | Built-in tools; subagents; sessions; MCP; hooks | tau-runtime |

---

## Model Compatibility

The agent runtime must work across platforms available at all classification levels. Both **tool calling** and **reasoning** are critical model capabilities -- tool calling drives the agentic loop, while reasoning enables effective tool selection, result interpretation, error recovery, and multi-step planning.

### Ollama (Self-Hosted)

Models supporting both tool calling and reasoning are the strongest candidates:

| Model | Parameters | Tool Calling | Reasoning | Notes |
|-------|-----------|-------------|-----------|-------|
| **Qwen 3** | 0.6B - 235B | Strong | Strong | Hybrid think/no_think modes. Recommended starting point for local development |
| **GPT-OSS** (OpenAI) | 20B, 120B | Strong | Strong | Adjustable reasoning effort levels. 20B runs on 16GB devices |
| **Kimi K2 Thinking** (Moonshot AI) | MoE (~32B active) | Excellent | Excellent | Best-in-class agentic capability; 200-300 sequential tool calls |
| **DeepSeek-V3.1** | 671B MoE | Strong | Strong | Hybrid thinking/non-thinking modes. Requires significant hardware |
| **Devstral Small 2** (Mistral) | 24B | Strong | Limited | 68% on SWE-Bench Verified. Strong for software engineering agents |
| **Nemotron-3-Nano** (NVIDIA) | 30B | Strong | Strong | Designed as an efficient agentic model |
| **GLM-4.7-Flash** | 30B | Strong | Strong | Lightweight deployment, good balance of performance and efficiency |

**Recommendation**: Qwen 3 (8B or larger) or GPT-OSS 20B for development and testing.

### Azure AI Foundry (Cloud-Hosted)

| Model Family | Tool Calling | Reasoning | Notes |
|-------------|-------------|-----------|-------|
| **GPT-5.2** | Excellent | Excellent | Freeform tool calling. 272K context. State-of-the-art reasoning |
| **Claude Opus 4.5 / Sonnet 4.5** | Excellent | Excellent | Full tool suite including computer use, code execution, web search |
| **Grok 4** (xAI) | Excellent | Excellent | First-principles reasoning with native tool use |
| **DeepSeek-V3.2** | Excellent | Excellent | Thinking Retention Mechanism preserves reasoning across tool calls |
| **o3 / o4-mini** | Good | Excellent | Native function calling within chain-of-thought |
| **Mistral Large 3** | Excellent | Very Good | 673B MoE. Best open-weight option for multi-tool orchestration |
| **Phi-4-reasoning** (Microsoft) | Moderate | Very Good | Best small model option for constrained infrastructure |

---

## Discussion Points

This concept represents a significant expansion of the TAU ecosystem. Your input would be valuable on the following:

1. **Architecture** -- Does the 9-library decomposition strike the right balance between modularity and complexity? Are there libraries that should be merged or further split?

2. **Build Order** -- Does the tiered approach make sense? Are there dependencies or risks that suggest a different sequencing?

3. **tau-core Deconstruction** -- The restructuring of tau-core and extraction of tau-agent is a breaking change. What concerns or considerations should we address before proceeding?

4. **Capability Priorities** -- Of the key capabilities (agentic loop, tools, memory, skills, MCP, plan mode), which are most critical to deliver first? Are there capabilities missing from this vision?

5. **Deployment Model** -- The local vs. container deployment modes with a standard API -- does this align with how you envision using the runtime?

6. **Model Support** -- Are there additional models or platforms we should be considering for compatibility?

7. **Web Service Viability** -- It is my belief that we cannot build an effective web service on the TAU ecosystem until these underlying capabilities are in place. The current infrastructure supports static workflow execution, but a production web service needs dynamic tool execution, persistent memory, context management, and the other capabilities outlined here. The agent runtime is the foundation that makes a TAU-powered web service viable. Do you agree with this assessment, or do you see a path to an effective web service with the current infrastructure?

Looking forward to hearing your thoughts on the vision and direction.
