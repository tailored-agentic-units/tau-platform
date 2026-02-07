# Aggregate Classification Report

**Directory**: ./marked-documents
**Documents Processed**: 27
**Date**: 2026-02-05
**Run ID**: 20260205-075202

## Summary

| # | Document | Classification | Confidence | Recommendation |
|---|----------|---------------|------------|----------------|
| 1 | marked-documents_1.pdf | SECRET//SPECAT | 0.92 | ACCEPT |
| 2 | marked-documents_2.pdf | SECRET | 0.96 | ACCEPT |
| 3 | marked-documents_3.pdf | SECRET//NOFORN | 0.98 | ACCEPT |
| 4 | marked-documents_4.pdf | SECRET//NOFORN | 0.865 | REVIEW |
| 5 | marked-documents_5.pdf | UNCLASSIFIED | 0.41 | REJECT |
| 6 | marked-documents_6.pdf | SECRET//NOFORN | 0.94 | ACCEPT |
| 7 | marked-documents_7.pdf | SECRET//NOFORN | 0.98 | ACCEPT |
| 8 | marked-documents_8.pdf | SECRET | 0.85 | REVIEW |
| 9 | marked-documents_9.pdf | UNCLASSIFIED | 0.73 | REVIEW |
| 10 | marked-documents_10.pdf | SECRET//NOFORN//X1 | 0.985 | ACCEPT |
| 11 | marked-documents_11.pdf | SECRET//NOFORN//X1 | 0.94 | ACCEPT |
| 12 | marked-documents_12.pdf | SECRET//NOFORN | 0.925 | ACCEPT |
| 13 | marked-documents_13.pdf | SECRET//NOFORN | 0.925 | ACCEPT |
| 14 | marked-documents_14.pdf | UNCLASSIFIED | 0.825 | REVIEW |
| 15 | marked-documents_15.pdf | UNCLASSIFIED | 0.88 | REVIEW |
| 16 | marked-documents_16.pdf | SECRET//NOFORN//XI | 0.935 | ACCEPT |
| 17 | marked-documents_17.pdf | SECRET | 0.89 | REVIEW |
| 18 | marked-documents_18.pdf | SECRET//NOFORN | 0.87 | REVIEW |
| 19 | marked-documents_19.pdf | SECRET//NOFORN | 0.86 | REVIEW |
| 20 | marked-documents_20.pdf | SECRET//NOFORN//WNINTEL | 0.935 | ACCEPT |
| 21 | marked-documents_21.pdf | SECRET//NOFORN | 1.00 | ACCEPT |
| 22 | marked-documents_22.pdf | SECRET//NOFORN | 0.98 | ACCEPT |
| 23 | marked-documents_23.pdf | SECRET | 1.00 | ACCEPT |
| 24 | marked-documents_24.pdf | CONFIDENTIAL | 0.95 | ACCEPT |
| 25 | marked-documents_25.pdf | SECRET//NOFORN | 0.93 | ACCEPT |
| 26 | marked-documents_26.pdf | SECRET//X1 | 0.955 | ACCEPT |
| 27 | marked-documents_27.pdf | SECRET//NOFORN | 0.84 | REVIEW |

## Statistics

- **ACCEPT**: 17 documents (63%)
- **REVIEW**: 9 documents (33%)
- **REJECT**: 1 document (4%)

### Classification Distribution

| Classification | Count |
|---------------|-------|
| SECRET//NOFORN | 10 |
| SECRET | 4 |
| UNCLASSIFIED | 4 |
| SECRET//NOFORN//X1 | 2 |
| SECRET//NOFORN//XI | 1 |
| SECRET//NOFORN//WNINTEL | 1 |
| SECRET//SPECAT | 1 |
| SECRET//X1 | 1 |
| CONFIDENTIAL | 1 |
| **Classified (SECRET+)** | **20** |
| **Not classified (CONFIDENTIAL or below)** | **5** |

### Confidence Distribution

- **Average confidence**: 0.91
- **Highest**: 1.00 (docs 21, 23)
- **Lowest**: 0.41 (doc 5)

## Documents Requiring Review

### REJECT (Human Verification Required)

| Document | Classification | Confidence | Reason |
|----------|---------------|------------|--------|
| marked-documents_5.pdf | UNCLASSIFIED | 0.41 | No markings detected on heavily redacted document; possible detection failure |

### REVIEW (Human Verification Recommended)

| Document | Classification | Confidence | Reason |
|----------|---------------|------------|--------|
| marked-documents_4.pdf | SECRET//NOFORN | 0.865 | Slight date code variation between header/footer |
| marked-documents_8.pdf | SECRET | 0.85 | Clarity slightly below threshold |
| marked-documents_9.pdf | UNCLASSIFIED | 0.73 | No markings on heavily redacted document |
| marked-documents_14.pdf | UNCLASSIFIED | 0.825 | No markings on heavily redacted document |
| marked-documents_15.pdf | UNCLASSIFIED | 0.88 | No markings detected; high clarity suggests genuine absence |
| marked-documents_17.pdf | SECRET | 0.89 | Single-page, clarity slightly below threshold |
| marked-documents_18.pdf | SECRET//NOFORN | 0.87 | Enhancement not applied |
| marked-documents_19.pdf | SECRET//NOFORN | 0.86 | Footer markings show fading (legibility 0.65-0.75) |
| marked-documents_27.pdf | SECRET//NOFORN | 0.84 | Header marking faded (legibility 0.75) |
