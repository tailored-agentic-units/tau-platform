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
