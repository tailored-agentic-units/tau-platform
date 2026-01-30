# Ecosystem Vision: A Recommendation for Architectural Alignment

## Current State

### Sophia (Organization's Repository)
A document RAG application with:
- Go backend (Fiber) + React frontend
- Tightly coupled Ollama integration
- Embedded document processing, entity extraction, vector storage
- All infrastructure co-located in a single repository

### Existing Library Ecosystem
I've been developing a set of modular Go libraries:

| Library | Purpose | Status |
|---------|---------|--------|
| [go-agents](https://github.com/JaimeStill/go-agents) | LLM protocol execution (Chat, Vision, Tools, Embeddings) | v0.3.0 |
| [go-agents-orchestration](https://github.com/JaimeStill/go-agents-orchestration) | Multi-agent coordination, workflows, state management | v0.3.0 |
| [document-context](https://github.com/JaimeStill/document-context) | PDF rendering for LLM vision consumption | v0.1.0 |
| [agent-lab](https://github.com/JaimeStill/agent-lab) | Reference web service platform | Active development |

---

## Proposed Vision

### Principle: Composition Over Monolith

Rather than building self-contained applications, we build applications from composable libraries. Each library has a focused responsibility and clean interfaces.

### Library Ecosystem (Consolidated)

After analyzing Sophia's extractable components, I recommend consolidating into **5 core libraries**, a **web infrastructure kit**, and a **Claude Code plugin**.

**Naming Convention Note**: The current `go-agents` / `go-agents-orchestration` naming is verbose. Under a new GitHub organization, we have an opportunity to establish shorter, more consistent names. This document uses placeholders until we align:

- **`<org>`** - GitHub organization name (to be determined)
- **`<prefix>`** - Library name prefix (options discussed in [Organization and Branding](#organization-and-branding))
- **`service-kit`** and **`claude-plugin`** - Standalone names that don't require the prefix

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│         (Sophia, agent-lab, future applications)            │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│   <prefix>-rag  │ │ <prefix>-data   │ │  service-kit    │
│                 │ │                 │ │                 │
│ - Hybrid search │ │ - Chunking      │ │ - HTTP routing  │
│ - Retrieval     │ │ - Parsing       │ │ - Middleware    │
│ - Vector store  │ │ - Extraction    │ │ - Database      │
│ - Context build │ │ - Caching       │ │ - Lifecycle     │
└────────┬────────┘ └────────┬────────┘ └─────────────────┘
         │                   │
         └─────────┬─────────┘
                   ▼
     ┌─────────────────────────┐
     │  <prefix>-orchestrate   │
     │                         │
     │  - Hub/messaging        │
     │  - State graphs         │
     │  - Workflow patterns    │
     │  - Observability        │
     └────────────┬────────────┘
                  │
                  ▼
        ┌─────────────────┐
        │  <prefix>-core  │
        │                 │
        │  - Agent API    │
        │  - Providers    │
        │  - Protocols    │
        │  - Streaming    │
        └─────────────────┘
```

### Library Descriptions

#### 1. `<prefix>-core` (currently go-agents)
**Purpose**: LLM protocol execution - the foundation for all agent interactions.

**Provides**:
- Agent interface with `Chat()`, `ChatStream()`, `Vision()`, `Embed()`, `Tools()`, `Audio()`
- Provider abstraction (Ollama, Azure, Whisper, extensible)
- Protocol-centric design (Chat, Vision, Tools, Embeddings, **Audio**)
- Retry logic, health tracking, streaming via channels

**Planned Enhancement - Audio Protocol**:
- New `Audio` protocol for speech-to-text transcription
- `WhisperProvider` implementation (multipart form data handling)
- `Audio(ctx, audioData, filename, opts...)` method on Agent interface
- Enables unified LLM + transcription workflows
- ~700 LOC addition, medium complexity (multipart encoding is non-standard)

**Dependencies**: `github.com/google/uuid` library

#### 2. `<prefix>-orchestrate` (currently go-agents-orchestration)
**Purpose**: Multi-agent coordination and workflow patterns.

**Provides**:
- Hub for inter-agent messaging (Send, Request, Broadcast, Pub/Sub)
- State graphs (LangGraph-inspired) with checkpointing
- Workflow patterns (ProcessChain, ProcessParallel, ProcessConditional)
- Observability framework

**Dependencies**: `<prefix>-core`, `github.com/google/uuid`

#### 3. `<prefix>-data` (NEW - extracted from Sophia)
**Purpose**: Data preparation for LLM consumption.

**Provides**:
- Text chunking (sentence-aware, configurable size/overlap)
- Structured data parsing (CSV, JSON, XLSX → typed rows)
- Entity extraction (20+ patterns: emails, phones, SSNs, coordinates, dates)
- Confidence scoring and normalization
- MGRS coordinate parsing (military grid reference)
- Embedding cache (TTL-based, thread-safe)

**Dependencies**: stdlib (optional: encoding libraries for XLSX support)

**Source**: Sophia's `internal/extraction/`, `internal/document/chunker.go`, `internal/document/*_parser.go`, `internal/cache/`

#### 4. `<prefix>-rag` (NEW - extracted from Sophia)
**Purpose**: Retrieval-Augmented Generation patterns with pluggable search providers.

**Provides**:
- Search provider interfaces (vector, entity, sparse stores)
- Hybrid retrieval (vector + sparse + entity matching)
- Context building and relevance scoring

**Provider Architecture**:
The library defines core interfaces that all search providers must implement, with progressive enhancements available for platform-specific capabilities:

| Capability | Type | Description |
|------------|------|-------------|
| Vector search (kNN) | Core | Required for all providers |
| Entity indexing | Core | Structured entity storage and retrieval |
| Text/numeric filtering | Core | Basic field filtering |
| Geo queries (bounding box, distance) | Progressive | Elasticsearch, OpenSearch |
| Typed sparse index | Progressive | Elasticsearch, OpenSearch |
| Aggregations | Progressive | Platform-specific analytics |

**Planned Providers**:
- Elasticsearch (full feature support)
- OpenSearch (full feature support)
- Additional providers can be added by implementing core interfaces

**Dependencies**: `<prefix>-core` (for embedding generation)

**Source**: Sophia's `internal/storage/`, `internal/rag/`

#### 5. `document-context` (exists)
**Purpose**: Document rendering for LLM vision APIs.

**Provides**:
- PDF page extraction and image rendering
- Image enhancement filters
- Base64 encoding for vision APIs
- Persistent caching

**Dependencies**: `github.com/pdfcpu/pdfcpu` (PDF metadata), ImageMagick external binary (rendering)

#### 6. `service-kit` (NEW - extracted from agent-lab)
**Purpose**: Web service infrastructure patterns.

**Provides**:
- Lifecycle coordination (startup/shutdown orchestration)
- Modular HTTP routing with middleware
- Database management (PostgreSQL connection pooling)
- Repository helpers (generic queries, transactions)
- Query builder (fluent SQL construction)
- Pagination utilities
- OpenAPI generation
- Configuration management (TOML loading, environment overrides)

**Dependencies**:
- `github.com/pelletier/go-toml/v2` (configuration parsing)
- `github.com/docker/go-units` (size/duration parsing)
- Database driver (e.g., `github.com/jackc/pgx/v5`)

**Prerequisite Refactoring**:
agent-lab currently has a package dependency violation: `pkg/runtime/infrastructure.go` depends on `internal/config`. Before extraction, this requires:
1. Extract universal configuration infrastructure into `pkg/config`
2. Keep `internal/config` focused on server-specific implementation details
3. Update `pkg/runtime` to depend on `pkg/config` instead

**Source**: agent-lab's `pkg/` directory

#### 7. `claude-plugin` (NEW - Organizational Context)
**Purpose**: Standardized Claude Code plugin providing organizational skills and patterns.

A **Claude Code plugin** that provides standardized skills, patterns, and conventions across all projects. Serves as the foundation for AI-assisted development within the organization.

**Repository**: `github.com/<org>/claude-plugin`

**Plugin Structure**:
```
claude-plugin/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest
├── skills/                      # Auto-triggered by description matching
│   ├── lca/SKILL.md             # Layered Composition Architecture
│   ├── go-patterns/SKILL.md     # Go code organization & error handling
│   ├── go-testing/SKILL.md      # Testing patterns (black-box, table-driven)
│   ├── go-database/SKILL.md     # Repository & query patterns
│   ├── go-http/SKILL.md         # HTTP handler & routing patterns
│   ├── go-storage/SKILL.md      # Blob storage patterns
│   ├── openapi/SKILL.md         # OpenAPI specification patterns
│   ├── workflow/SKILL.md        # State graph & orchestration patterns
│   ├── core-library/SKILL.md    # <prefix>-core usage patterns
│   ├── orchestrate/SKILL.md     # <prefix>-orchestrate usage patterns
│   ├── data-library/SKILL.md    # <prefix>-data usage patterns
│   ├── rag-library/SKILL.md     # <prefix>-rag usage patterns
│   └── methodology/SKILL.md     # Development session workflow
├── agents/                      # Custom subagent definitions
│   ├── explore.md
│   └── plan.md
└── README.md
```

**Key Design Principles**:
- **Automatic triggering**: Skills use optimized descriptions with "REQUIRED for...", "Use when...", and "Triggers:" patterns
- **No manual invocation needed**: Claude detects context and loads relevant skills automatically
- **Organizational standards**: Codifies patterns consistent across all projects
- **Extensible**: Projects add their own `.claude/skills/` for project-specific patterns

**Skill Description Pattern** (for automatic triggering):
```yaml
---
name: go-database
description: >
  REQUIRED for database access patterns. Use when writing repositories,
  building queries, implementing pagination, or handling transactions.
  Triggers: repository.go, sql.DB, query.Builder, ProjectionMap,
  QueryOne, QueryMany, WithTx, pagination, "database query".
---
```

**Installation**:
```bash
# Via marketplace (once published)
/plugin install <org>/claude-plugin

# Or during development
claude --plugin-dir /path/to/claude-plugin
```

**Source**: Patterns extracted and optimized from agent-lab's `.claude/` system

---

## Organization and Branding

### GitHub Organization

We need to establish a GitHub organization to house these libraries. The organization name should be:
- Unique and available on GitHub
- Short enough for practical import paths
- Memorable and aligned with the project's purpose

The organization name is represented as `<org>` throughout this document until we align on a final choice.

### Naming Convention Options

The current `go-agents` / `go-agents-orchestration` naming works but is verbose. Under a new organization, we have options:

**Option A: Keep Current Pattern**
```
github.com/<org>/go-agents
github.com/<org>/go-agents-orchestration
github.com/<org>/go-agents-data
github.com/<org>/go-agents-rag
```
- Pro: Familiar, descriptive
- Con: Verbose package names

**Option B: Short Prefix (shadow-)**

> Note: the program name that my efforts are referenced under is named shadow clone, hence where this naming came from in Claude's recommendation for naming conventions. I initially wanted to name the organization ShadowClone, but it's already taken on GitHub.

```
github.com/<org>/shadow-core
github.com/<org>/shadow-orchestrate
github.com/<org>/shadow-data
github.com/<org>/shadow-rag
```
- Pro: Shorter, brand-aligned
- Con: Less descriptive of function

**Option C: Functional Names**
```
github.com/<org>/agents
github.com/<org>/orchestrate
github.com/<org>/dataprep
github.com/<org>/retrieval
```
- Pro: Clear purpose
- Con: Generic, potential naming conflicts

**My Recommendation**: I lean toward **Option B** or **Option C** for uniqueness and brevity, but I'd value input on what feels right for the team.

---

## Sophia Transformation

### Current Architecture
```
sophia/
├── server/
│   ├── cmd/server/main.go          # Entry point
│   ├── internal/
│   │   ├── api/                    # HTTP handlers
│   │   ├── embedding/ollama.go     # Tightly coupled LLM client
│   │   ├── rag/chain.go            # RAG orchestration
│   │   ├── storage/                # Elasticsearch clients
│   │   ├── extraction/             # Entity extraction
│   │   ├── document/               # Document processing
│   │   ├── blob/                   # MinIO client
│   │   └── cache/                  # Embedding cache
│   └── pkg/models/                 # Shared types
└── client/                         # React frontend
```

### Target Architecture
```
sophia/
├── server/
│   ├── cmd/server/main.go          # Composition root
│   ├── internal/
│   │   ├── api/                    # HTTP handlers (thin)
│   │   ├── config/                 # Application configuration
│   │   └── domain/                 # Application-specific logic
│   └── pkg/                        # (minimal, app-specific types only)
└── client/                         # React frontend

Dependencies:
├── <prefix>-core                   # LLM interaction
├── <prefix>-orchestrate            # Workflows (optional)
├── <prefix>-data                   # Document/entity processing
├── <prefix>-rag                    # Retrieval patterns
├── document-context                # PDF rendering
└── service-kit                     # Web infrastructure
```

### What Remains in Sophia
After extraction, Sophia becomes a **thin application layer**:
- HTTP API routing (application-specific endpoints)
- Configuration management
- Service composition (wiring libraries together)
- Domain-specific prompts and business logic
- React frontend

---

## Extraction Roadmap

### Phase 0: Organization & Repository Setup
**Goal**: Establish GitHub organization and migrate existing libraries

1. **Create GitHub organization**
   - Finalize organization name (`<org>`)
   - Set up organization settings, teams, and permissions

2. **Initialize repositories**
   - Create empty repositories for all planned libraries
   - Establish consistent repository structure (README, LICENSE, .gitignore, go.mod)

3. **Migrate existing libraries**
   - `go-agents` → `<org>/<prefix>-core`
   - `go-agents-orchestration` → `<org>/<prefix>-orchestrate`
   - `document-context` → `<org>/document-context`
   - Update module paths and internal imports
   - Set up CI/CD pipelines

4. **Establish development standards**
   - Repository templates
   - Branch protection rules
   - Release tagging conventions

### Phase 1: Foundation (No Breaking Changes)
**Goal**: Extract libraries with zero external dependencies + establish development standards

1. **Create `<prefix>-data`**
   - Extract from: `internal/extraction/`, `internal/document/chunker.go`, parsers, `internal/cache/`
   - ~2,400 LOC
   - No dependencies on LLM or storage

2. **Create `service-kit`**
   - Extract from: agent-lab's `pkg/`
   - ~2,500 LOC
   - No dependencies on agent libraries

3. **Create `claude-plugin`**
   - Extract patterns from agent-lab's `.claude/` system
   - Optimize skill descriptions for automatic triggering
   - Establish organizational standards plugin
   - Enables consistent AI-assisted development across all projects

### Phase 2: Core Enhancements & Storage
**Goal**: Extend core capabilities and extract storage patterns

4. **Add Audio protocol to `<prefix>-core`**
   - New `Audio` protocol constant
   - `WhisperProvider` implementation (multipart form handling)
   - `Audio()` method on Agent interface
   - ~700 LOC addition
   - Enables Sophia's transcription workflow via unified interface

5. **Create `<prefix>-rag`**
   - Extract from: `internal/storage/`, `internal/rag/`
   - ~2,400 LOC
   - Depends on `<prefix>-core` for embeddings

### Phase 3: Integration
**Goal**: Update applications to consume libraries

6. **Update Sophia**
   - Replace `internal/embedding/ollama.go` with `<prefix>-core` Agent
   - Replace extracted packages with library imports
   - Migrate audio transcription to `Audio()` protocol
   - Thin down to application-specific code

7. **Update agent-lab**
   - Replace `pkg/` with `service-kit` imports
   - Restructure `.claude/` to use `claude-plugin`
   - Validate library usability

### Phase 4: Documentation & Examples
**Goal**: Enable adoption

8. **Create examples**
   - Basic RAG application using libraries
   - Document processing pipeline
   - Multi-provider setup (including Whisper for audio)

9. **Documentation**
   - Library READMEs with usage examples
   - Architecture decision records
   - Migration guides

10. **Publish `claude-plugin` to marketplace**
    - Finalize skill descriptions based on real-world usage
    - Create plugin README and installation guide
    - Enable organization-wide adoption

---

## Benefits of This Approach

### For Sophia
- **Multi-provider support**: Not locked to Ollama
- **Proven patterns**: Leverage tested library code
- **Reduced maintenance**: Shared libraries maintained centrally
- **Cleaner codebase**: Application-focused, not infrastructure-focused

### For the Ecosystem
- **Reusable components**: Other applications can use same libraries
- **Consistent patterns**: Same patterns across projects
- **Community potential**: Libraries can be open-sourced
- **Clear responsibilities**: Each library has focused scope

### For the Team
- **Shared ownership**: Libraries owned by team, not individual
- **Parallel development**: Work on libraries independently
- **Testing**: Smaller, focused test suites
- **Onboarding**: Understand one library at a time

---

## Open Questions for Discussion

1. **Organization name**: What should the GitHub organization be named? (Must be available on GitHub)

2. **Library naming**: Which naming convention (Options A-C) feels right?

3. **service-kit scope**: This library establishes standards and extensibility around stdlib and core dependency infrastructure (HTTP, database drivers, configuration, lifecycle). What additional infrastructure patterns should be included, or should we stay focused on the initial agent-lab public API infrastructure?

4. **Sophia timeline**: How quickly should Sophia adopt these libraries vs. continuing with current architecture?

5. **Open source**: Should these libraries be public or remain organization-internal?

6. **claude-plugin scope**: What organizational standards beyond Go patterns should be included? (e.g., Git workflow, PR conventions, documentation standards)

7. **claude-plugin distribution**: Should this be published to the public Claude Code marketplace or remain internal via `--plugin-dir`? We could potentially setup our own marketplace?

---

## Appendix: Detailed File Mapping

### Files to Extract for `<prefix>-data`

| Source File | LOC | Target Location |
|-------------|-----|-----------------|
| `internal/extraction/patterns.go` | 177 | `pkg/extraction/patterns.go` |
| `internal/extraction/confidence.go` | 128 | `pkg/extraction/confidence.go` |
| `internal/extraction/mgrs.go` | 272 | `pkg/extraction/mgrs.go` |
| `internal/extraction/extractor.go` | 585 | `pkg/extraction/extractor.go` |
| `internal/document/chunker.go` | 159 | `pkg/chunking/chunker.go` |
| `internal/document/csv_parser.go` | 343 | `pkg/parsing/csv.go` |
| `internal/document/json_parser.go` | 540 | `pkg/parsing/json.go` |
| `internal/document/xlsx_parser.go` | 295 | `pkg/parsing/xlsx.go` |
| `internal/cache/embedding_cache.go` | 123 | `pkg/cache/embedding.go` |

### Files to Extract for `<prefix>-rag`

| Source File | LOC | Target Location |
|-------------|-----|-----------------|
| `internal/storage/elasticsearch.go` | 743 | `pkg/vectorstore/elasticsearch.go` |
| `internal/storage/entities_index.go` | 378 | `pkg/entitystore/elasticsearch.go` |
| `internal/storage/sparse_index.go` | 860 | `pkg/sparsestore/elasticsearch.go` |
| `internal/rag/chain.go` | 391 | `pkg/chain/chain.go` |

### Files to Extract for `service-kit`

| Source Directory | Purpose |
|------------------|---------|
| `agent-lab/pkg/lifecycle/` | Startup/shutdown coordination |
| `agent-lab/pkg/runtime/` | Infrastructure composition |
| `agent-lab/pkg/middleware/` | Middleware system |
| `agent-lab/pkg/module/` | Modular routing |
| `agent-lab/pkg/routes/` | Route definitions |
| `agent-lab/pkg/handlers/` | Response helpers |
| `agent-lab/pkg/database/` | Database management |
| `agent-lab/pkg/repository/` | Query utilities |
| `agent-lab/pkg/query/` | SQL builder |
| `agent-lab/pkg/pagination/` | Pagination utilities |
| `agent-lab/pkg/storage/` | Blob abstraction |
| `agent-lab/pkg/openapi/` | OpenAPI generation |

### Patterns to Extract for `claude-plugin`

| Source | Purpose |
|--------|---------|
| `agent-lab/.claude/skills/` | Domain-specific skill patterns (to be optimized for automatic triggering) |
| `agent-lab/.claude/CLAUDE.md` | Project orientation template |

Note: `claude-plugin` contains patterns and documentation, not code extraction. Skills will be rewritten with optimized descriptions following the "REQUIRED for...", "Use when...", "Triggers:" pattern for automatic invocation.
