# Implementation Plan: classify-docs Claude Code Infrastructure

## Overview

Create 12 files within `concepts/tau-runtime/claude-classify-docs/` that constitute a complete Claude Code project implementing the classify-docs workflow. This reimplements agent-lab's Go web service (~11,000 lines) as pure Claude Code configuration (~600 lines of markdown/JSON).

**Base directory**: `concepts/tau-runtime/claude-classify-docs/`

**Existing assets**:
- `spec.md` — Full specification (reference, not modified)
- `marked-documents/` — 27 test PDFs for validation

---

## Spec Review: Confirmed Sound

The spec's approach is solid. Key validations:

- **SKILL.md frontmatter fields** (`name`, `description`, `user-invocable`, `argument-hint`, `allowed-tools`) — all standard Claude Code fields
- **agent.md frontmatter fields** (`name`, `description`, `tools`, `model`, `permissionMode`, `skills`) — all valid per Claude Agent SDK
- **No subagent nesting** — correctly avoided; orchestrator spawns agents, agents don't spawn agents
- **Vision via Read tool** — Claude Code's Read tool displays images visually; detect-markings leverages this
- **Parallel Task spawning** — orchestrator sends multiple Task calls in a single message for concurrent page detection
- **System prompts preserved** — verified against agent-lab's `profile.go` (3 prompts at lines 11-99); adapted from "Respond with ONLY a JSON object" to "Write a JSON file" for file I/O

**One minor adaptation**: The original prompts end with "JSON response only; no preamble or dialog". This suffix is removed in skill versions since skills naturally produce structured output through file writes rather than direct response.

---

## File Creation Plan (12 files)

### Phase 1: Project Infrastructure

**File 1: `.gitignore`**
- Content: `.work/`
- 1 line

**File 2: `README.md`** (~100-120 lines)
- Purpose and importance framing (TAU ecosystem proof-of-concept demonstrating memory/skills/subagents compose into a functional agent runtime)
- Prerequisites (Claude Code, ImageMagick 7.0+)
- Usage: `cd classify-docs && claude` → `/classify-docs <path> [--recurse]`
- Architecture diagram (from spec section 2.3)
- File structure with brief descriptions
- Working directory layout explanation
- Validation section referencing `marked-documents/` test set
- Link to `spec.md` for deep design rationale

### Phase 2: Bootstrap Memory

**File 3: `.claude/CLAUDE.md`** (~80 lines)
- Verbatim from spec section 3.1
- Domain knowledge: classification hierarchy, caveats, marking locations, critical rules
- Workflow overview (5 stages)
- System requirements (ImageMagick)
- Working directory conventions

### Phase 3: Configuration

**File 4: `.claude/settings.json`**
- Permissions for: `Bash(magick *)`, `Bash(mkdir *)`, all skills, all agents
- Plans directory: `.claude/plans`

### Phase 4: Standalone Skills (5 files, independent of each other)

**File 5: `.claude/skills/render-pages/SKILL.md`** (~40 lines)
- `user-invocable: false`, `allowed-tools: Bash, Read, Write`
- Standard rendering: `magick -density 300` for all pages
- Enhancement rendering: `-modulate` and `-brightness-contrast` with filter params
- DPI fixed at 300, PNG format, 0-based ImageMagick indexing → 1-based filenames

**File 6: `.claude/skills/detect-markings/SKILL.md`** (~50 lines)
- `user-invocable: false`, `allowed-tools: Read, Write`
- Preserves DetectionSystemPrompt from `agent-lab/workflows/classify/profile.go:11-40`
- Adapted for file I/O: read image at path, write JSON to output path
- JSON schema: page_number, markings_found[], clarity_score, filter_suggestion

**File 7: `.claude/skills/classify-document/SKILL.md`** (~45 lines)
- `user-invocable: false`, `allowed-tools: Read, Write`
- Preserves ClassificationSystemPrompt from `profile.go:42-68`
- Reads all detection JSONs from directory, writes classification.json
- JSON schema: classification, alternative_readings[], marking_summary[], rationale

**File 8: `.claude/skills/score-confidence/SKILL.md`** (~50 lines)
- `user-invocable: false`, `allowed-tools: Read, Write`
- Preserves ScoringSystemPrompt from `profile.go:70-99`
- 6 weighted factors: marking_clarity(0.30), marking_consistency(0.25), spatial_coverage(0.15), enhancement_impact(0.10), alternative_count(0.10), detection_legibility(0.10)
- Thresholds: >=0.90 ACCEPT, 0.70-0.89 REVIEW, <0.70 REJECT

**File 9: `.claude/skills/enhance-page/SKILL.md`** (~80 lines)
- `user-invocable: false`, `allowed-tools: Bash, Read, Write`
- Runs in main session (not subagent) — has full vision + bash access
- Steps: read original detection → extract filter_suggestion → re-render with magick filters → read enhanced image + re-detect markings → merge original + enhanced detections
- Detection instructions included inline (duplicated from detect-markings for self-containment)
- Merge algorithm: keep higher clarity_score; for each marking matched by text+location, prefer enhanced if original was faded or legibility < 0.4

### Phase 5: Subagent Definitions (2 files)

**File 10: `.claude/agents/page-detector/agent.md`** (~25 lines)
- `model: sonnet`, `permissionMode: bypassPermissions`
- `tools: Read, Write, Bash, Glob`
- `skills: [detect-markings]`
- System prompt: analyze single page, focus exclusively on assigned page, write JSON

**File 11: `.claude/agents/classification-analyst/agent.md`** (~30 lines)
- `model: sonnet`, `permissionMode: bypassPermissions`
- `tools: Read, Write, Glob, Grep`
- `skills: [classify-document, score-confidence]`
- System prompt: execute classify then score sequentially, both must complete

### Phase 6: Orchestrator Skill

**File 12: `.claude/skills/classify-docs/SKILL.md`** (~150-200 lines)
- User-invocable (default), `argument-hint: <pdf-path-or-directory> [--recurse]`
- `allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Task`
- Full workflow orchestration logic:
  1. **Input validation** — verify PDF exists or discover PDFs in directory
  2. **Working directory setup** — `.work/<YYYYMMDD-HHMMSS>/` with subdirs
  3. **Render pages** — `magick identify` for count, then render-pages skill pattern
  4. **Parallel detection** — spawn N `page-detector` subagents via Task (single message)
  5. **Enhancement evaluation** — check legibility < 0.4 AND filter_suggestion present
  6. **Conditional enhancement** — invoke enhance-page for flagged pages
  7. **Classification + scoring** — spawn `classification-analyst` subagent
  8. **Report generation** — compile report.md with all results
  9. **Output** — display report; for directories, aggregate summary

---

## Verification Strategy

Test using the 27 PDFs in `marked-documents/`. Manually inspect results against the PDF contents.

| Test | Input | Validates |
|------|-------|-----------|
| Smoke test | `marked-documents/marked-documents_1.pdf` | Full single-doc pipeline end-to-end |
| No markings | `marked-documents/marked-documents_2.pdf` | UNCLASSIFIED handling, empty detections |
| Caveats | `marked-documents/marked-documents_3.pdf` | Caveat aggregation (NOFORN + declassification date) |
| Directory batch | `marked-documents/ --recurse` | Directory discovery, sequential multi-doc processing |
| Missing magick | Temp remove from PATH | Error handling for missing dependency |
| Non-PDF input | Point to a .txt file | Input validation / rejection |

---

## Key Source Files Referenced

- `concepts/tau-runtime/claude-classify-docs/spec.md` — Full specification (sections 3-6 contain verbatim content)
- `/home/jaime/code/agent-lab/workflows/classify/profile.go:11-99` — Three system prompts to preserve
- `/home/jaime/code/agent-lab/.claude/skills/*/SKILL.md` — Format conventions (name + description frontmatter, "When This Skill Applies" sections)
- `/home/jaime/code/agent-lab/.claude/settings.json` — Permissions format reference
