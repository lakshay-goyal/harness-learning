---
type: implementation
harness: opencode
concept: tool-call-id-normalization
commit: ecc4916b5a
files: [packages/opencode/src/provider/transform.ts:224-301, packages/llm/src/protocols/gemini.ts:441-459]
---
[[tool-call-id-normalization]] in [[opencode]].

## Mechanism

### Legacy runtime (`ProviderTransform.normalizeMessages`)
- Claude: `scrub(id) = id.replace(/[^a-zA-Z0-9_-]/g, "_")` on tool-call and tool-result ids (`packages/opencode/src/provider/transform.ts:224-251`).
- Mistral family (mistral, devstral, codestral, pixtral, mixtral…): ids → alphanumeric, first 9 chars, `padEnd(9, "0")` (`transform.ts:255-264`).
- Mistral role order: a synthetic assistant `"Done."` is inserted between a tool message and a following user message (`transform.ts:285-298`).

### v2 runtime
- Gemini function calls carry no id → parser synthesizes `tool_0, tool_1, …` per stream (`packages/llm/src/protocols/gemini.ts:441-459`). Ids repeat across turns; the v2 store binds settlement to the assistant message id.
- Tool results matched to Gemini calls by **name** in `functionResponse` (`gemini.ts:250-284`).

## Constants
| name | value | path:line |
|---|---|---|
| Mistral id length | 9 | `packages/opencode/src/provider/transform.ts:262` |

## Evolution
- 2026-05-25 `87e9e700cd` Google SDK tool-call id change reverted.
- 2026-07-20 `3033afba51` Mistral family check matched only "mistral"/"devstral"; codestral/pixtral/mixtral added.

## Quirks / drift
- Truncate-and-pad (no hash) means two ids sharing 9 leading alphanumerics collide (inference; no reported incident).

Failures: [[cross-provider-tool-call-id-normalization]] · [[tool-call-id-collision]] · [[capability-sniffing-misses-opaque-ids]].

Contrast: [[pi--tool-call-id-normalization|pi]] hashes over-long ids and keeps bidirectional maps for Mistral; opencode truncates and pads.
