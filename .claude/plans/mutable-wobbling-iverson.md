# Phase 1 - Foundation: Preparation Session Plan

## Context

The TAU ecosystem has a complete concept document (`concepts/tau-runtime/README.md`) describing 9 libraries that compose into an agent runtime. Two libraries exist (`tau-core` v0.0.1, `tau-orchestrate` v0.0.1). The proof-of-concept (claude-classify-docs) validates the vision — demonstrating that three primitives (memory, skills, subagents) compose into a functional agent runtime capable of executing a non-trivial, multi-stage workflow. This session finalizes the concept, creates all necessary repositories, establishes project management infrastructure, and sets up skills — so that subsequent development sessions can focus purely on implementation.

**Key architectural decision**: The user requires clean import paths (`github.com/tailored-agentic-units/tau-core`, not `github.com/tailored-agentic-units/tau-runtime/tau-core`). Since Go ties module paths to repository paths, **each library remains a separate repository**. A `go.work` workspace at `~/tau/` provides the monorepo-like local development experience.

**Runtime boundary principle**: The runtime is a closed-loop input/output processing system. Its only concern is managing context receipt, processing workflows, and returning results. The runtime has **zero awareness of extensions**. The *interface* to the runtime serves as the extensibility point — external infrastructure connects through this interface, not to the runtime itself.

---

## Deliverable 1: Finalize `concepts/tau-runtime/README.md`

Merge the user's additional concept details and claude-classify-docs validation insights into the existing document.

### Section 1 — Add after "Design Philosophy" (new subsections)

**1.5 Runtime Boundary Principle**
- The runtime is a closed-loop input/output processing system
- Inside the runtime: context receipt → workflow processing → result return
- The runtime has zero awareness of extensions — it does not know or care what connects to it
- The runtime *interface* provides the extensibility points:
  - Bootstrapping a session (context provided from external persistence sources)
  - Sending and receiving additional context in various formats
  - Outputting metadata
  - Administrative / diagnostic / validation features
- Outside the runtime: data persistence, IAM integration, containerization standards, platform-specific concerns
- Linux kernel analogy: the runtime is the kernel; the interface is the syscall boundary; extensions are userspace services that connect through that boundary

**1.6 Extension Ecosystem**
- Each component external to the runtime is a sub-system (like OS components)
- Extensions connect to the runtime through its interface — the runtime never reaches out to extensions
- Local dev: compose runtime with locally-run extensions (Docker Compose)
- Production: transition to cloud/self-hosted extension services
- Extension categories: Persistence, IAM, Container/Sandbox, MCP Gateway, Observability, UI
- This decoupling enables the same runtime to serve as the foundation for embedded agents, desktop agents, server agents, or any other deployment model — just as Linux kernel + extensions = embedded OS, desktop OS, or server OS

### Section 1 — Add new subsection

**1.7 Proof-of-Concept Validation**

The claude-classify-docs implementation validates the core design thesis. Key findings that inform the runtime architecture:

- **Three primitives suffice**: Memory (bootstrap context), skills (capability specifications), and subagents (parallel task delegation) compose into a functional agent runtime. The TAU libraries formalize exactly these: tau-memory, tau-skills, tau-runtime (subagent orchestration).
- **Component mapping is direct**: Every agent-lab component (StateGraph, State nodes, ProcessParallel, system prompts, checkpoints) maps cleanly to a Claude Code feature (orchestrating skill, individual skills, concurrent Tasks, skill content, working directory files). This validates that the TAU library interfaces capture the essential abstractions.
- **Interfaces are the product**: The classify-docs workflow runs identically on any runtime that implements the tau-tools, tau-skills, and tau-memory interfaces. The workflow specification (prompts, graph topology, scoring factors) remains constant — only the execution machinery changes.
- **Filesystem as state**: Working directory files provide implicit checkpointing and state management without a database. This validates tau-memory's filesystem-first approach.
- **Skill orchestration replaces code**: ~11,000 lines of Go infrastructure replaced by ~600 lines of markdown configuration. The runtime's value is enabling this level of capability through standardized, composable primitives.

### Section 3 — Update topology

**3.1 Library Topology**: Add "Repository" column — each library is a separate repository under `github.com/tailored-agentic-units/`

**3.3 Repository Strategy**: New subsection
- Separate repos for clean Go module paths (`github.com/tailored-agentic-units/tau-core`)
- `go.work` at `~/tau/` for local cross-repo development
- Standard Go versioning: `v0.1.0`, `v0.2.0`, etc.
- CI per-repo; integration testing via workspace

### Section 9 — Rewrite deployment vision

**9.3 ConnectRPC Interface** (replaces vague "Standard API")
- Technology: ConnectRPC (connectrpc.com) — Go-native, standard `net/http`
- Multi-protocol: serves gRPC, gRPC-Web, and Connect (HTTP/JSON) from same handler
- Streaming: server-stream for token output, bidirectional for interactive sessions
- Interceptors: auth, validation, logging, metrics — extension points for cross-cutting concerns
- Protobuf schemas define the contract; `buf generate` produces type-safe Go code
- Why ConnectRPC over alternatives:
  - vs gRPC: uses standard `net/http`, no custom transport layer
  - vs REST: protobuf schemas for type safety, native streaming, code generation
  - vs Go interfaces alone: Go interfaces define in-process API; ConnectRPC defines the network API
- The runtime exposes this interface but has no knowledge of what connects to it

**9.4 Extension Architecture**
```
External services connect THROUGH the interface.
The runtime has zero awareness of extensions.

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
         tau-runtime (closed-loop processing)
```

**9.5 Local Development Environment**
- Runtime runs as a Go binary on the host (development mode)
- Extensions run locally or in Docker containers (e.g., Ollama for LLM)
- Docker Compose for local dev orchestration
- Transition to containerized runtime + cloud extensions on deployment

### Section 10 — Update risk table

Add two entries:

| Risk | Impact | Mitigation |
|------|--------|------------|
| Multi-repo coordination overhead | Dependency version drift, release coordination friction across 9 repos | go.work workspace for local dev, shared CI patterns, coordinated releases via GitHub project phases |
| Knowledge fragmentation across repos | Contributors lose context switching between repos; patterns diverge | tau-marketplace plugin provides shared skills and subagents usable across all project boundaries; tau-platform skill provides ecosystem-wide context; dev skills in each repo encode local patterns |

### Appendix B — Add revision entry

`2026-02-06 | Runtime boundary, extension ecosystem, ConnectRPC, separate repos, claude-classify-docs validation | Preparation session`

---

## Deliverable 2: Repository Setup

### 2.1 New Repositories to Create (7)

| Repository | Module Path | Description | Dependencies |
|-----------|------------|-------------|--------------|
| `tau-agent` | `github.com/tailored-agentic-units/tau-agent` | LLM communication layer (extracted from tau-core) | tau-core, uuid |
| `tau-memory` | `github.com/tailored-agentic-units/tau-memory` | Filesystem-based persistent memory | (none) |
| `tau-tools` | `github.com/tailored-agentic-units/tau-tools` | Tool execution interface, registry, permissions | tau-core |
| `tau-session` | `github.com/tailored-agentic-units/tau-session` | Conversation history, context management | tau-core |
| `tau-skills` | `github.com/tailored-agentic-units/tau-skills` | Progressive disclosure skill system | tau-memory |
| `tau-mcp` | `github.com/tailored-agentic-units/tau-mcp` | MCP client with transport abstraction | tau-tools |
| `tau-runtime` | `github.com/tailored-agentic-units/tau-runtime` | Agent runtime: closed-loop agentic processing with ConnectRPC interface | tau-agent, tau-tools, tau-session, tau-memory, tau-skills, tau-mcp, tau-orchestrate, connectrpc |

### 2.2 tau-runtime Repository Structure

```
tau-runtime/
├── go.mod                    (module github.com/tailored-agentic-units/tau-runtime)
├── README.md                 (overview, getting started, architecture)
├── PROJECT.md                (long-term vision and roadmap)
├── .gitignore
├── proto/                    (ConnectRPC protobuf definitions)
│   └── tau/runtime/v1/
│       ├── runtime.proto     (service definition)
│       └── buf.gen.yaml
├── gen/                      (generated ConnectRPC Go code)
│   └── tau/runtime/v1/
│       └── runtimev1connect/
├── pkg/                      (runtime packages)
│   └── runtime/              (stub: agentic loop, composition)
├── cmd/                      (entry points)
│   └── tau-runtime/          (stub: main binary)
├── .claude/
│   ├── CLAUDE.md
│   ├── settings.json
│   ├── skills/
│   │   └── tau-runtime-dev/
│   │       └── SKILL.md
│   └── context/
│       └── concepts/
│           └── tau-runtime.md
└── .github/
    └── workflows/
        └── ci.yml
```

### 2.3 Stub Repositories (tau-memory, tau-tools, tau-session, tau-skills, tau-mcp)

Each stub contains:
```
tau-<name>/
├── go.mod          (module path + dependencies)
├── README.md       (purpose, planned scope, "under development")
├── .gitignore
├── doc.go          (package-level documentation)
└── .claude/
    ├── settings.json
    └── skills/
        └── tau-<name>-dev/
            └── SKILL.md   (minimal dev skill)
```

All stubs are created as part of this preparation session, not tracked as separate issues.

### 2.4 tau-agent Repository

Full structure with code extracted from tau-core:
```
tau-agent/
├── go.mod          (module github.com/tailored-agentic-units/tau-agent)
├── README.md
├── .gitignore
├── pkg/
│   ├── agent/      (Agent interface + impl — from tau-core pkg/agent/)
│   ├── client/     (HTTP client + retry — from tau-core pkg/client/)
│   ├── providers/  (Provider interface, Base, Ollama, Azure — from tau-core pkg/providers/)
│   ├── request/    (Protocol-specific requests — from tau-core pkg/request/)
│   ├── parse/      (Response parsing — extracted from tau-core pkg/response/)
│   └── mock/       (Test mocks — from tau-core pkg/mock/)
├── tests/          (black-box tests mirroring pkg/)
└── .claude/
    ├── settings.json
    └── skills/
        └── tau-agent-dev/
            └── SKILL.md
```

### 2.5 tau-core Restructuring

After extraction, tau-core retains:
```
tau-core/
├── pkg/
│   ├── protocol/   (Protocol enum, Message type)
│   ├── response/   (Struct definitions ONLY — ChatResponse, ToolsResponse, ToolCall, TokenUsage, StreamingChunk, EmbeddingsResponse, AudioResponse)
│   ├── config/     (AgentConfig, ProviderConfig, ModelConfig, ClientConfig, Duration)
│   └── model/      (Model runtime type)
└── tests/
```

**Removed from tau-core**: `pkg/agent/`, `pkg/client/`, `pkg/providers/`, `pkg/request/`, `pkg/mock/`, and all `Parse*` functions from `pkg/response/`.

### 2.6 tau-orchestrate Adjustment

Only import path changes:
- `github.com/tailored-agentic-units/tau-core/pkg/agent` → `github.com/tailored-agentic-units/tau-agent/pkg/agent`
- `go.mod` dependency: tau-core → tau-agent (which transitively brings tau-core)

Files affected: `pkg/hub/hub.go`, `pkg/hub/handler.go`

### 2.7 Local Development Workspace

Create `~/tau/go.work`:
```
go 1.25.2

use (
    ./tau-core
    ./tau-agent
    ./tau-orchestrate
    ./tau-memory
    ./tau-tools
    ./tau-session
    ./tau-skills
    ./tau-mcp
    ./tau-runtime
)
```

This enables cross-repo development without `replace` directives in go.mod files.

### 2.8 Standard Repository Configuration

Every new repo gets:
- Standard `.gitignore` for Go
- Standard labels (cloned from tau-platform): bug, feature, improvement, documentation, refactor, testing, infrastructure
- `.claude/settings.json` with tau-marketplace plugin reference

No LICENSE files at this stage — will be addressed at a later stage of maturity.

### 2.9 tau-runtime PROJECT.md

Derived from the concept document. Contains:
- **Vision**: What the tau-runtime ecosystem is and why it exists
- **Architecture**: The 9-library dependency hierarchy, runtime boundary principle, extension ecosystem
- **Roadmap**:
  - Phase 1 — Foundation: tau-core restructuring, tau-agent extraction, tau-orchestrate adjustment, stub libraries, ConnectRPC proto skeleton, tau-runtime skeleton
  - Phase 2 — Core Libraries: tau-memory, tau-tools, tau-session implementation
  - Phase 3 — Integration: tau-skills, tau-mcp implementation
  - Phase 4 — Composition: tau-runtime agentic loop, plan mode, environment abstraction
  - Phase 5 — Extensions: persistence, container/sandbox, UI
- **ConnectRPC Interface**: Proto service definitions, streaming patterns
- **Local Development**: go.work workspace, Docker Compose

---

## Deliverable 3: ConnectRPC Interface Definition

### 3.1 Technology Validation

**ConnectRPC** (`connectrpc.com/connect`) is validated as the runtime interface technology:
- **Go-native**: Uses standard `net/http.Handler` and `http.ServeMux` — no custom transport
- **Multi-protocol**: Single service serves gRPC, gRPC-Web, and Connect (HTTP/JSON with Protobuf)
- **Streaming**: Server streaming (token output), client streaming (context injection), bidirectional (interactive sessions)
- **Interceptors**: Middleware for auth, validation, logging — clean extension points at the interface boundary (not inside the runtime)
- **Protobuf schemas**: Type-safe code generation via `buf generate`
- **Protovalidate**: Request validation via protobuf annotations
- **Runtime-agnostic**: The interface is the extensibility point; the runtime behind it is a black box

### 3.2 Initial Proto Skeleton

`proto/tau/runtime/v1/runtime.proto` — draft service definition:

```protobuf
syntax = "proto3";
package tau.runtime.v1;

// Session lifecycle — the extensibility boundary
// External services connect to the runtime through this interface.
// The runtime has no awareness of what connects to it.
service RuntimeService {
  // Create a new session with bootstrap context
  rpc CreateSession(CreateSessionRequest) returns (CreateSessionResponse);

  // Submit a prompt and stream back results (agentic loop)
  rpc Run(RunRequest) returns (stream RunEvent);

  // Inject additional context into an active session
  rpc InjectContext(InjectContextRequest) returns (InjectContextResponse);

  // Get session metadata and diagnostics
  rpc GetSession(GetSessionRequest) returns (GetSessionResponse);
}
```

This is marked as **experimental/draft** — the contract evolves as libraries are implemented.

---

## Deliverable 4: Marketplace Skill Adjustments

### 4.1 Strategy

**Rename `tau-overview` → `tau-platform`** — reflects its role as the ecosystem-level context skill for the tau-platform.

**New `tau-runtime` skill** — ecosystem entry point describing:
- Overall architecture and the 9-library hierarchy
- Runtime boundary principle and extension ecosystem
- ConnectRPC interface overview
- How to reference each sub-component skill for detailed API usage
- Getting started with the full runtime

**Updated existing skills**:
- `tau-core` — narrow scope to foundational types only (remove agent/provider content)
- `tau-orchestrate` — update import paths (tau-core → tau-agent for Agent interface)
- `tau-platform` (renamed from tau-overview) — add tau-agent, tau-runtime to skills table; update tau-core description

**New per-library skills** (created as libraries are implemented):
- `tau-agent` — agent creation, protocols, providers, mocks (content moved from tau-core skill)

**Deferred skills** (created with their libraries):
- `tau-memory`, `tau-tools`, `tau-session`, `tau-skills`, `tau-mcp`

### 4.2 Immediate Skill Changes

**Rename `tau-overview` → `tau-platform`** (`~/tau/tau-marketplace/plugins/tau/skills/`):
- Rename directory from `tau-overview/` to `tau-platform/`
- Update SKILL.md: name field `tau-platform`, update description
- Add `tau:tau-agent` and `tau:tau-runtime` to skills table
- Update `tau:tau-core` description to reflect narrowed scope
- Note the go.work workspace pattern for local development

**`tau-core` SKILL.md** (`~/tau/tau-marketplace/plugins/tau/skills/tau-core/`):
- Update description: "Foundational type vocabulary for the TAU ecosystem"
- Remove: agent creation, protocol execution, provider setup, mock testing (→ tau-agent)
- Retain: Protocol enum, Message type, Response structs, Config structures, Model type
- Add: "See tau-agent for LLM interaction and provider setup"

**`tau-orchestrate` SKILL.md** (`~/tau/tau-marketplace/plugins/tau/skills/tau-orchestrate/`):
- Update imports in code examples: `tau-core/pkg/agent` → `tau-agent/pkg/agent`
- Update "Related Skills" references

**New `tau-agent` SKILL.md** (`~/tau/tau-marketplace/plugins/tau/skills/tau-agent/`):
- Content migrated from tau-core skill: installation, agent creation, Chat/Vision/Tools/Embed/Audio usage, provider setup (Ollama, Azure), mock testing
- Updated import paths: `github.com/tailored-agentic-units/tau-agent/pkg/...`

**New `tau-runtime` SKILL.md** (`~/tau/tau-marketplace/plugins/tau/skills/tau-runtime/`):
- Architecture overview with dependency diagram
- Runtime boundary principle (closed-loop, zero extension awareness)
- ConnectRPC interface and proto definitions
- Extension ecosystem description
- References to sub-component skills
- Getting started guide

### 4.3 Dev Skills in Repositories

Each repository contains its own dev skill at `.claude/skills/tau-<name>-dev/SKILL.md`:
- `tau-core-dev` — update for narrowed scope
- `tau-agent-dev` — new, covers extracted agent layer
- `tau-orchestrate-dev` — update import references
- `tau-runtime-dev` — new, covers runtime architecture, ConnectRPC patterns

---

## Deliverable 5: GitHub Project Management Adjustments

### 5.1 Reclassify Existing Issues → Backlog

Move all 6 open items to the Backlog phase in the TAU Platform project and remove them from the "Phase 1 - Foundation" milestone on their respective repos:

| Issue | Repo | Current Phase | New Phase |
|-------|------|--------------|-----------|
| Whisper provider (#4) | tau-core | Phase 1 - Foundation | Backlog |
| Trace correlation (#1) | tau-orchestrate | Phase 1 - Foundation | Backlog |
| Decision logging (#2) | tau-orchestrate | Phase 1 - Foundation | Backlog |
| Metrics aggregation (#3) | tau-orchestrate | Phase 1 - Foundation | Backlog |
| OTel observer (#4) | tau-orchestrate | Phase 1 - Foundation | Backlog |
| Design questions (#5) | tau-orchestrate | Phase 1 - Foundation | Backlog |

### 5.2 Link Repos to TAU Platform Project

Link all new repositories to the TAU Platform project:
```bash
gh project link 1 --owner tailored-agentic-units --repo tailored-agentic-units/tau-runtime
gh project link 1 --owner tailored-agentic-units --repo tailored-agentic-units/tau-agent
# ... (all new repos)
```

### 5.3 Create Phase 1 Issues on tau-runtime

Issues focus on **development work**, not infrastructure setup (stubs and repos are created in this session):

| Issue Title | Labels | Description |
|-------------|--------|-------------|
| Restructure tau-core as foundational types | refactor | Remove agent/client/provider/request/mock/parse packages from tau-core, keeping only type definitions |
| Extract tau-agent from tau-core | refactor | Populate tau-agent repo with agent, client, providers, request, parse, and mock packages extracted from tau-core |
| Adjust tau-orchestrate imports for tau-agent | refactor | Update hub dependency from tau-core to tau-agent, adjust all import paths |
| Define ConnectRPC proto for runtime interface | feature | Design and implement proto/tau/runtime/v1/runtime.proto with session lifecycle RPCs, buf config, generated Go code |
| Update marketplace skills for ecosystem restructuring | documentation | tau-core narrowing, tau-orchestrate updates, tau-overview → tau-platform rename, new tau-agent and tau-runtime skills |
| Tag tau-core v0.1.0 and tau-orchestrate v0.1.0 post-restructuring | infrastructure | Release restructured libraries with new version tags |

### 5.4 Create Milestones

On `tau-runtime` repo:
- **Phase 1 - Foundation**: Code migration, restructuring, ConnectRPC skeleton
- Assign all Phase 1 issues to this milestone

### 5.5 Add All Phase 1 Issues to TAU Platform Project

Each issue added to the project with:
- Status: Todo
- Phase: Phase 1 - Foundation

---

## Execution Sequence

```
Step 1: Finalize concept document
         └─ Edit concepts/tau-runtime/README.md with new sections
             (runtime boundary, extension ecosystem, claude-classify-docs
              validation, ConnectRPC, repo strategy)

Step 2: Create repositories and stubs (can be parallelized)
         ├─ Create tau-runtime repo with full structure (PROJECT.md, proto skeleton, .claude config, CI)
         ├─ Create tau-agent repo (empty initially, populated in Step 3)
         ├─ Create stub repos (tau-memory, tau-tools, tau-session, tau-skills, tau-mcp)
         ├─ Set up standard labels on all new repos
         └─ Create ~/tau/go.work workspace

Step 3: Code migration (sequential — each step depends on previous)
         ├─ Extract tau-agent code from tau-core
         ├─ Restructure tau-core (remove extracted packages)
         ├─ Adjust tau-orchestrate imports
         └─ Verify all repos build and pass tests

Step 4: ConnectRPC setup
         ├─ Create proto definitions in tau-runtime
         ├─ Set up buf configuration
         └─ Generate Go code

Step 5: GitHub project management
         ├─ Reclassify existing issues → Backlog
         ├─ Link new repos to TAU Platform project
         ├─ Create Phase 1 development issues on tau-runtime
         ├─ Create milestone on tau-runtime
         └─ Add issues to project board

Step 6: Marketplace skills
         ├─ Rename tau-overview → tau-platform
         ├─ Update tau-core, tau-orchestrate skills
         ├─ Create tau-agent skill
         └─ Create tau-runtime skill

Step 7: Tag releases
         ├─ Tag tau-core v0.1.0 (restructured scope)
         ├─ Tag tau-agent v0.0.1 (initial release)
         └─ Tag tau-orchestrate v0.1.0 (adjusted imports)
```

Steps 2's sub-tasks can run in parallel. Steps 3-4 are sequential. Steps 5-6 can run in parallel after Step 4.

---

## Verification

After all deliverables are complete:

1. **Build verification**: Each repo builds independently (`go build ./...`)
2. **Test verification**: All existing tests pass after migration (`go test ./...`)
3. **Workspace verification**: `go work sync` succeeds at `~/tau/`; cross-repo imports resolve locally
4. **Import path verification**: `go get github.com/tailored-agentic-units/tau-core@v0.1.0` works (after tagging)
5. **Project board verification**: All Phase 1 issues visible on TAU Platform project, Backlog items correctly reclassified
6. **Skill verification**: Skills load correctly when referenced in Claude Code sessions
7. **Proto verification**: `buf lint` passes on proto definitions; generated Go code compiles

---

## Key Files

| File | Purpose |
|------|---------|
| `/home/jaime/tau/tau-platform/concepts/tau-runtime/README.md` | Concept document to finalize |
| `/home/jaime/tau/tau-platform/concepts/tau-runtime/claude-classify-docs/spec.md` | PoC spec — validation insights for concept doc |
| `/home/jaime/tau/tau-core/go.mod` | Existing module: `github.com/tailored-agentic-units/tau-core` |
| `/home/jaime/tau/tau-core/pkg/agent/agent.go` | Agent interface to extract into tau-agent |
| `/home/jaime/tau/tau-core/pkg/response/` | Parse functions to extract; struct definitions to keep |
| `/home/jaime/tau/tau-orchestrate/pkg/hub/hub.go` | Hub import of agent.Agent — changes to tau-agent |
| `/home/jaime/tau/tau-orchestrate/pkg/hub/handler.go` | MessageContext with agent.Agent — changes to tau-agent |
| `/home/jaime/tau/tau-marketplace/plugins/tau/skills/tau-core/SKILL.md` | Skill to narrow scope |
| `/home/jaime/tau/tau-marketplace/plugins/tau/skills/tau-orchestrate/SKILL.md` | Skill to update imports |
| `/home/jaime/tau/tau-marketplace/plugins/tau/skills/tau-overview/SKILL.md` | Skill to rename → tau-platform |
