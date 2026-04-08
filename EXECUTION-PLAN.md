# TAU Platform Execution Plan

Strategic execution sequence for current TAU platform development initiatives, established 2026-04-08.

## Context

The TAU ecosystem underwent a major library extraction from the kernel monorepo, producing five standalone libraries (protocol, format, provider, agent, orchestrate) all at v0.1.0. A new container library has been conceptualized. The kernel needs post-extraction refactoring before development can resume. The marketplace needs decomposition from its monolithic plugin structure, and new skills are needed across the ecosystem.

## Execution Sequence

### Group A: Tooling Foundation

Enables clean workflows for all downstream development.

| Step | Initiative | Scope | Target |
|------|-----------|-------|--------|
| ~~A1~~ | ~~Marketplace refactor~~ | ~~Done (2026-04-08). Decomposed into 6 standalone plugins (dev-workflow, github-cli, go-patterns, project-management, tau-overview, kernel) at v0.1.0. Established per-plugin `{plugin}/v{version}` release convention. Removed dev-types, .lsp.json. Steps 4-5 (agent/orchestrate plugins, kernel split) remain deferred.~~ | ~~tau-marketplace~~ |
| A2 | Formalize iterative-dev skill | Adapt ~/code/revolutions/.claude/skills/iterative-dev into a marketplace skill. Maintain dev-workflow role boundaries (developer owns source code; AI owns testing, comments, docs, contextual artifacts). Include implementation guide patterns. | tau-marketplace |
| A3 | Library-dev skills | Create [library]-dev skills for each extracted library: agent, orchestrate, format, provider, protocol. Modeled after kernel-dev. Each lives within its host library repository. | tau/agent, tau/orchestrate, tau/format, tau/provider, tau/protocol |

### Group B: Container Toolkit Mode

Provides sandboxed execution environments for development and prototyping.

| Step | Initiative | Scope | Target |
|------|-----------|-------|--------|
| B1 | Container Phase 1 - Runtime Foundation | OCI-aligned runtime interface, Docker implementation, container lifecycle, one-shot exec, file copy, image capability manifest | tau/container v0.1.0 |
| B2 | Container Phase 2 - Agent Tool Bridge | Tool-wrapped persistent shell, structured file/process tools, dynamic tool generation from manifest, agent integration surface | tau/container v0.2.0 |
| B3 | Container library-dev skill | Create container-dev skill within the container library repository | tau/container |

### Group C: Kernel Development (All Phases)

Resumes kernel development with post-extraction modernization, then completes all planned phases.

| Step | Initiative | Scope | Target |
|------|-----------|-------|--------|
| C1 | Post-extraction refactor | Replace local packages with extracted library imports (protocol, format, provider, agent, orchestrate). Migrate response model to unified types. Remove ConnectRPC infrastructure. Update go.mod. Update README and project docs to reflect post-extraction architecture. | tau/kernel |
| C2 | Update kernel-dev skill | Reflect post-extraction architecture: reduced package scope, library-based dependencies, new extension patterns. Subsequent phases update the skill incrementally. | tau/kernel |
| C3 | Reassign orphaned issues | Close #5-9 on kernel, create equivalents on tau/orchestrate. Close #10, create equivalent on tau/agent. Archive `_project/library-extraction.md`. | tau/kernel, tau/orchestrate, tau/agent |
| C4 | Phase 1 completion | Objective #2: #26 (multi-session kernel) → #27 (HTTP API with SSE streaming) → #28 (server entry point). Sequential dependency chain. | tau/kernel v0.1.0 |
| C5 | Phase 2+ | Objective #3: Skills and MCP Integration | tau/kernel |
| C6 | Phase 3+ | Objective #4: Local Development Mode | tau/kernel |

### Group D: Container Embedded Mode

Requires completed kernel for embedding kernel binary into container images.

| Step | Initiative | Scope | Target |
|------|-----------|-------|--------|
| D1 | Container Phase 3 - Image Management | Image builder from configuration, pre-built image profiles, resource limits, health checks, networking for service integration. Embedded mode: kernel binary baked into container images. | tau/container v0.3.0 |

## Dependency Graph

```
Group A (Tooling)
  A1 ──┐
  A2 ──┤── All downstream work benefits from clean tooling
  A3 ──┘
         │
         v
Group B (Container Toolkit)          Group C (Kernel)
  B1 ── B2 ── B3                      C1 ── C2 ── C3 ── C4 ── C5 ── C6
                                                                       │
         ┌─────────────────────────────────────────────────────────────┘
         v
Group D (Container Embedded)
  D1
```

Groups B and C can execute concurrently after Group A completes — they are independent until Group D, which requires both the container library (B1-B3) and the completed kernel (C4-C6).

## Repositories

| Repository | Status | Project Board |
|-----------|--------|---------------|
| [tau/protocol](https://github.com/tailored-agentic-units/protocol) | v0.1.0 stable | — |
| [tau/format](https://github.com/tailored-agentic-units/format) | v0.1.0 stable | — |
| [tau/provider](https://github.com/tailored-agentic-units/provider) | v0.1.0 stable | — |
| [tau/agent](https://github.com/tailored-agentic-units/agent) | v0.1.0 stable | — |
| [tau/orchestrate](https://github.com/tailored-agentic-units/orchestrate) | v0.1.0 stable | — |
| [tau/kernel](https://github.com/tailored-agentic-units/kernel) | Pre-extraction, Phase 1 in progress | TAU Kernel |
| [tau/container](https://github.com/tailored-agentic-units/container) | Concept complete, no code | TAU Container |
| [tau/tau-marketplace](https://github.com/tailored-agentic-units/tau-marketplace) | Monolithic v0.1.3 | — |

## Session Init Artifacts

Each task has a session initialization artifact that provides full context for starting a development session.

| Step | Artifact |
|------|----------|
| A1 | `~/tau/tau-marketplace/.claude/marketplace-refactor.md` |
| A2 | `~/tau/tau-marketplace/.claude/iterative-dev-skill.md` |
| A3 | `~/tau/tau-marketplace/.claude/library-dev-skills.md` |
| B1-B2 | `~/tau/container/_project/README.md` |
| B3 | References A3 pattern + container architecture |
| C1 | `~/tau/kernel/_project/post-extraction.md` |
| C2 | `~/tau/kernel/.claude/kernel-dev-skill-update.md` |
| C3 | `~/tau/kernel/_project/post-extraction.md` (Project Management Updates section) |
| C4-C6 | Kernel GitHub issues + `~/tau/kernel/_project/README.md` |
| D1 | `~/tau/container/_project/README.md` (Phase 3 + Embedded Mode sections) |
