/dev-workflow concept

This is going to be a deep research and planning task focused on establishing a long-term vision for the tailored agentic units (TAU) ecosystem.

## Current State

Library | Location | Description
--------|----------|------------
*tau-core* | `~/tau/tau-core` | Originally conceived as `go-agents`, a library for platform and model agnostic agents in Go.
*tau-orchestrate* | `~/tau/tau-orchestrate` | Originally conceived as `go-agents-orchestration`, Go-native agent coordination primitives for building multi-agent systems with LangGraph-inspired state management and composable workflow patterns.
*agent-lab* | `~/code/agent-lab` | Built with `go-agents` and `go-agents-orchestration` (the predecessors to `tau-core` and `tau-orchestrate`), was built with the purpose of being able to execute hand-crafted workflows through a web service API with an embedded web client interface.

## Problem State

It feels like there's a missing gap. Currently, the TAU infrastructure is heavily focused on explicit, code-based workflow definitions that seek to solve a very singular use case. How can we leverage this infrastructure, along with the power of Go, to establish a dynamic, modular, and effective agent runtime that enables the same style of features provided by Claude Code?

Specifically:

- The ability to self-organize workflow sequences, request missing, unclear, or contradictory detail clarifications, and dynamically react to injections.
- Plan mode, with markdown-based files defined at a configurable location for organization the plan details (.claude/settings.json "plansDirectory" setting)
- A secure, configurable tool system (.claude/settings.json "permissions" setting)
- Persistent, bootstrapped (always loaded into context on start) memory (.claude/CLAUDE.md)
- Skills for structured capability specifications and commands (.claude/skills)
- MCP support for integration with external data platforms / APIS (.claude/.mcp.json)

## Conceptual Vision

We need to be able to identify which models would be effective candidates for `tau-core` that are available on the platforms we have access to connect to across all classification levels (currently Ollama for self-hosted, Azure AI Foundry for cloud-hosted frontier models), and determine how we can organize these additional features into the TAU ecosystem. Once we have these features established as foundational, modular elements, we can begin to establish an agent runtime that can either be run directly on a local environment (similar to Claude Code), or run within a container sandbox (where the agent can only operate within the container system). A standard API would exist for allowing a user, system, or agent to interface with the agent runtime in a standard way so that whether the runtime is hosted locally or sandboxed within a container, the actual interface to the agent is the same.

## Directive

Using the projects specified in the Current State section above, as well as the sources described in the References section below, perform a comprehensive analysis of:

A. The current state of the TAU ecosystem, along with the infrastructure that was built out in the `agent-lab` web service to see what currently exists, as well as the limitations described in the Problem State section.

B. Engineering research and Claude platform infrastructure that Anthropic has published outlining principles, best practices, lessons learned, and strategies for building out effective agent infrastructure.

C. Determine how we can adapt and extend the TAU ecosystem to provide the features laid out in the Problem State and Conceptual Vision sections.

## References

There are two primary reference sources that I would like for you to use in your analysis.

1. The Anthropic Engineering Blog: https://www.anthropic.com/engineering
  * Building Effective Agents is a solid starting point: https://www.anthropic.com/engineering/building-effective-agents
2. The Claude API Docs: https://platform.claude.com/docs/en/home placing an emphasis on the following sub-sections:
  * Build with Claude + Capabilities: https://platform.claude.com/docs/en/build-with-claude/overview
  * Tool Use: https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
  * Agent Skills: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
  * Agent SDK: https://platform.claude.com/docs/en/agent-sdk/overview
  * Prompt Engineering: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview

Note that there is much more available than directly linked here. This is intended to serve as a starting point and provide appropriate scope to your web search sources. You should avoid referencing sources outside of what has been provided to you in this prompt.
