# TAU Platform Execution Plan

Strategic execution sequence for current TAU platform development initiatives. Revised 2026-04-17.

## Context

Post-extraction state: five standalone libraries (`protocol`, `format`, `provider`, `agent`, `orchestrate`) are at `v0.1.0` stable. The kernel is still pre-extraction-refactor; it needs to swap local packages for the extracted-library imports.

A `container` library exists at `v0.1.0` with Phase 2 partially landed, but is **paused**. Architectural review revealed that "container" is not a distinct platform layer — it is one implementation of a universal kernel-runtime contract that also needs to cover direct execution on physical hosts, edge devices, and embedded systems. That contract lives at `tau/runtime` (scaffolded in Phase 0). `tau/container` is retained as a reference artifact and will be deleted once its contributions are distilled into `tau/runtime/container` during Phase 6.

The execution sequence below reorients around the revised architecture. Completed work (marketplace refactor, iterative-dev skill, container Phase 1) is archival and is not reflected here.

## Execution Sequence

| Phase | Initiative | Scope | Target |
|-------|-----------|-------|--------|
| 0 | Scaffold `tau/runtime`; update `tau/container` contextual docs | Initialize `~/tau/runtime/` with Go module scaffolding and the initial runtime-contract concept doc at `_project/runtime-contract.md`; create `github.com/tailored-agentic-units/runtime` GitHub repo; update `tau/container/_project/README.md` and `tau/container/.claude/CLAUDE.md` to reflect paused state and position in revised vision. No project board for `tau/runtime` (deferred to Phase 3). No code. | `tau/runtime` (scaffolding only), `tau/container` (docs only) |
| 1 | Kernel post-extraction refactor | Replace local packages with extracted-library imports (`protocol`, `format`, `provider`, `agent`, `orchestrate`). Migrate response model to unified types. Remove ConnectRPC infrastructure. Update `go.mod`. Update README and project docs to reflect post-extraction architecture. | `tau/kernel` |
| 2 | Kernel context + GH cleanup | Close orphaned issues #5–9 on kernel, create equivalents on `tau/orchestrate`. Close #10, create equivalent on `tau/agent`. Archive `_project/library-extraction.md`. Review kernel's GitHub project board, milestones, and objectives (#2, #3, #4) for alignment with the revised direction. | `tau/kernel`, `tau/orchestrate`, `tau/agent` |
| 3 | Kernel Interface + `tau/runtime` contract + minimal `tau/runtime/native` | Evaluate the Phase 2 kernel architecture against the Phase 0 concept doc at `tau/runtime/_project/runtime-contract.md`; refine the concept based on what the cleaned-up kernel actually needs; establish implementation scope. Populate `tau/runtime` root with the `Runtime` interface + registry. Create `tau/runtime/native` sub-module with a minimal implementation sufficient to run an end-to-end kernel loop (pass-through to `tau/agent` acceptable initially). Wire the kernel's public interface through the contract. Initialize the `tau/runtime` GitHub project board now that implementation scope is known. | `tau/kernel`, `tau/runtime`, `tau/runtime/native` (new) |
| 4 | Kernel Interface completion | Objective #2: #26 (multi-session kernel) → #27 (HTTP API with SSE streaming) → #28 (server entry point), completed against the Phase 3 contract. Open issues may need revision to be contract-aware. | `tau/kernel` v0.1.0 |
| 5 | Native implementation maturation + kernel internals reshaping | `tau/runtime/native` matures into the production-grade reference. Kernel internals reshape to handle capabilities through the contract. Objective #4 ("Local Development Mode") rescopes or retires — local dev is a config of `native`, not a separate deployment mode. | `tau/runtime/native`, `tau/kernel` |
| 6 | Bootstrap `tau/runtime/container` + `tau/runtime/container/docker` | With contract + native proven, initialize the container sub-modules fresh. Derive a container spec doc from the finished `Runtime` interface. Review `tau/container`'s `_project/`, `.claude/context/`, and implementation source (`runtime.go`, `exec.go`, `shell.go`, `docker/*`) as design references for decisions worth preserving (label conventions, manifest pathing, PTY sentinel framing, exec plumbing). Implement `tau/runtime/container` + `tau/runtime/container/docker` against the contract. Delete `tau/container` locally and on GitHub once transition completes. | `tau/runtime/container` (new), `tau/runtime/container/docker` (new) |
| 7 | Skills and MCP Integration | Objective #3, proceeds against the matured kernel + runtime stack. | `tau/kernel` |
| 8 | Embedded container mode | Kernel binary baked into container images for production self-contained agentic units. | `tau/runtime/container`, `tau/kernel` |

## Dependency Graph

```
Phase 0 ── 1 ── 2 ── 3 ── 4 ── 5 ── 6 ── 7 ── 8
```

Phases are largely sequential. Phases 3–5 are the load-bearing block: contract design, kernel interface, and the native reference implementation must land before container work resumes in Phase 6.

## Repositories

| Repository | Status | Project Board |
|-----------|--------|---------------|
| [tau/protocol](https://github.com/tailored-agentic-units/protocol) | v0.1.0 stable | — |
| [tau/format](https://github.com/tailored-agentic-units/format) | v0.1.0 stable | — |
| [tau/provider](https://github.com/tailored-agentic-units/provider) | v0.1.0 stable | — |
| [tau/agent](https://github.com/tailored-agentic-units/agent) | v0.1.0 stable | — |
| [tau/orchestrate](https://github.com/tailored-agentic-units/orchestrate) | v0.1.0 stable | — |
| [tau/kernel](https://github.com/tailored-agentic-units/kernel) | Pre-extraction; Phase 1 refactor pending | TAU Kernel |
| [tau/container](https://github.com/tailored-agentic-units/container) | v0.1.0 shipped; Phase 2 paused; contextual docs updated Phase 0; reference artifact through Phase 6; deleted after | TAU Container (will close) |
| [tau/runtime](https://github.com/tailored-agentic-units/runtime) | Phase 0 scaffolded; contract populated in Phase 3 | Deferred to Phase 3 |

## Session Init Artifacts

Each phase has a session initialization artifact that provides full context for starting a development session.

| Phase | Artifact |
|------|----------|
| 0 | `~/tau/runtime/_project/README.md` + `~/tau/runtime/_project/runtime-contract.md` (created here); `~/tau/container/_project/README.md` (updated here); this file (rewritten here) |
| 1 | `~/tau/kernel/_project/post-extraction.md` |
| 2 | `~/tau/kernel/_project/post-extraction.md` (Project Management Updates section) |
| 3 | `~/tau/runtime/_project/runtime-contract.md` (refined here) + kernel `_project/` docs after Phase 2 |
| 4 | Kernel GH issues #26/#27/#28 + `~/tau/kernel/_project/README.md` |
| 5 | `~/tau/runtime/native/_project/README.md` (created in Phase 3) + kernel `_project/` docs |
| 6 | `~/tau/container/_project/` + `~/tau/container/.claude/context/` as reference material; new `~/tau/runtime/container/_project/README.md` |
| 7 | Kernel GH Objective #3 |
| 8 | `~/tau/runtime/container/_project/README.md` (Embedded Mode section) |
