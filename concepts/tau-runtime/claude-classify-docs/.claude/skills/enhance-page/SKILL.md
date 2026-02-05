---
name: enhance-page
description: >
  Re-render a document page with brightness, contrast, and saturation
  adjustments, then re-detect markings on the enhanced image. Merges
  original and enhanced detections, keeping the highest-quality results.
user-invocable: false
allowed-tools: Bash, Read, Write
---

# Enhance Page

Re-render a page with filter adjustments, re-detect markings on the enhanced
image, and merge the original and enhanced detection results.

## Inputs

You will receive:
1. The original detection JSON path for the page
2. The PDF file path and page index (0-based)
3. The output directory for enhanced results

## Procedure

### Step 1: Read Original Detection

Read the original detection JSON file for the page. Extract the
`filter_suggestion` values (brightness, contrast, saturation).

### Step 2: Re-Render with Filters

Re-render the page using `magick` with the suggested filter adjustments:

```bash
magick -density 300 "${PDF_PATH}[$PAGE_INDEX]" \
  -background white -flatten \
  -modulate ${BRIGHTNESS},${SATURATION} \
  -brightness-contrast ${CONTRAST} \
  "${OUTPUT_DIR}/enhanced-page-${PAGE_NUM}.png"
```

Only include `-modulate` and `-brightness-contrast` when values differ from
neutral (brightness=100, saturation=100, contrast=0).

### Step 3: Re-Detect Markings on Enhanced Image

Read the enhanced image and analyze it for security classification markings.
Apply the same detection logic:

- Identify ALL security markings (UNCLASSIFIED, CONFIDENTIAL, SECRET,
  TOP SECRET, caveats like NOFORN, ORCON, etc.)
- Note location (header, footer, margin, body)
- Score legibility (0.0-1.0) and faded status
- Score overall clarity (0.0-1.0)

Produce a detection result with the same schema as the original:

```json
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
  "filter_suggestion": null
}
```

### Step 4: Merge Detections

Merge the original and enhanced detection results:

1. **clarity_score**: Keep the HIGHER value between original and enhanced.
2. **For each marking** (matched by text + location):
   - If the original marking was faded OR had legibility < 0.4: use the
     enhanced detection's values for that marking.
   - Otherwise: keep the original detection's values.
3. **New markings**: If the enhanced detection found markings not present in
   the original (by text + location), add them to the merged result.
4. **Deduplicate**: Remove duplicate markings by text + location.
5. **filter_suggestion**: Set to null in the merged result (enhancement already applied).

### Step 5: Write Merged Result

Write the merged detection JSON to `${OUTPUT_DIR}/page-${PAGE_NUM}.json`.
