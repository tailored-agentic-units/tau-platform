---
name: render-pages
description: >
  Convert PDF document pages to PNG images using ImageMagick.
  Supports standard rendering and enhancement re-rendering with
  brightness, contrast, and saturation adjustments.
user-invocable: false
allowed-tools: Bash, Read, Write
---

# Render Pages

Convert PDF pages to PNG images at 300 DPI using the `magick` CLI.

## Standard Rendering (All Pages)

Given a PDF path and output directory, render every page:

```bash
# Get page count
PAGE_COUNT=$(magick identify -format "%n\n" "$PDF_PATH" | head -1)

# Render each page (0-indexed for ImageMagick, 1-indexed for output)
for i in $(seq 0 $((PAGE_COUNT - 1))); do
  PAGE_NUM=$((i + 1))
  magick -density 300 "${PDF_PATH}[$i]" \
    -background white -flatten \
    "${OUTPUT_DIR}/page-${PAGE_NUM}.png"
done
```

After rendering, verify all expected page images exist.

## Enhancement Re-Rendering (Specific Page with Filters)

Given a PDF path, page index (0-based), output directory, and filter parameters:

```bash
magick -density 300 "${PDF_PATH}[$PAGE_INDEX]" \
  -background white -flatten \
  -modulate ${BRIGHTNESS},${SATURATION} \
  -brightness-contrast ${CONTRAST} \
  "${OUTPUT_DIR}/enhanced-page-${PAGE_NUM}.png"
```

## Parameters

- **DPI**: Fixed at 300 (required for classification marking clarity)
- **Format**: PNG (lossless quality)
- **Page indexing**: ImageMagick uses 0-based indexing; output filenames are 1-based
- **Filter parameters**:
  - `brightness`: 0-200 (100 = neutral)
  - `contrast`: -100 to 100 (0 = neutral)
  - `saturation`: 0-200 (100 = neutral)
- Only include `-modulate` and `-brightness-contrast` flags when enhancement values differ from neutral
