---
name: classify-docs
description: >
  Analyze PDF documents for security classification markings.
  Determines overall classification level and confidence score.
  Use when the user wants to classify a document or directory of documents.
argument-hint: <pdf-path-or-directory> [--recurse]
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Task
---

# Classify Documents

Orchestrate the full document classification workflow: render pages, detect
markings in parallel, conditionally enhance low-quality pages, determine
overall classification, score confidence, and generate a report.

## Step 1: Input Validation

Parse `$ARGUMENTS` for the file/directory path and optional `--recurse` flag.

**If path is a file:**
- Verify the file exists
- Verify it is a PDF (check `.pdf` extension and confirm with `magick identify`)
- If not a PDF or does not exist, report an error and stop

**If path is a directory:**
- Discover PDF files using Glob:
  - Without `--recurse`: `<directory>/*.pdf`
  - With `--recurse`: `<directory>/**/*.pdf`
- If no PDFs found, report an error and stop

## Step 2: Create Working Directory

```bash
RUN_ID=$(date +%Y%m%d-%H%M%S)
WORK_DIR=".work/${RUN_ID}"
mkdir -p "${WORK_DIR}/pages" "${WORK_DIR}/detections" "${WORK_DIR}/enhanced" "${WORK_DIR}/results"
```

For directory processing with multiple PDFs, create per-document subdirectories:
```bash
mkdir -p "${WORK_DIR}/${DOC_NAME}/pages" "${WORK_DIR}/${DOC_NAME}/detections" \
         "${WORK_DIR}/${DOC_NAME}/enhanced" "${WORK_DIR}/${DOC_NAME}/results"
```

## Step 3: Process Each Document

For each PDF, execute steps 3a through 3e. When processing a directory,
documents are processed sequentially (each document may spawn parallel
subagents internally).

### Step 3a: Render Pages

Get the page count:
```bash
PAGE_COUNT=$(magick identify -format "%n\n" "${PDF_PATH}" | head -1)
```

Render all pages to PNG at 300 DPI:
```bash
for i in $(seq 0 $((PAGE_COUNT - 1))); do
  PAGE_NUM=$((i + 1))
  magick -density 300 "${PDF_PATH}[$i]" \
    -background white -flatten \
    "${WORK_DIR}/pages/page-${PAGE_NUM}.png"
done
```

Verify all page images exist by listing `${WORK_DIR}/pages/`.

### Step 3b: Detect Markings (Parallel)

Spawn one `page-detector` subagent per page using the Task tool. All
subagents MUST be spawned in a single message for parallel execution.

For each page N (1 to PAGE_COUNT), spawn a Task with:
- **subagent_type**: `page-detector`
- **prompt**: Include:
  - Image path: `${WORK_DIR}/pages/page-${N}.png`
  - Page number: `${N}`
  - Output path: `${WORK_DIR}/detections/page-${N}.json`

Wait for ALL page detectors to complete.

Read all detection JSON files from `${WORK_DIR}/detections/` and collect
the results.

### Step 3c: Evaluate Enhancement Need

For each detection result, check:
1. Does any marking have `legibility < 0.4`?
2. Is `filter_suggestion` present (not null)?

If BOTH conditions are true for ANY page, that page needs enhancement.

If no pages need enhancement, skip to Step 3d.

### Step 3d: Enhance Pages (Conditional)

For each page that needs enhancement:

1. Read the original detection JSON from `${WORK_DIR}/detections/page-${N}.json`
2. Extract the `filter_suggestion` values
3. Re-render the page with filter adjustments:
   ```bash
   magick -density 300 "${PDF_PATH}[$((N-1))]" \
     -background white -flatten \
     -modulate ${BRIGHTNESS},${SATURATION} \
     -brightness-contrast ${CONTRAST} \
     "${WORK_DIR}/enhanced/page-${N}.png"
   ```
4. Read the enhanced image and re-detect markings (analyze visually for
   security markings using the same detection criteria from CLAUDE.md)
5. Merge original and enhanced detections:
   - Keep the HIGHER clarity_score
   - For each marking matched by text + location: prefer enhanced if
     original was faded or had legibility < 0.4
   - Add any new markings found only in the enhanced detection
   - Deduplicate by text + location
   - Set filter_suggestion to null
6. Write merged detection to `${WORK_DIR}/enhanced/page-${N}.json`

### Step 3e: Classify and Score

Spawn a `classification-analyst` subagent using the Task tool with:
- **subagent_type**: `classification-analyst`
- **prompt**: Include:
  - Detection directory: `${WORK_DIR}/detections/`
  - Enhanced directory (if enhancement was applied): `${WORK_DIR}/enhanced/`
  - Instruction: read detections from both directories; for pages that have
    an enhanced detection, use the enhanced version instead of the original
  - Output directory: `${WORK_DIR}/results/`
  - Metadata: total page count, whether enhancement was applied

Wait for the classification analyst to complete.

Read the results:
- `${WORK_DIR}/results/classification.json`
- `${WORK_DIR}/results/confidence.json`

## Step 4: Generate Report

Write `${WORK_DIR}/report.md` with:

```markdown
# Classification Report

**Document**: <filename>
**Path**: <full path>
**Date**: <current date>
**Run ID**: <run-id>

## Classification

**Overall**: <classification level with caveats>
**Confidence**: <overall_score> (<recommendation>)

## Detection Summary

| Page | Markings Found | Location | Legibility |
|------|---------------|----------|------------|
| 1    | SECRET        | header   | 0.95       |
| ...  | ...           | ...      | ...        |

**Pages Analyzed**: <count>
**Enhancement Applied**: <yes/no, which pages>

## Confidence Factors

| Factor | Score | Weight | Description |
|--------|-------|--------|-------------|
| marking_clarity | 0.92 | 0.30 | ... |
| ... | ... | ... | ... |

## Alternative Readings

<list alternatives with probability and reason, or "None">

## Rationale

<rationale from classification result>
```

## Step 5: Display Results

Display the final report to the user.

For directory processing with multiple documents, after all documents are
processed, compile an aggregate summary:

```markdown
# Aggregate Classification Report

**Directory**: <path>
**Documents Processed**: <count>
**Date**: <current date>

| Document | Classification | Confidence | Recommendation |
|----------|---------------|------------|----------------|
| file1.pdf | SECRET//NOFORN | 0.87 | REVIEW |
| file2.pdf | UNCLASSIFIED | 0.95 | ACCEPT |
| ... | ... | ... | ... |
```

Display the aggregate report to the user.
