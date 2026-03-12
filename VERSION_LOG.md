# PRISM Version Log

## v3.0 — Initial Release
**Date**: 2026-03-12  
**Iteration**: 0 (baseline)

### Scores (baseline — test battery: docx, research-agent, data-pipeline, email-classifier, file-converter)
| Metric | Score |
|---|---|
| M1 Reasoning Chain Quality | 82 |
| M2 Constraint Adherence | 88 |
| M3 Ambiguity Score | 79 |
| M4 Semantic Density | 74 |
| M5 Failure Coverage | 71 |
| M6 Branch Completeness | 85 |
| M7 Cognitive Learnability | 76 |
| **Composite** | **80.4** |

### Notes
- v3.0 is the first formally specified version, evolved from SKL v1 and FLUX v2
- Lowest metric: M5 Failure Coverage (71) — `fail:` fields tend to be formulaic
- Second lowest: M3 Ambiguity (79) — referential ambiguity in `think:` fields
- Recommended first iteration target: M5, then M3
