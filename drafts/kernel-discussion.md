# Kernel Monorepo Migration — Architecture Overview

## Summary

The TAU kernel consolidates 9 separate Go library repositories (tau-core, tau-agent, tau-orchestrate, tau-tools, tau-memory, tau-session, tau-skills, tau-mcp, tau-runtime) into a single monorepo at `github.com/tailored-agentic-units/kernel`.

## Motivation

The original multi-repo strategy introduced friction that was disproportionate to the project's scale:

- **Version cascade**: A change to tau-core required cascading commits and tags through tau-agent and tau-orchestrate — 3+ commits and 3+ tags per atomic change.
- **Diamond dependency risk**: Multiple libraries depending on different versions of tau-core during rapid iteration.
- **Organizational overhead**: 9 sets of CI configs, CLAUDE.md files, settings.json, release workflows, and label taxonomies to maintain.
- **Only 3 of 9 repos had code**: 5 were skeleton-only, and pre-release status meant zero external consumers.

The Go team's own guidance — "packages that change together should live together" — and the kernel architectural metaphor (a single codebase with internal subsystems) both pointed toward consolidation.

## Architecture

Single Go module (`github.com/tailored-agentic-units/kernel`) with subsystems as top-level packages:

| Subsystem | Domain | Status |
|-----------|--------|--------|
| core | Foundational types (Protocol, Message, Response, Config) | Complete |
| agent | LLM client (Agent, Client, Provider, Request) | Complete |
| orchestrate | Coordination (Hub, State, Workflows, Observer) | Complete |
| memory | Persistent memory (bootstrap, working memory, notes) | Skeleton |
| tools | Tool execution (registry, permissions, built-ins) | Skeleton |
| session | Conversation management (history, compaction) | Skeleton |
| skills | Progressive disclosure (SKILL.md discovery, loading) | Skeleton |
| mcp | MCP client (transport, tool discovery) | Skeleton |
| kernel | Agent runtime (agentic loop, plan mode, hooks) | Skeleton |

The kernel exposes a ConnectRPC interface (`tau.kernel.v1.KernelService`) as the extensibility boundary. Extensions (persistence, IAM, UI, observability) connect through this boundary — the kernel has zero awareness of what connects to it.

## Versioning

Single version scheme for the entire module:
- **Phase target**: `v<major>.<minor>.<patch>` (e.g., `v0.1.0`)
- **Dev releases**: `v<target>-dev.<objective>.<issue>` (e.g., `v0.1.0-dev.3.7`)

## Import Path Changes

All imports changed from `github.com/tailored-agentic-units/tau-X/pkg/*` to `github.com/tailored-agentic-units/kernel/*`. For example:
- `tau-core/pkg/protocol` → `kernel/core/protocol`
- `tau-agent/pkg/agent` → `kernel/agent`
- `tau-orchestrate/pkg/hub` → `kernel/orchestrate/hub`

## Full Concept Document

See [concepts/kernel/README.md](https://github.com/tailored-agentic-units/tau-platform/blob/main/concepts/kernel/README.md) for the complete architecture, subsystem descriptions, build order, model compatibility analysis, and retrospective.
