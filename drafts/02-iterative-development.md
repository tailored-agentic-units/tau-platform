# Iterative Development: A Kaizen Approach to Architecture

**Category:** Architecture

---

This discussion establishes our development philosophy. The core idea: build from the ground up, introduce complexity at precisely the moment it's needed, and let architecture emerge from real requirements rather than speculative design.

## Core Philosophy

It's important to plan the big picture -- understanding where we're headed informs the decisions we make today. But big-picture planning should guide direction, not dictate implementation. The moment we try to lay out a perfect architecture up front, we risk:

- **Analysis paralysis** -- Spending time on decisions that don't matter yet
- **Premature abstraction** -- Building for hypothetical requirements that may never materialize
- **Wasted effort** -- Implementing features that real usage reveals aren't needed
- **Rigidity** -- Locking into patterns before we understand the problem well enough

Instead, we focus on small, digestible requirements that allow capabilities to be built incrementally. Each iteration delivers real value and teaches us what to build next.

## The Target System

We use a range metaphor to prioritize work:

### 20m Targets

Critical, immediate focus items. This is what we're building **right now**.

Characteristics:
- Clearly scoped and well-understood
- Directly unblocks current work
- Can be completed and shipped in the current phase
- Has concrete acceptance criteria

Examples from the TAU ecosystem:
- Add Audio protocol to tau-core (unblocks transcription workflows)
- Migrate go-agents-orchestration to tau-orchestrate (unblocks cross-repo coordination)

### 300m Targets

Directional goals we're working **toward**. They inform architectural decisions but don't dictate implementation details.

Characteristics:
- Broadly scoped, details will emerge through iteration
- Helps us avoid decisions that close off future paths
- Not scheduled for immediate implementation
- May evolve as we learn more

Examples from the TAU ecosystem:
- Full RAG library extraction from Sophia (informed by current library design patterns)
- service-kit extraction from agent-lab (informed by which patterns prove reusable)
- Organization-wide Claude Code plugin (informed by which skills prove valuable)

## How This Works in Practice

### Start with What We Have

tau-core didn't begin as a grand architecture. It started as `go-agents` -- a focused library for LLM protocol execution. Through real usage, it evolved:

1. **Chat protocol** -- The first and simplest protocol
2. **Vision protocol** -- Added when document analysis required it
3. **Tools protocol** -- Added when function calling was needed
4. **Embeddings protocol** -- Added when RAG workflows required vector generation
5. **Audio protocol** -- Next, driven by transcription requirements

Each protocol was added when a real use case demanded it, not because a spec said we'd eventually need it. The architecture accommodated each addition cleanly because the foundational patterns (protocol-centric design, provider abstraction) were sound.

### Only Decide When You Must

Premature decisions carry hidden costs. Every architectural commitment narrows future options. The Kaizen approach defers decisions until the last responsible moment -- the point where not deciding costs more than deciding.

For example:
- **Don't** design a generic plugin system before you have two things to plug in
- **Do** write clean interfaces that could support plugins later
- **Don't** build a message broker integration before you need inter-service communication
- **Do** keep domain logic decoupled from transport so broker integration is feasible later

### Small Iterations, Large Impact

The compounding effect of consistent, focused improvements is significant. Each phase:

1. **Identifies** the highest-value 20m targets
2. **Implements** them with clean patterns and good tests
3. **Reflects** on what we learned and how it changes the 300m targets
4. **Adjusts** the next phase's priorities based on real experience

This creates a feedback loop where each iteration makes us faster and more informed for the next.

## Interaction with Project Management

Phases on the GitHub Projects board serve as iteration boundaries:

- Each phase has a focused set of 20m targets
- Phase completion is a natural reflection point
- 300m targets inform which phases come next, but don't commit to specific implementation
- Backlog items are 300m targets waiting to become 20m targets

## Open Questions

1. **Phase duration**: Should phases be time-boxed (e.g., 2-week sprints) or scope-boxed (done when targets are met)?
2. **Decision records**: Should we track architectural decisions formally (ADRs) or is the discussion + standard lifecycle sufficient?
3. **Retrospectives**: How should we capture lessons learned at phase boundaries?
