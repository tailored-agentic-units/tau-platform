# Plan: classify-docs Claude Code Concept Document

## Objective

Create `concepts/tau-runtime/claude-classify-docs.md` — a comprehensive concept document that:

1. Makes the case for the TAU ecosystem by demonstrating what's possible with Claude Code's native agent features today
2. Provides a complete specification for implementing the agent-lab classify-docs workflow as a pure Claude Code project (a repository with nothing but README + `.claude/` configuration)

## Research Completed

All sources have been thoroughly explored:

- **tau-runtime concept** (`concepts/tau-runtime/README.md`): 9-library ecosystem, tiered build order, design philosophy
- **agent-lab classify workflow** (`~/code/agent-lab/workflows/classify/`): 4 Go files, 882 lines — init→detect→enhance?→classify→score graph
- **go-agents** (`~/code/go-agents`): LLM interaction library (Agent interface, Vision protocol, provider abstraction)
- **go-agents-orchestration** (`~/code/go-agents-orchestration`): StateGraph, immutable State, Edge predicates, ProcessParallel, CheckpointStore
- **document-context** (`~/code/document-context`): PDF→image via ImageMagick, page extraction via pdfcpu, filesystem caching
- **Claude Code docs**: Subagents, Skills, Memory specifications — features, constraints, best practices

## Document Structure

### 1. Introduction (~300 words)
- TAU ecosystem vision: modular agent runtime libraries establishing standards
- Linux analogy: libraries = kernel/standards, runtime = configurable distribution
- AI transformation: what required a full web service stack can now be agent configuration
- This document: proving the concept by replacing agent-lab's classify-docs with pure Claude Code

### 2. Architecture Overview
- New repository: `classify-docs` — README.md + `.claude/` directory only
- Component mapping table: agent-lab infrastructure → Claude Code features
  - StateGraph → Orchestrating skill with conditional logic
  - State nodes → Individual skills + subagents
  - Immutable State → Working directory files (.work/)
  - System prompts → Skill content
  - document-context → Bash skill calling `magick` directly
  - Parallel vision analysis → Concurrent subagent tasks
  - PostgreSQL/blob storage → Filesystem (eliminated entirely)
- ASCII architecture diagram

### 3. Memory Design (.claude/CLAUDE.md)
- Project identity and purpose
- Classification domain knowledge (hierarchy, caveats, marking locations)
- Workflow overview with expected inputs/outputs
- @imports referencing key skill documentation
- Operational instructions for using the repository

### 4. Skill Specifications (6 skills)

**a. classify-docs** — User-invocable orchestrator (`/classify-docs <path> [--recurse]`)
- Entry point for the entire workflow
- Validates input (PDF file or directory), discovers PDFs if directory
- Creates working directory (.work/<run-id>/)
- Orchestrates: render → detect → enhance? → classify → score
- Handles directory recursion and multi-document processing
- Generates final classification report

**b. render-pages** — PDF→PNG conversion
- Bash-based skill calling ImageMagick `magick` directly
- `magick -density 300 "input.pdf[page]" -background white -flatten output.png`
- Supports enhancement re-rendering with brightness/contrast/saturation
- Outputs to .work/<run-id>/pages/ directory

**c. detect-markings** — Per-page vision analysis (forked context)
- Vision-based analysis of a single page image
- Full DetectionSystemPrompt preserved from agent-lab
- JSON output: markings_found, clarity_score, legibility, filter_suggestion
- context: fork, agent: page-detector

**d. classify-document** — Overall classification determination
- Analyzes aggregate detection results from all pages
- Full ClassificationSystemPrompt preserved from agent-lab
- Applies "highest marking wins" rule
- JSON output: classification, alternative_readings, marking_summary, rationale

**e. score-confidence** — Confidence assessment
- Full ScoringSystemPrompt preserved from agent-lab
- 6 weighted factors, threshold-based recommendation
- JSON output: overall_score, factors[], recommendation

**f. enhance-page** — Conditional page enhancement
- Re-renders page with suggested filter adjustments via ImageMagick
- Re-detects markings on enhanced image
- Merges original and enhanced detections (keeps highest scores)

### 5. Subagent Specifications (2 subagents)

**a. page-detector** — Parallel page detection
- Spawned per-page for concurrent analysis
- Tools: Read, Bash, Glob
- Preloaded skills: detect-markings
- Model: sonnet (vision-capable)
- permissionMode: bypassPermissions (non-interactive batch processing)

**b. classification-analyst** — Classification + scoring
- Sequential analysis of aggregate data
- Preloaded skills: classify-document, score-confidence
- Tools: Read, Write, Glob, Grep
- Model: sonnet

### 6. Workflow Execution Flow
Detailed step-by-step showing the complete execution path with file I/O at each step.

### 7. Working Directory Layout
File tree showing .work/<run-id>/ structure: pages/, detections/, enhanced/, results/

### 8. Repository Directory Layout
Complete file tree of the classify-docs repository.

### 9. Comparison: Agent-Lab vs Claude Code
Side-by-side table comparing infrastructure, lines of code, capabilities, and trade-offs.

### 10. Implications for TAU Ecosystem
- Skills + subagents + memory = agent runtime (proven by this implementation)
- TAU library ↔ Claude Code feature mapping
- The standardization argument: Claude Code is one runtime; TAU provides the standards any runtime can implement
- Alternative implementation potential

### 11. Implementation Milestones
Ordered steps with deliverables.

## Deliverable

Single file: `concepts/tau-runtime/claude-classify-docs.md`

## Verification

After writing the document:
1. Confirm the file renders properly as markdown
2. Verify all system prompts match the originals in agent-lab profile.go
3. Confirm the directory layout is internally consistent (skills reference each other correctly)
4. Verify the architecture mapping covers all agent-lab components
