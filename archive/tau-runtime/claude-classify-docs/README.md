# Classify-Docs: Claude Code Agent Runtime

A document classification workflow implemented entirely as Claude Code configuration — no compiled code, no database, no deployment pipeline. This project analyzes PDF documents for security classification markings, determines the overall classification level, and scores confidence in the assessment.

## Why This Matters

The TAU (Tailored Agentic Units) ecosystem envisions a modular set of Go libraries that compose into a powerful agent runtime. The central claim is that three primitives — **memory**, **skills**, and **subagents** — are sufficient to express non-trivial, multi-stage agentic workflows.

This project proves that claim. It reimplements a Go web service backed by PostgreSQL, blob storage, and three custom libraries (totaling ~11,000 lines of code) as a Claude Code project consisting of nothing but markdown configuration files (~600 lines). The workflow is real, not a toy example: it processes PDF documents through parallel page analysis, conditional image enhancement, classification determination, and confidence scoring.

The implication for the TAU ecosystem is direct: if this level of capability can be expressed through memory, skills, and subagents alone, then a standardized library ecosystem (tau-memory, tau-skills, tau-runtime) that formalizes these patterns unlocks the same capability across any runtime platform — not just Claude Code.

See [spec.md](spec.md) for the full design rationale, component mapping, and comparison with the original Go implementation.

## Prerequisites

- **Claude Code** — The Anthropic CLI tool
- **ImageMagick 7.0+** — The `magick` command must be in PATH

Verify ImageMagick:
```bash
magick --version
```

## Usage

Navigate to this directory and start Claude Code:
```bash
cd classify-docs
claude
```

### Single Document
```
/classify-docs path/to/document.pdf
```

### Directory of Documents
```
/classify-docs path/to/documents/ --recurse
```

### What Happens

The workflow executes five stages:

1. **Render** — Converts PDF pages to PNG images at 300 DPI using ImageMagick
2. **Detect** — Analyzes each page for security markings using vision (parallel subagents)
3. **Enhance** — Re-renders pages with low legibility using adjusted brightness/contrast/saturation, then re-detects markings (conditional — only runs when needed)
4. **Classify** — Determines overall document classification from all detections, applying highest-marking-wins with caveat aggregation
5. **Score** — Evaluates confidence using six weighted factors, producing ACCEPT/REVIEW/REJECT recommendation

Results are written to `.work/<run-id>/` and a classification report is displayed.

## Architecture

```
User invokes: /classify-docs report.pdf

                    ┌────────────────────────────────┐
                    │     classify-docs (skill)      │
                    │     Orchestrator / Entry       │
                    │                                │
                    │  1. Validate input             │
                    │  2. Create .work/<run-id>/     │
                    │  3. Render pages (magick)      │
                    │  4. Spawn page-detectors       │
                    │  5. Evaluate enhancement need  │
                    │  6. Classify + Score           │
                    │  7. Generate report            │
                    └──────┬────────────────────┬────┘
                           │                    │
              ┌────────────▼──────┐   ┌─────────▼──────────────┐
              │  render-pages     │   │ classification-analyst │
              │  (skill, bash)    │   │ (subagent)             │
              │                   │   │                        │
              │  magick CLI       │   │  classify-document     │
              │  PDF → PNG @300   │   │  score-confidence      │
              └───────────────────┘   └────────────────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
    ┌───────▼──────┐ ┌─────▼───────┐ ┌────▼────────┐
    │page-detector │ │page-detector│ │page-detector│
    │ (subagent)   │ │ (subagent)  │ │ (subagent)  │
    │              │ │             │ │             │
    │ detect-      │ │ detect-     │ │ detect-     │
    │ markings     │ │ markings    │ │ markings    │
    └──────────────┘ └─────────────┘ └─────────────┘
         Page 1          Page 2          Page N
```

### Component Mapping to Claude Code Features

| Concept | Claude Code Feature | Files |
|---------|-------------------|-------|
| Domain knowledge | Memory (CLAUDE.md) | `.claude/CLAUDE.md` |
| Workflow stages | Skills (SKILL.md) | `.claude/skills/*/SKILL.md` |
| Parallel execution | Subagents (agent.md) | `.claude/agents/*/agent.md` |
| Orchestration | Orchestrator skill | `.claude/skills/classify-docs/SKILL.md` |
| State management | Working directory | `.work/<run-id>/` |

## File Structure

```
classify-docs/
├── README.md                                    # This file
├── .gitignore                                   # Ignores .work/
├── spec.md                                      # Design specification
├── marked-documents/                            # Test PDFs
│
└── .claude/
    ├── CLAUDE.md                                # Bootstrap memory
    ├── settings.json                            # Permissions
    │
    ├── skills/
    │   ├── classify-docs/SKILL.md               # Orchestrator (entry point)
    │   ├── render-pages/SKILL.md                # PDF → PNG conversion
    │   ├── detect-markings/SKILL.md             # Per-page vision analysis
    │   ├── classify-document/SKILL.md           # Classification determination
    │   ├── score-confidence/SKILL.md            # Confidence scoring
    │   └── enhance-page/SKILL.md               # Page enhancement + re-detection
    │
    └── agents/
        ├── page-detector/agent.md               # Per-page detection (parallel)
        └── classification-analyst/agent.md      # Classification + scoring
```

## Working Directory Layout

Each run creates a self-contained working directory:

```
.work/<YYYYMMDD-HHMMSS>/
├── pages/              # Rendered PNG images
│   ├── page-1.png
│   └── page-N.png
├── detections/         # Per-page detection results (JSON)
│   ├── page-1.json
│   └── page-N.json
├── enhanced/           # Enhanced images + merged detections (if needed)
│   ├── page-3.png
│   └── page-3.json
├── results/            # Final analysis
│   ├── classification.json
│   └── confidence.json
└── report.md           # Human-readable report
```

## Validation

The `marked-documents/` directory contains 27 test PDFs with various classification markings (UNCLASSIFIED through SECRET with caveats like NOFORN, ORCON, and compartment codes). Use these to validate the workflow:

```
/classify-docs marked-documents/marked-documents_1.pdf
```

Or process the entire directory:
```
/classify-docs marked-documents/ --recurse
```
