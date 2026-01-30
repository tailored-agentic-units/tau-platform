# Ontology: Common Vocabulary for the TAU Platform

**Category:** Architecture

---

> **Note:** This discussion is intentionally published last. Resolutions from the other topic discussions -- web service architecture, project management, iterative development, and Claude Code skills -- will have a significant impact on the vocabulary we ultimately adopt. Expect this ontology to evolve as those discussions conclude.

A shared vocabulary is essential for cross-repo development. When we say "agent" or "service" or "tool," everyone on the team should have the same mental model. This discussion establishes the initial common vocabulary for the TAU Platform ecosystem.

### Guiding Principle: Align with Established Disciplines

Wherever possible, our vocabulary should match the idiomatic terminology of the disciplines we build on -- LLM engineering, distributed systems, web service architecture, information retrieval, and Go software design. We shouldn't invent new terms when established ones exist and carry the right meaning. Novel terminology should be reserved for concepts that are genuinely specific to the TAU Platform.

### Two Vocabularies

This ontology covers two distinct vocabularies:

1. **TAU Platform vocabulary** -- Terms specific to our platform, libraries, and the concepts they implement. This is what we're building.
2. **Claude Code vocabulary** -- Terms from Claude Code and its ecosystem of tooling (skills, plans, plugins). This is how we develop and coordinate.

Both are documented here because they overlap in daily use, but they serve different purposes and evolve independently.

---

These definitions are drawn from existing implementations across tau-core, go-agents-orchestration, document-context, agent-lab, and our planning artifacts.

## TAU Platform Vocabulary

### Platform Layer

| Term | Definition |
|------|-----------|
| **Ecosystem** | The full set of TAU libraries, services, and tooling |
| **Library** | A Go module with a focused responsibility, consumed as a dependency |
| **Service** | A deployed application composed from libraries (e.g., Sophia, agent-lab) |

### Agent Primitives (tau-core)

| Term | Definition |
|------|-----------|
| **Agent** | A configured runtime that can execute LLM protocols against a specific provider and model. Created via `agent.New()` with a config and options. |
| **Protocol** | A category of LLM interaction: Chat, Vision, Tools, Embeddings, Audio. Each protocol defines its own request/response types. |
| **Provider** | An LLM service backend (Ollama, Azure, Whisper). Providers implement the `Provider` interface and handle protocol-specific request marshaling. |
| **Model** | A runtime type binding a provider to a specific model name (e.g., `llama3.2` on Ollama). Contains the resolved provider instance. |
| **Client** | The HTTP execution layer that sends marshaled requests to providers and returns raw responses. |

### Orchestration (tau-orchestrate)

| Term | Definition |
|------|-----------|
| **Hub** | A coordination point for inter-agent messaging. Supports send, request/response, broadcast, and pub/sub patterns. |
| **State Graph** | A LangGraph-inspired execution model with nodes, edges, and state transitions. Supports checkpointing and replay. |
| **Workflow** | A composed execution pattern: sequential chains, parallel execution, or conditional routing across agents. |
| **Chain** | A sequential pipeline where the output of one step feeds into the next. |
| **Message** | A typed communication unit between agents, carrying payload and metadata through a hub. |

### Data & RAG (planned: tau-data, tau-rag)

| Term | Definition |
|------|-----------|
| **Chunk** | A segment of text produced by splitting a document, sized for LLM consumption (configurable size and overlap). |
| **Entity** | A structured data element extracted from text via pattern matching (emails, phones, dates, coordinates, etc.). |
| **Embedding** | A vector representation of text produced by an LLM embedding model. Used for semantic similarity search. |
| **Retrieval** | The process of finding relevant documents/chunks given a query, typically via hybrid search (vector + sparse + entity). |
| **Context** | The assembled set of relevant information (retrieved chunks, entities, metadata) provided to an LLM alongside a prompt. |

### Infrastructure (planned: service-kit)

> **Note:** This is just an initial set of terms derived from my agent-lab project. This will change drastically based on the standard we align on in the [Web Service Architecture Discussion]().

| Term | Definition |
|------|-----------|
| **Module** | An isolated HTTP sub-application with its own prefix, router, and middleware stack (e.g., `/api`, `/app`). |
| **System** | A domain's business logic interface. Exposes operations (CRUD, execution) and provides handlers for HTTP binding. |
| **Handler** | The HTTP translation layer for a system -- parses requests, calls system methods, formats responses. |
| **Repository** | The data access layer for a domain -- encapsulates database queries, transactions, and persistence. |
| **Lifecycle** | The startup/shutdown coordination mechanism. Systems register hooks that execute during application start and graceful shutdown. |

## Claude Code Vocabulary

These terms belong to Claude Code and its ecosystem of tooling. We use them daily for development and coordination, but they are not TAU Platform concepts.

| Term | Definition |
|------|-----------|
| **Skill** | A Claude Code instruction set (SKILL.md) that provides domain-specific knowledge and patterns. Auto-triggered by context matching. |
| **Plugin** | A distributable collection of Claude Code skills and other Claude context management resources, installable across projects. Used for organization-wide standards. |
| **Plan** | A session continuity file in `.claude/plans/` that captures context, decisions, and next steps for resuming work. |
| **Frontmatter** | The YAML metadata block in a SKILL.md that controls trigger conditions, permissions, and invocation behavior. |
