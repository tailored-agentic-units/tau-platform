# Web Service Architecture: Go + Fiber Standards

**Category:** Architecture

---

This discussion establishes the standard architecture for Go web services in the TAU ecosystem. We're adopting [Fiber](https://gofiber.io/) as our HTTP framework, and mapping proven patterns from the agent-lab reference implementation into a Fiber-based architecture.

The goal is alignment on the structural patterns, not prescriptive code -- implementations will vary by service, but the underlying architecture should be consistent.

## Directory Structure

Two approaches are on the table. Both use the same architectural concepts (systems, handlers, repositories), but differ in how files are organized.

### Option A: Layer-Oriented

Derived from the [`web-service-template.txt`](../archive/web-service-template.txt) proposal. Organizes `internal/` by server layer, where each directory groups all files of a given role:

```
web-service/
├── cmd/server/main.go              # Application entry point
├── internal/
│   ├── config/                     # Configuration management
│   ├── routes/                     # URL mapping to controllers
│   ├── controllers/                # HTTP request/response handling
│   ├── services/                   # Business logic + domain-specific helpers
│   ├── repositories/               # Database operations (CRUD)
│   ├── clients/                    # External API integrations
│   ├── models/                     # Data structures and entities
│   └── utils/                      # Shared utilities
└── pkg/                            # Public libraries (reusable across projects)
```

**Strengths**: Familiar to developers coming from MVC frameworks. Easy to find "all handlers" or "all repositories" at a glance.

**Weaknesses**: Understanding a single domain requires traversing multiple directories (its model is in `models/`, its handler in `controllers/`, its repository in `repositories/`, etc.). As domains grow, layer directories become large and unrelated code lives side-by-side. Cross-domain dependencies are implicit -- nothing in the structure signals which domains depend on which.

### Option B: Domain-Oriented

Derived from the [agent-lab](https://github.com/JaimeStill/agent-lab) architecture. Organizes `internal/` by domain concern, where each domain package encapsulates its own models, system interface, handler, repository, errors, and API specs as a vertical slice:

```
web-service/
├── cmd/server/
│   ├── main.go                     # Entry point: load config, build server, run
│   ├── server.go                   # Server struct: infrastructure + modules
│   ├── modules.go                  # Module assembly and mounting
│   └── http.go                     # HTTP server lifecycle
├── internal/
│   ├── config/                     # Application configuration
│   ├── infrastructure/             # Shared initialization (db, logging, storage, lifecycle)
│   ├── api/                        # API module: wires domain systems into routes
│   ├── agents/                     # Domain: agent management + LLM execution
│   │   ├── agent.go                #   Models and types
│   │   ├── system.go               #   System interface + implementation
│   │   ├── handler.go              #   HTTP handlers
│   │   ├── repository.go           #   Data access
│   │   ├── errors.go               #   Domain-specific errors
│   │   └── openapi.go              #   API documentation specs
│   ├── documents/                  # Domain: document storage
│   │   ├── document.go
│   │   ├── system.go
│   │   ├── handler.go
│   │   ├── repository.go
│   │   └── errors.go
│   └── workflows/                  # Domain: workflow orchestration
│       ├── workflow.go
│       ├── system.go
│       ├── handler.go
│       ├── repository.go
│       ├── runtime.go              #   Cross-domain: references agents, documents
│       └── errors.go
├── pkg/                            # Public libraries (reusable across projects)
└── tests/                          # Black-box tests mirroring public package infrastructure
```

Each domain is a self-contained package that encapsulates its full vertical slice.

**Strengths**:
- **Domain boundaries are explicit** -- Opening `internal/agents/` tells you everything about the agents domain: its types, business logic, HTTP layer, data access, and errors.
- **Dependencies are visible** -- When `workflows/runtime.go` imports `agents.System`, the cross-domain dependency is explicit in the import graph. No hidden coupling.
- **Easier to reason about** -- You don't need to mentally assemble a domain from scattered directories. The package *is* the domain.
- **Scales naturally** -- Adding a new domain means adding a new package. No existing directories get larger.
- **Supports encapsulation** -- Higher-level domains (workflows) can depend on lower-level domain interfaces (agents, documents) without constraining the architecture to a rigid layer hierarchy.

**Weaknesses**: Less familiar to developers accustomed to layer-oriented frameworks. Requires discipline to keep domain packages focused.

The remaining patterns in this discussion (configuration lifecycle, modular sub-systems, domain system pattern, lifecycle coordination) apply to both options, but are demonstrated using the domain-oriented structure.

## Configuration Lifecycle

Configuration is ephemeral -- loaded at startup, transformed into domain types, then discarded.

### Pattern: Layered Configuration

```
base config file --> environment overlay --> environment variables --> finalize
```

1. **Base file** -- Default settings (e.g., `config.toml`)
2. **Environment overlay** -- Optional overrides per environment (e.g., `config.production.toml`)
3. **Environment variables** -- Runtime overrides for deployment-specific values
4. **Finalize** -- Apply defaults, validate, and produce the final typed configuration

### Pattern: Subsystem Configuration

Each subsystem owns its own configuration struct:

- **ServerConfig** -- Host, port, timeouts
- **DatabaseConfig** -- Connection pool settings (max open, max idle, lifetime)
- **StorageConfig** -- Blob storage path, upload limits
- **APIConfig** -- Base path, CORS settings, pagination defaults

Each config struct should have:
- Struct tags for the config file format (TOML, JSON, etc.)
- A `Finalize()` method that applies defaults, loads environment variables, and validates
- A `Merge()` method for layering overlays

### Key Principle

After initialization, no component should hold a reference to a config struct. Configuration is consumed during construction, not stored for runtime access.

## Application Lifecycle

### Cold Start (Construction)

The deterministic initialization sequence:

```
Load config --> Create infrastructure --> Create domain systems --> Wire modules
```

1. **Load config** -- Parse files, merge overlays, finalize
2. **Create infrastructure** -- Shared resources: lifecycle coordinator, logger, database pool, storage
3. **Create domain systems** -- Business logic components, each receiving infrastructure dependencies
4. **Wire modules** -- Assemble HTTP modules from domain handlers, apply middleware

### Hot Start (Runtime)

```
Start lifecycle --> Register hooks --> Serve --> Wait for shutdown signal
```

1. **Start lifecycle** -- Trigger all registered startup hooks (concurrent via WaitGroup)
2. **Register hooks** -- Database ping, storage directory creation, HTTP listener
3. **Serve** -- Accept requests
4. **Wait for signal** -- SIGTERM/SIGINT triggers graceful shutdown

## Modular Sub-Systems

### Infrastructure Layer (Shared)

A single infrastructure struct created once and passed to all domain systems:

```
Infrastructure
├── Lifecycle    # Startup/shutdown coordination
├── Logger       # Structured logging (slog)
├── Database     # Connection pool
└── Storage      # Blob storage
```

Infrastructure components register their own lifecycle hooks:
- **Database**: Pings on startup, closes pool on shutdown
- **Storage**: Creates base directories on startup
- **HTTP server**: Listens on startup, gracefully drains on shutdown

### Domain Systems (Isolated)

Each domain follows this pattern:

```
System interface --> Handler --> Routes --> Repository
```

- **System** -- Interface defining business operations. The public contract.
- **Handler** -- Translates HTTP requests/responses to system calls. Parses input, calls system, writes output.
- **Routes** -- Declares endpoints (method, path, handler function).
- **Repository** -- Data access layer. SQL queries, transactions, persistence.

Domain systems receive infrastructure dependencies via constructor injection:

```go
func New(db *sql.DB, logger *slog.Logger, storage storage.System) System {
    return &system{db: db, logger: logger, storage: storage}
}
```

### Cross-Domain Dependencies

When one domain needs another (e.g., workflows need agents), use interface references:

```go
type Runtime struct {
    agents    agents.System
    documents documents.System
    lifecycle *lifecycle.Coordinator
    logger    *slog.Logger
}
```

The application layer (in `cmd/server/`) wires these dependencies during cold start.

## Lifecycle Coordination

### Coordinator Pattern

A lifecycle coordinator manages startup and shutdown with:

- `OnStartup(fn)` -- Register a concurrent startup hook
- `OnShutdown(fn)` -- Register a shutdown cleanup hook
- `WaitForStartup()` -- Block until all startup hooks complete, then mark ready
- `Shutdown(timeout)` -- Cancel context, wait for shutdown hooks with timeout

### Health Endpoints

Two standard health endpoints:

| Endpoint | Purpose | Returns 200 when |
|----------|---------|-------------------|
| `GET /healthz` | Liveness probe | Server process is running |
| `GET /readyz` | Readiness probe | All startup hooks have completed |

`/readyz` returns 503 until the lifecycle coordinator marks the system as ready.

### Graceful Shutdown

1. Receive SIGTERM or SIGINT
2. Cancel the lifecycle context (signals all subsystems)
3. Stop accepting new HTTP connections
4. Wait for in-flight requests to complete (with timeout)
5. Execute all shutdown hooks concurrently
6. Exit

## Mapping to Fiber

| agent-lab Pattern | Fiber Equivalent |
|-------------------|------------------|
| Custom module router | `fiber.App` with `app.Group()` for prefix isolation |
| Custom middleware stack | Fiber middleware (`app.Use()`) |
| Route registration | `group.Get()`, `group.Post()`, etc. |
| `http.Handler` interface | `fiber.Handler` function signature |
| `http.Server.Shutdown()` | `app.ShutdownWithContext()` |
| Native mux for health | Register health routes directly on the main app |

What Fiber gives us that the stdlib doesn't:
- Built-in middleware ecosystem (CORS, logger, recover, limiter)
- Route grouping with shared middleware
- Request body parsing and validation
- Batteries-included WebSocket support (`fiber.Websocket()`)
- Performance (fasthttp underneath)

What stays the same regardless of framework:
- Configuration lifecycle (framework-agnostic)
- Domain system pattern (business logic doesn't know about HTTP framework)
- Infrastructure layer (database, storage, lifecycle -- framework-agnostic)
- Repository pattern (pure data access)

## Open Questions

1. **Config format**: TOML is the current recommendation -- it supports comments natively, is easier to read and write than JSON, and has a standards-compliant Go library ([go-toml](https://github.com/pelletier/go-toml)) that is well maintained. YAML lacks a standards-compliant Go library, and the available YAML libraries are not maintained by reputable sources. JSON is an option but lacks comments, which limits its usefulness for configuration files.
2. **Error handling convention**: Should we standardize error response format across all services? (e.g., `{"error": "message", "code": "IDENTIFIER"}`)
3. **OpenAPI generation**: agent-lab builds OpenAPI specs inline with route declarations. Fiber has community middleware for this -- should we adopt one, or build our own pattern?
