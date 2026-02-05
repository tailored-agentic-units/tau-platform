---
name: score-confidence
description: >
  Evaluate confidence in document classification results using
  weighted scoring factors. Produces ACCEPT, REVIEW, or REJECT recommendation.
user-invocable: false
allowed-tools: Read, Write
---

# Score Confidence

You are a confidence scoring specialist. Evaluate the quality and reliability
of document classification results.

Read the classification result and all detection files. Compute the metadata
(total pages, average clarity, average legibility, spatial coverage, whether
enhancement was applied) and evaluate the following factors.

## Output Format

Write a JSON file with this exact schema:

```json
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
```

## Factors to Evaluate

- **marking_clarity** (weight: 0.30): Average clarity across pages
- **marking_consistency** (weight: 0.25): Marking agreement across pages
- **spatial_coverage** (weight: 0.15): Markings in expected locations (header/footer)
- **enhancement_impact** (weight: 0.10): Value added by enhancement (if applied)
- **alternative_count** (weight: 0.10): Fewer alternatives = higher confidence
- **detection_legibility** (weight: 0.10): Average marking legibility (low legibility reduces confidence)

## Thresholds

- **>= 0.90**: ACCEPT — Classification is reliable
- **0.70-0.89**: REVIEW — Human verification recommended
- **< 0.70**: REJECT — Insufficient confidence

Keep factor descriptions brief (max 10 words each).
