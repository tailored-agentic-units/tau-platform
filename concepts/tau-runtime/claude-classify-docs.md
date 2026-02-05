# Classify-Docs: A Claude Code Implementation of the TAU Agent Runtime Pattern

## 1. Introduction

The TAU (Tailored Agentic Units) ecosystem envisions a modular set of Go libraries — tau-core, tau-agent, tau-orchestrate, tau-memory, tau-tools, tau-session, tau-skills, tau-mcp, and tau-runtime — that compose into a powerful agent runtime. Each library encapsulates a single capability domain. Together, they enable agents that organize their own workflows, select tools based on environmental feedback, maintain persistent memory, and integrate with external systems. Building this ecosystem is a significant undertaking: nine libraries across four dependency tiers, each requiring careful interface design and integration testing.

Why invest in building it?

Consider the analogy to Linux. The Linux kernel establishes standards — process scheduling, memory management, filesystem interfaces, device drivers — and provides default implementations for each. But the power of the ecosystem lies in the fact that these are *standards*, not mandates. You choose your terminal emulator, your window manager, your file manager, your shell. Each component implements the standard interface while offering its own design philosophy. The result is an ecosystem where innovation happens at every layer, constrained only by the interfaces that hold the system together.

The TAU libraries serve the same purpose for agent runtimes. tau-tools defines what a tool executor looks like; tau-session defines how conversation history is managed; tau-skills defines how capabilities are discovered and loaded. Each library provides a default implementation, but the interfaces are the real product. Someone building an agent runtime on a different platform — or a different language entirely — can implement the same interfaces with their own internals. The standards are portable. The implementations are replaceable.

This document demonstrates why that vision matters by proving what a system-level agent runtime can accomplish *today*. We take the classify-docs workflow — currently a Go web service backed by PostgreSQL, blob storage, and three custom libraries totaling thousands of lines of code — and reimplement it as a Claude Code project consisting of nothing but a README and a `.claude/` directory. No compilation. No database. No deployment pipeline. Just agent configuration: memory files that bootstrap domain knowledge, skills that encode workflow logic, and subagents that provide parallel execution with tool isolation.

The classify-docs workflow analyzes PDF documents to identify security classification markings (UNCLASSIFIED through TOP SECRET, with caveats like NOFORN and ORCON), determines the overall document classification, and scores confidence in that assessment. It is a real workflow solving a real problem — not a toy example. Its successful reimplementation as pure agent configuration makes the argument: if this level of capability can be expressed through memory, skills, and subagents alone, then a standardized library ecosystem that formalizes these patterns unlocks transformative potential.

---

## 2. Architecture Overview

### 2.1 The Repository

The implementation is a standalone repository called `classify-docs`. It contains:

```
classify-docs/
├── README.md
├── .gitignore
└── .claude/
    ├── CLAUDE.md
    ├── settings.json
    ├── skills/
    │   ├── classify-docs/
    │   ├── render-pages/
    │   ├── detect-markings/
    │   ├── classify-document/
    │   ├── score-confidence/
    │   └── enhance-page/
    └── agents/
        ├── page-detector/
        └── classification-analyst/
```

No Go source files. No Dockerfile. No database migrations. No configuration management. The entire workflow is expressed through Claude Code's three native agent features: **memory** (CLAUDE.md), **skills** (SKILL.md files), and **subagents** (agent.md files).

### 2.2 Component Mapping

Every component of the agent-lab implementation maps to a Claude Code feature:

| Agent-Lab Component | Claude Code Equivalent | Notes |
|---|---|---|
| StateGraph execution engine | Orchestrating skill (`classify-docs`) | Skill instructions encode the graph topology |
| State nodes (init, detect, enhance, classify, score) | Individual skills + subagents | Each node becomes a skill; parallelism via subagents |
| Immutable State (key-value store) | Working directory files (`.work/`) | JSON files replace in-memory state |
| Edge predicates (`KeyEquals`, `KeyExists`) | Conditional logic in orchestrator skill | Claude evaluates conditions from detection results |
| `ProcessParallel` worker pool | Concurrent subagent `Task` calls | Multiple page-detector subagents spawned in parallel |
| System prompts (Detection, Classification, Scoring) | Skill content (SKILL.md body) | Prompts preserved verbatim in skill instructions |
| `document-context` library (PDF rendering) | Bash skill calling `magick` directly | ~1,500 lines of Go replaced by shell commands |
| `go-agents` (LLM interaction) | Claude Code's native model access | No HTTP client, provider abstraction, or retry logic needed |
| `go-agents-orchestration` (coordination) | Skill orchestration + subagent delegation | ~2,500 lines of Go replaced by skill sequencing |
| PostgreSQL (checkpoints, runs, stages) | Filesystem state (`.work/` directory) | JSON files provide implicit checkpointing |
| Blob storage (rendered images) | Local filesystem | Images stored in working directory |
| SSE streaming API | Claude Code's native output | Real-time output to terminal |
| Profile system (prompt overrides) | Skill content + memory rules | Override by editing SKILL.md or CLAUDE.md |

### 2.3 Architecture Diagram

```
User invokes: /classify-docs report.pdf

                    ┌──────────────────────────────┐
                    │     classify-docs (skill)    │
                    │     Orchestrator / Entry     │
                    │                              │
                    │  1. Validate input           │
                    │  2. Create .work/<run-id>/   │
                    │  3. Invoke render-pages      │
                    │  4. Spawn page-detectors     │
                    │  5. Evaluate enhancement need│
                    │  6. Invoke classify-document │
                    │  7. Invoke score-confidence  │
                    │  8. Generate final report    │
                    └──────┬───────────────────┬───┘
                           │                   │
              ┌────────────▼──────┐   ┌───────▼───────────────┐
              │  render-pages     │   │ classification-analyst│
              │  (skill, bash)    │   │ (subagent)            │
              │                   │   │                       │
              │  magick CLI       │   │  classify-document    │
              │  PDF → PNG @300   │   │  score-confidence     │
              │  Enhancement      │   │                       │
              └───────────────────┘   └───────────────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
    ┌───────▼──────┐ ┌────▼────────┐ ┌────▼────────┐
    │page-detector │ │page-detector│ │page-detector│
    │ (subagent)   │ │ (subagent)  │ │ (subagent)  │
    │              │ │             │ │             │
    │ detect-      │ │ detect-     │ │ detect-     │
    │ markings     │ │ markings    │ │ markings    │
    │ (skill)      │ │ (skill)     │ │ (skill)     │
    └──────────────┘ └─────────────┘ └─────────────┘
         Page 1          Page 2          Page N
```

---

## 3. Memory Design

The `.claude/CLAUDE.md` file bootstraps the agent with project identity, domain knowledge, and operational instructions. It is loaded into context at the start of every session.

### 3.1 Content Specification

```markdown
# Classify-Docs

Document classification analysis powered by Claude Code skills and subagents.

## Purpose

This repository provides an automated document classification workflow that:
- Analyzes PDF documents to identify security classification markings
- Determines the overall classification level of each document
- Scores confidence in the classification assessment
- Supports single files and recursive directory processing

## Classification Domain

### Hierarchy (lowest to highest)

UNCLASSIFIED < CONFIDENTIAL < SECRET < TOP SECRET

Classification is determined by the **highest marking level** found anywhere
in the document. A single SECRET marking on one page means the entire document
is classified SECRET, regardless of markings on other pages.

### Caveats

Caveats are appended to classification levels with double-slash separators:
- **NOFORN** — Not Releasable to Foreign Nationals
- **ORCON** — Originator Controlled
- **REL TO** — Releasable To (specific countries)
- **PROPIN** — Proprietary Information
- **FISA** — Foreign Intelligence Surveillance Act

Example: `SECRET//NOFORN//ORCON`

### Marking Locations

Standard locations for security markings on document pages:
- **Header** — Top of page, centered or left-aligned
- **Footer** — Bottom of page, centered or left-aligned
- **Margin** — Left or right margins, sometimes rotated
- **Body** — Inline with document content (paragraph markings)

### Critical Rules

1. ALL detected markings contribute to classification regardless of legibility
   or fading. A faded marking is still a valid marking.
2. Legibility and fading affect *confidence scoring*, NOT classification itself.
3. The classification is always the HIGHEST level found.
4. Caveats from any page are included in the overall classification.

## Workflow

Invoke: `/classify-docs <path> [--recurse]`

The workflow executes five stages:
1. **Render** — Convert PDF pages to PNG images at 300 DPI
2. **Detect** — Analyze each page for security markings (parallel)
3. **Enhance** — Re-render and re-analyze pages with low legibility (conditional)
4. **Classify** — Determine overall classification from all detections
5. **Score** — Evaluate confidence with weighted factors

## System Requirements

- ImageMagick 7.0+ (`magick` command must be in PATH)
- Sufficient disk space for rendered page images

## Working Directory

Each workflow run creates `.work/<run-id>/` with:
- `pages/` — Rendered PNG images
- `detections/` — Per-page detection JSON results
- `enhanced/` — Enhanced images and merged detections
- `results/` — Classification and confidence output
- `report.md` — Final classification report
```

### 3.2 Design Rationale

The memory file serves three purposes:

1. **Domain injection**: The classification hierarchy, caveat definitions, and critical rules ensure Claude applies the correct logic regardless of which skill or subagent is active. This replaces the hard-coded system prompts and profile system in agent-lab.

2. **Workflow overview**: By documenting the five-stage workflow and its invocation, the memory enables Claude to answer questions about the project, explain what the workflow does, and reason about execution order without loading any skill content.

3. **System requirements**: Documenting the ImageMagick dependency ensures Claude can diagnose failures when the tool is missing, rather than producing opaque error messages.

---

## 4. Skill Specifications

### 4.1 classify-docs (Orchestrator)

The entry point for the entire workflow. User-invocable via `/classify-docs`.

**File**: `.claude/skills/classify-docs/SKILL.md`

```yaml
---
name: classify-docs
description: >
  Analyze PDF documents for security classification markings.
  Determines overall classification level and confidence score.
  Use when the user wants to classify a document or directory of documents.
argument-hint: <pdf-path-or-directory> [--recurse]
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Task
---
```

**Instructions outline**:

1. **Input validation**
   - If `$ARGUMENTS` is a file path: verify it exists and is a PDF
   - If `$ARGUMENTS` is a directory: discover all PDFs (with `--recurse` for subdirectories)
   - Create working directory: `.work/<timestamp>/`

2. **For each PDF document**:
   - Get page count: `magick identify -format "%n\n" "<pdf>" | head -1`
   - Invoke the `render-pages` skill to convert all pages to PNG
   - Spawn one `page-detector` subagent per page using the Task tool (parallel)
     - Each subagent receives: the page image path and page number
     - Each subagent writes detection results to `.work/<run-id>/detections/page-<N>.json`
   - Wait for all page detectors to complete
   - Read all detection JSON files and evaluate enhancement need:
     - If ANY page has a marking with `legibility < 0.4` AND a `filter_suggestion`: enhancement needed
   - If enhancement needed: invoke `enhance-page` skill for each flagged page
   - Compile all detection results (original + enhanced) into a single summary
   - Invoke `classify-document` skill with the compiled detections
   - Invoke `score-confidence` skill with classification results + detection metadata
   - Write results to `.work/<run-id>/results/`

3. **Report generation**
   - Generate `.work/<run-id>/report.md` with:
     - Document name and path
     - Overall classification
     - Confidence score and recommendation
     - Per-page detection summary
     - Alternative readings (if any)
     - Rationale

4. **Output**: Display the final report to the user

### 4.2 render-pages (PDF to PNG Conversion)

Converts PDF pages to PNG images using ImageMagick. Replaces the entire `document-context` library.

**File**: `.claude/skills/render-pages/SKILL.md`

```yaml
---
name: render-pages
description: >
  Convert PDF document pages to PNG images using ImageMagick.
  Supports standard rendering and enhancement re-rendering with
  brightness, contrast, and saturation adjustments.
user-invocable: false
allowed-tools: Bash, Read, Write
---
```

**Instructions outline**:

Standard rendering (all pages):
```bash
# Get page count
PAGE_COUNT=$(magick identify -format "%n\n" "$PDF_PATH" | head -1)

# Render each page (0-indexed for ImageMagick)
for i in $(seq 0 $((PAGE_COUNT - 1))); do
  PAGE_NUM=$((i + 1))
  magick -density 300 "${PDF_PATH}[$i]" \
    -background white -flatten \
    "${OUTPUT_DIR}/page-${PAGE_NUM}.png"
done
```

Enhancement re-rendering (specific page with filters):
```bash
magick -density 300 "${PDF_PATH}[$PAGE_INDEX]" \
  -background white -flatten \
  -modulate ${BRIGHTNESS},${SATURATION} \
  -brightness-contrast ${CONTRAST} \
  "${OUTPUT_DIR}/enhanced-page-${PAGE_NUM}.png"
```

**Key details**:
- DPI fixed at 300 (matches agent-lab default)
- PNG format for lossless quality (classification markings need clarity)
- ImageMagick page indexing is 0-based; output filenames are 1-based
- Filter parameters: brightness (0-200, 100=neutral), contrast (-100 to 100, 0=neutral), saturation (0-200, 100=neutral)
- Only include `-modulate` and `-brightness-contrast` flags when enhancement values differ from neutral

### 4.3 detect-markings (Per-Page Vision Analysis)

Analyzes a single page image for security classification markings using vision capabilities.

**File**: `.claude/skills/detect-markings/SKILL.md`

```yaml
---
name: detect-markings
description: >
  Analyze a document page image for security classification markings.
  Returns structured detection results with legibility and clarity scores.
  Used by the page-detector subagent for per-page analysis.
user-invocable: false
allowed-tools: Read, Write
---
```

**Instructions** (preserving the DetectionSystemPrompt from agent-lab):

```
You are a document security marking detection specialist. Analyze the
provided document page image and identify all security classification markings.

Read the page image at the provided path. Analyze it for security markings
and write a JSON result file.

OUTPUT FORMAT: Write a JSON file with this exact schema:
{
  "page_number": <integer>,
  "markings_found": [
    {
      "text": "<exact marking text>",
      "location": "<header|footer|margin|body>",
      "legibility": <0.0-1.0>,
      "faded": <boolean>
    }
  ],
  "clarity_score": <0.0-1.0>,
  "filter_suggestion": {
    "brightness": <optional integer 0-200>,
    "contrast": <optional integer -100 to 100>,
    "saturation": <optional integer 0-200>
  } or null
}

INSTRUCTIONS:
- Identify ALL security markings (e.g., UNCLASSIFIED, CONFIDENTIAL, SECRET,
  TOP SECRET, caveats like NOFORN, ORCON, or any other code names)
- Note the location of each marking (header, footer, margin, or body)
- LEGIBILITY measures readability: 1.0 = perfectly readable, 0.0 = illegible
- FADED indicates visual appearance: true if marking appears washed out or pale
- IMPORTANT: A faded marking can still have high legibility if the text is readable
- clarity_score reflects overall page quality for marking detection
- Only suggest filter_suggestion if legibility < 0.4 AND you believe enhancement
  would improve readability
- Write the JSON result to the specified output path
```

### 4.4 classify-document (Classification Determination)

Analyzes aggregate detection results to determine overall document classification.

**File**: `.claude/skills/classify-document/SKILL.md`

```yaml
---
name: classify-document
description: >
  Determine overall document classification from aggregated page detections.
  Applies the highest-marking-wins rule with caveat aggregation.
user-invocable: false
allowed-tools: Read, Write
---
```

**Instructions** (preserving the ClassificationSystemPrompt from agent-lab):

```
You are a document classification specialist. Analyze security marking
detections across all pages to determine the overall document classification.

Read all detection JSON files from the provided directory. Analyze the
aggregate results and write a classification JSON file.

OUTPUT FORMAT: Write a JSON file with this exact schema:
{
  "classification": "<overall classification level>",
  "alternative_readings": [
    {
      "classification": "<alternative classification>",
      "probability": <0.0-1.0>,
      "reason": "<brief phrase, max 15 words>"
    }
  ],
  "marking_summary": ["<list of unique markings found>"],
  "rationale": "<1-2 sentences, max 40 words>"
}

INSTRUCTIONS:
- Analyze all marking detections provided
- Determine the HIGHEST classification level present
- IMPORTANT: ALL detected markings contribute to classification regardless of
  legibility or fading
- A faded or low-legibility marking is still a valid marking — include it in
  your classification decision
- Include detected caveats (NOFORN, ORCON, REL TO, etc.) in the primary
  classification (e.g., SECRET//NOFORN)
- Legibility and fading affect confidence scoring, NOT the classification itself
- Only list alternative readings if there is genuine ambiguity about what marking
  text says
- marking_summary should list unique marking texts (deduplicated)
- Keep rationale brief: 1-2 sentences explaining the key deciding factor
```

### 4.5 score-confidence (Confidence Assessment)

Evaluates classification confidence using weighted factors.

**File**: `.claude/skills/score-confidence/SKILL.md`

```yaml
---
name: score-confidence
description: >
  Evaluate confidence in document classification results using
  weighted scoring factors. Produces ACCEPT, REVIEW, or REJECT recommendation.
user-invocable: false
allowed-tools: Read, Write
---
```

**Instructions** (preserving the ScoringSystemPrompt from agent-lab):

```
You are a confidence scoring specialist. Evaluate the quality and reliability
of document classification results.

Read the classification result and all detection files. Compute the metadata
(total pages, average clarity, average legibility, spatial coverage, whether
enhancement was applied) and evaluate the following factors.

OUTPUT FORMAT: Write a JSON file with this exact schema:
{
  "overall_score": <0.0-1.0>,
  "factors": [
    {
      "name": "<factor name>",
      "score": <0.0-1.0>,
      "weight": <weight>,
      "description": "<brief phrase, max 10 words>"
    }
  ],
  "recommendation": "<ACCEPT|REVIEW|REJECT>"
}

FACTORS TO EVALUATE:
- marking_clarity (weight: 0.30): Average clarity across pages
- marking_consistency (weight: 0.25): Marking agreement across pages
- spatial_coverage (weight: 0.15): Markings in expected locations (header/footer)
- enhancement_impact (weight: 0.10): Value added by enhancement (if applied)
- alternative_count (weight: 0.10): Fewer alternatives = higher confidence
- detection_legibility (weight: 0.10): Average marking legibility

THRESHOLDS:
- >= 0.90: ACCEPT — Classification is reliable
- 0.70-0.89: REVIEW — Human verification recommended
- < 0.70: REJECT — Insufficient confidence

Keep factor descriptions brief (max 10 words each).
```

### 4.6 enhance-page (Conditional Page Enhancement)

Re-renders a page with filter adjustments and re-detects markings.

**File**: `.claude/skills/enhance-page/SKILL.md`

```yaml
---
name: enhance-page
description: >
  Re-render a document page with brightness, contrast, and saturation
  adjustments, then re-detect markings on the enhanced image. Merges
  original and enhanced detections, keeping the highest-quality results.
user-invocable: false
allowed-tools: Bash, Read, Write
---
```

**Instructions outline**:

1. Read the original detection JSON for the page
2. Extract the `filter_suggestion` values (brightness, contrast, saturation)
3. Re-render the page using `magick` with the suggested filters (invoke render-pages pattern)
4. Read the enhanced image and re-detect markings (apply detect-markings logic)
5. Merge original and enhanced detections:
   - Keep the **higher** clarity_score
   - For each marking (matched by text + location):
     - If original was faded or had legibility < 0.4, prefer the enhanced detection
     - Otherwise keep the original
   - Deduplicate markings by text + location
6. Write merged detection to `.work/<run-id>/enhanced/page-<N>.json`

---

## 5. Subagent Specifications

### 5.1 page-detector (Parallel Page Analysis)

Spawned once per page for concurrent vision analysis. Multiple instances run in parallel.

**File**: `.claude/agents/page-detector/agent.md`

```yaml
---
name: page-detector
description: >
  Analyze a single document page image for security classification markings.
  Spawned per-page for parallel detection across multi-page documents.
tools: Read, Write, Bash, Glob
model: sonnet
permissionMode: bypassPermissions
skills:
  - detect-markings
---
```

**System prompt**:

```
You are a document page analysis agent. Your task is to analyze a single
page image for security classification markings.

You will receive:
1. A page image path (PNG file)
2. A page number
3. An output path for detection results

Use the detect-markings skill to analyze the page image. Read the image at
the provided path, identify all security markings, and write the structured
JSON detection result to the specified output path.

Focus exclusively on the single page you are assigned. Do not attempt to
analyze other pages or make document-level classification decisions.
```

**Design rationale**:
- `model: sonnet` — Vision-capable with sufficient reasoning for marking detection
- `permissionMode: bypassPermissions` — Non-interactive batch processing; these agents run as part of an automated workflow and should not prompt for permission
- `tools: Read, Write, Bash, Glob` — Read for image access, Write for JSON output, Bash for any image preprocessing, Glob for file discovery
- `skills: [detect-markings]` — Preloads the detection prompt so the subagent has full instructions

### 5.2 classification-analyst (Classification and Scoring)

Performs the analytical stages: classification determination and confidence scoring.

**File**: `.claude/agents/classification-analyst/agent.md`

```yaml
---
name: classification-analyst
description: >
  Analyze aggregated page detection results to determine overall document
  classification and evaluate confidence. Performs classification and scoring
  sequentially.
tools: Read, Write, Glob, Grep
model: sonnet
permissionMode: bypassPermissions
skills:
  - classify-document
  - score-confidence
---
```

**System prompt**:

```
You are a document classification analyst. Your task is to determine the
overall security classification of a document and evaluate confidence in
that assessment.

You will receive:
1. A directory path containing per-page detection JSON files
2. An output directory for classification and confidence results
3. Metadata: total pages, whether enhancement was applied

Execute two analyses sequentially:

1. CLASSIFY: Read all detection files. Use the classify-document skill to
   determine the overall classification level. Write the result to
   classification.json in the output directory.

2. SCORE: Read the classification result and all detection files. Use the
   score-confidence skill to evaluate confidence. Write the result to
   confidence.json in the output directory.

Both analyses must complete. Do not skip scoring even if classification
seems highly confident.
```

**Design rationale**:
- `model: sonnet` — Sufficient reasoning for classification logic; this operates on JSON data, not images
- `tools: Read, Write, Glob, Grep` — File I/O for reading detections and writing results; no Bash needed (pure analytical work)
- `skills: [classify-document, score-confidence]` — Both analytical skills preloaded for sequential execution
- Classification and scoring run sequentially within a single subagent rather than as separate subagents, because scoring depends on classification output

---

## 6. Workflow Execution Flow

### 6.1 Single Document Flow

```
User: /classify-docs /path/to/report.pdf

Step 1: Validate & Initialize
  ├─ Verify report.pdf exists and is a PDF
  ├─ Create .work/20260204-143022/
  ├─ Create subdirectories: pages/, detections/, enhanced/, results/
  └─ Get page count via: magick identify -format "%n\n" report.pdf

Step 2: Render Pages
  ├─ For each page (0 to N-1):
  │   magick -density 300 "report.pdf[$i]" -background white -flatten \
  │     .work/20260204-143022/pages/page-$((i+1)).png
  └─ Verify all page images exist

Step 3: Detect Markings (Parallel)
  ├─ Spawn N page-detector subagents via Task tool (single message, parallel)
  │   Each receives:
  │     - Image path: .work/20260204-143022/pages/page-<N>.png
  │     - Page number: <N>
  │     - Output path: .work/20260204-143022/detections/page-<N>.json
  ├─ All subagents execute concurrently
  └─ Collect results: read all .work/20260204-143022/detections/*.json

Step 4: Evaluate Enhancement Need
  ├─ For each detection, check:
  │   - Any marking with legibility < 0.4?
  │   - filter_suggestion present (not null)?
  ├─ If yes for ANY page → enhancement needed
  └─ If no → skip to Step 6

Step 5: Enhance Pages (Conditional)
  ├─ For each page needing enhancement:
  │   ├─ Extract filter_suggestion from detection JSON
  │   ├─ Re-render: magick -density 300 "report.pdf[$i]" \
  │   │     -background white -flatten \
  │   │     -modulate $BRIGHTNESS,$SATURATION \
  │   │     -brightness-contrast $CONTRAST \
  │   │     .work/20260204-143022/enhanced/page-<N>.png
  │   ├─ Re-detect markings on enhanced image
  │   └─ Merge original + enhanced detections
  └─ Write merged results to .work/20260204-143022/enhanced/page-<N>.json

Step 6: Classify Document
  ├─ Spawn classification-analyst subagent with:
  │   - Detection directory: .work/20260204-143022/detections/ (+ enhanced/)
  │   - Output directory: .work/20260204-143022/results/
  │   - Metadata: page count, enhancement applied flag
  ├─ Analyst reads all detections, writes classification.json
  └─ Analyst reads classification + detections, writes confidence.json

Step 7: Generate Report
  ├─ Read classification.json and confidence.json
  ├─ Compile report.md with:
  │   - Document: report.pdf
  │   - Classification: SECRET//NOFORN
  │   - Confidence: 0.87 (REVIEW)
  │   - Pages analyzed: 12
  │   - Markings found: SECRET (10 pages), NOFORN (8 pages)
  │   - Alternative readings: CONFIDENTIAL//NOFORN (0.15 probability)
  │   - Rationale: "Consistent SECRET markings across all pages..."
  └─ Display report to user
```

### 6.2 Directory Processing Flow

When invoked with a directory path and `--recurse`:

```
User: /classify-docs /path/to/documents --recurse

Step 1: Discover PDFs
  ├─ Glob: /path/to/documents/**/*.pdf (recursive)
  └─ Found: 5 PDF files

Step 2: Process Each Document
  ├─ For each PDF: execute Steps 1-7 from single document flow
  ├─ Each document gets its own .work/<run-id>/<document-name>/
  └─ Note: documents processed sequentially (each may spawn parallel subagents)

Step 3: Aggregate Report
  ├─ Compile summary across all documents
  └─ Display aggregate report to user
```

---

## 7. Working Directory Layout

Each workflow run creates a self-contained working directory:

```
.work/
└── 20260204-143022/                    # Run ID (timestamp)
    ├── pages/                           # Rendered page images
    │   ├── page-1.png
    │   ├── page-2.png
    │   └── page-N.png
    ├── detections/                      # Per-page detection results
    │   ├── page-1.json
    │   ├── page-2.json
    │   └── page-N.json
    ├── enhanced/                        # Enhanced images + merged detections
    │   ├── page-3.png                   # Only pages that needed enhancement
    │   ├── page-3.json                  # Merged detection for enhanced page
    │   ├── page-7.png
    │   └── page-7.json
    ├── results/                         # Final analysis output
    │   ├── classification.json
    │   └── confidence.json
    └── report.md                        # Human-readable final report
```

This directory structure serves as both intermediate state (replacing agent-lab's immutable `State` object) and implicit checkpoints (files persist if the workflow is interrupted). A resumed session can inspect `.work/` to determine which stages completed and continue from the last complete stage.

---

## 8. Repository Directory Layout

The complete file tree for the classify-docs repository:

```
classify-docs/
├── README.md                                    # Project overview and usage
├── .gitignore                                   # Ignore .work/ directory
│
└── .claude/
    ├── CLAUDE.md                                # Bootstrap memory (Section 3)
    ├── settings.json                            # Claude Code settings
    │
    ├── skills/
    │   ├── classify-docs/
    │   │   └── SKILL.md                         # Orchestrator (Section 4.1)
    │   │
    │   ├── render-pages/
    │   │   └── SKILL.md                         # PDF→PNG conversion (Section 4.2)
    │   │
    │   ├── detect-markings/
    │   │   └── SKILL.md                         # Vision analysis (Section 4.3)
    │   │
    │   ├── classify-document/
    │   │   └── SKILL.md                         # Classification logic (Section 4.4)
    │   │
    │   ├── score-confidence/
    │   │   └── SKILL.md                         # Confidence scoring (Section 4.5)
    │   │
    │   └── enhance-page/
    │       └── SKILL.md                         # Page enhancement (Section 4.6)
    │
    └── agents/
        ├── page-detector/
        │   └── agent.md                         # Per-page vision agent (Section 5.1)
        │
        └── classification-analyst/
            └── agent.md                         # Classification + scoring (Section 5.2)
```

**Total files**: 11 (1 README, 1 .gitignore, 1 CLAUDE.md, 1 settings.json, 6 SKILL.md, 2 agent.md)

---

## 9. Comparison: Agent-Lab vs Claude Code

### 9.1 Infrastructure

| Aspect | Agent-Lab | Claude Code |
|---|---|---|
| **Language** | Go (compiled binary) | Markdown configuration |
| **Database** | PostgreSQL (checkpoints, runs, stages, decisions, profiles, agents, documents) | None (filesystem) |
| **Storage** | S3-compatible blob storage | Local filesystem |
| **Web server** | HTTP API with SSE streaming | None (CLI interaction) |
| **Dependencies** | go-agents, go-agents-orchestration, document-context, pdfcpu, ImageMagick, PostgreSQL driver, S3 driver | ImageMagick only |
| **Deployment** | Build binary, provision database, configure storage, deploy service | Clone repository, run `claude` |
| **Configuration** | TOML config files, database profiles, environment variables | CLAUDE.md + SKILL.md files |

### 9.2 Code Volume

| Component | Agent-Lab | Claude Code |
|---|---|---|
| Workflow logic | ~880 lines Go (classify/) | ~400 lines markdown (6 SKILL.md) |
| LLM interaction | ~3,000 lines Go (go-agents) | 0 (native to Claude Code) |
| Orchestration | ~2,500 lines Go (go-agents-orchestration) | 0 (native to Claude Code) |
| PDF rendering | ~1,500 lines Go (document-context) | ~30 lines bash in render-pages skill |
| Web service | ~2,000 lines Go (api/, internal/) | 0 (not applicable) |
| Database layer | ~1,500 lines Go (migrations, stores, models) | 0 (not applicable) |
| Agent definitions | — | ~60 lines markdown (2 agent.md) |
| Memory | — | ~80 lines markdown (CLAUDE.md) |
| **Total** | **~11,000+ lines** | **~570 lines** |

### 9.3 Capabilities

| Capability | Agent-Lab | Claude Code |
|---|---|---|
| PDF page rendering | 300 DPI PNG via document-context | 300 DPI PNG via magick CLI |
| Vision analysis | Via go-agents Vision protocol | Native Claude vision |
| Parallel detection | ProcessParallel worker pool | Concurrent subagent Tasks |
| Conditional enhancement | StateGraph edge predicates | Orchestrator skill conditional logic |
| Classification logic | Dedicated system prompt | Skill with same prompt preserved |
| Confidence scoring | 6 weighted factors | Same 6 weighted factors |
| Profile overrides | Database-stored profiles | Edit SKILL.md files |
| Checkpoint/resume | PostgreSQL CheckpointStore | Filesystem working directory |
| Streaming events | SSE (Server-Sent Events) | Claude Code terminal output |
| API access | REST HTTP endpoints | Claude Code CLI |
| Multi-model support | Ollama, Azure providers | Claude models (native) |
| Observability | PostgreSQL stages + decisions tables | Claude Code session history |

### 9.4 Trade-offs

**Gained**:
- Zero infrastructure overhead (no database, no deployment, no compilation)
- Immediate iteration on prompts (edit SKILL.md, re-run)
- Portable across any machine with Claude Code and ImageMagick
- Natural language interaction during workflow (ask questions, request clarification)
- Built-in reasoning and error recovery from Claude's capabilities

**Different**:
- No persistent run history database (working directories serve as run archives)
- No REST API for programmatic integration (CLI-only)
- No SSE event streaming to external clients
- Model selection limited to Claude models (no Ollama/Azure provider switching)
- Parallel execution governed by Claude Code's subagent concurrency, not configurable worker pools

---

## 10. Implications for the TAU Ecosystem

### 10.1 The Pattern Is Proven

This implementation demonstrates that three primitives — **memory**, **skills**, and **subagents** — compose into a functional agent runtime capable of executing a non-trivial, multi-stage workflow with parallel processing, conditional routing, and structured output. The classify-docs workflow is not a contrived example; it is a real workflow that was previously implemented as a full web service.

The TAU ecosystem formalizes exactly these primitives:

| Claude Code Feature | TAU Library | Role |
|---|---|---|
| CLAUDE.md (memory) | **tau-memory** | Persistent context loaded at startup |
| SKILL.md (skills) | **tau-skills** | Progressive disclosure capability specifications |
| agent.md (subagents) | **tau-runtime** | Task delegation with isolation and tool control |
| Skill orchestration | **tau-orchestrate** | Workflow coordination and state management |
| Bash tool (magick) | **tau-tools** | Tool execution interface with permissions |
| Conversation context | **tau-session** | Context window management and compaction |
| Model interaction | **tau-agent** | LLM communication abstraction |
| Shared types | **tau-core** | Foundational type vocabulary |

### 10.2 The Standardization Argument

Claude Code is one agent runtime — an excellent one, but one nonetheless. It makes specific choices: Claude models only, CLI interaction, specific subagent concurrency behavior, particular skill loading semantics. These choices are appropriate for Claude Code's use case but are not universal.

The TAU ecosystem extracts the *patterns* from implementations like Claude Code and establishes them as standards:

- **tau-tools** doesn't mandate how tools execute; it defines what a tool executor interface looks like. An implementation could use Claude Code's Bash tool, a Docker container, a WASM sandbox, or a remote API.
- **tau-skills** doesn't mandate SKILL.md format; it defines how skills are discovered, how progressive disclosure works (metadata → instructions → resources), and how skills match to prompts. An implementation could use YAML, TOML, or a database.
- **tau-memory** doesn't mandate CLAUDE.md; it defines bootstrap loading, working memory, and structured notes as concepts. An implementation could use files, a key-value store, or a vector database.

This is the Linux analogy in practice. The TAU libraries are the kernel interfaces. Claude Code, agent-lab, and future platforms are distributions that implement those interfaces with their own design choices.

### 10.3 Alternative Implementation Potential

Because the TAU libraries define interfaces rather than mandating implementations, the same classify-docs workflow could be executed on different runtimes:

- **Claude Code runtime** (this document): Skills, subagents, memory as markdown files
- **tau-runtime (Go)**: The full TAU library stack with any LLM provider via tau-agent
- **Container runtime**: tau-runtime deployed as a containerized web service with REST API
- **Custom runtime**: Any platform that implements the tau-tools, tau-skills, and tau-memory interfaces

The workflow specification — the prompts, the graph topology, the scoring factors — remains constant. Only the execution machinery changes. This is the power of standardized interfaces.

### 10.4 What This Means for Development Priority

This implementation validates the core design thesis of the TAU ecosystem: the agent runtime pattern is real, it works, and it produces tangible value. The remaining question is not *whether* to build the ecosystem, but *how fast* the standardized libraries can be delivered to unlock the full potential across multiple runtimes and deployment modes.

The tiered build order from the tau-runtime concept document remains correct:
- **Tier 0** (tau-core restructure + tau-agent extraction) establishes the type vocabulary
- **Tier 1** (tau-memory, tau-tools, tau-session) provides the foundation primitives demonstrated here
- **Tier 2** (tau-skills, tau-mcp) adds the integration layer
- **Tier 3** (tau-runtime) composes everything into the full runtime

This classify-docs implementation serves as a reference target: when tau-runtime is complete, it should be able to execute this same workflow with the same prompts, the same graph topology, and the same scoring logic — but with the added benefits of provider flexibility, persistent checkpointing, API access, and deployment options that Claude Code alone cannot provide.

---

## 11. Implementation Milestones

### Milestone 1: Repository Initialization
- Create `classify-docs` repository
- Write README.md with project overview and usage instructions
- Create `.gitignore` (ignore `.work/` directory)
- Create `.claude/settings.json` with project configuration

### Milestone 2: Memory Bootstrap
- Write `.claude/CLAUDE.md` with domain knowledge, workflow overview, and operational instructions (Section 3)

### Milestone 3: Core Skills
- Create `render-pages` skill (Section 4.2) — PDF→PNG conversion
- Create `detect-markings` skill (Section 4.3) — Per-page vision analysis
- Create `enhance-page` skill (Section 4.6) — Conditional enhancement
- Create `classify-document` skill (Section 4.4) — Classification determination
- Create `score-confidence` skill (Section 4.5) — Confidence assessment

### Milestone 4: Subagents
- Create `page-detector` agent (Section 5.1) — Parallel page analysis
- Create `classification-analyst` agent (Section 5.2) — Classification + scoring

### Milestone 5: Orchestrator
- Create `classify-docs` skill (Section 4.1) — Workflow orchestration
- This is the most complex skill and depends on all others being in place

### Milestone 6: Validation
- Test with a single-page PDF containing clear markings
- Test with a multi-page PDF requiring enhancement
- Test with a directory of PDFs using `--recurse`
- Test error cases: missing ImageMagick, non-PDF file, empty directory
- Verify detection, classification, and scoring output match expected formats

---

## Appendix A: System Prompt Preservation

The three system prompts from agent-lab (`profile.go`) are preserved in their essential content within the corresponding skills. The only adaptations are:

1. **File I/O instructions**: Added guidance to read input from and write output to filesystem paths, since the Claude Code implementation uses files instead of in-memory state
2. **Removal of "JSON response only" suffix**: Skills naturally produce structured output; the instruction to avoid preamble is handled by skill context

The domain logic — marking detection rules, classification hierarchy, scoring factors and weights, recommendation thresholds — is identical between implementations.

## Appendix B: Revision History

| Date | Revision | Trigger |
|---|---|---|
| 2026-02-04 | Initial concept document | Concept development session |
