---
type: failure
concepts: [usage-cost-accounting, structured-classifier-api]
harnesses: [pi]
---
**Symptom** Classifier (System One) calls whose answers failed to parse produced an error result with no usage. Spend the provider still billed went untracked.

**Root cause** Usage was attached only after the answers parsed successfully, so a malformed-but-billed response dropped its cost.

**Fix · [[pi]]** Usage is set *before* answer parsing: "malformed answers were still billed" (`packages/ai/src/api/system-one-shared.ts:125-127`). `89a5c7bda` 2026-09-29: `parseClassifierUsage({input_tokens, output_tokens})` priced via `calculateCost` (`packages/ai/src/api/classifier-shared.ts:114-128`). The exact commit that moved usage before parsing is unverified.

**Lesson** Record cost as soon as the provider reports it, before any validation that can fail.

Related: [[usage-cost-accounting]] · [[structured-classifier-api]] · [[streamed-usage-misread]] · [[pi--structured-classifier-api|pi]]
