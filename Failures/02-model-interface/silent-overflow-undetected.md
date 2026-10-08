---
type: failure
concepts: [context-overflow-detection]
harnesses: [pi]
---
**Symptom** Some providers overflow without an error:
- z.ai accepts oversized input and returns `stop`.
- Xiaomi MiMo truncates the input, then returns `length` with 0 output tokens.
- OpenAI reasoning models near the window return `incomplete` with no visible output, and the agent stalls (#7540).
- Ollama truncates silently.

**Root cause** Overflow is only observable through usage accounting, not error text.

**Fix · [[pi]]**
- `a325c1c7d` 2025-12-06: `isContextOverflow` introduced with the regex table plus silent overflow. When `stopReason==="stop"` and `input+cacheRead > contextWindow`, it counts as overflow (`packages/ai/src/utils/overflow.ts:152-158`). `cacheWrite` is not included; the miner flagged that as possibly intentional.
- `a44622670` 2026-05-02: MiMo length-stop overflow, `length && output===0 && input+cacheRead ≥ 0.99·contextWindow` (`overflow.ts:160-168`).
- `32850ef7c` 2026-08-03 first tried "prompt usage within 1% of window". It was replaced by `isRecoverableLength`: a length stop with output below the intended max gets one compact-and-retry (`overflow.ts:173-181`). Only Responses `max_output_tokens` counts as a length stop (#7540). See [[length-stop-recovery]].
- Ollama silent truncation is explicitly undetectable (`overflow.ts:118-120`).
- Consumers guard against the same model and pre-compaction usage (`packages/coding-agent/src/core/agent-session.ts:2957-3008`).

**Lesson** Overflow detection needs the model's context window and the usage numbers, not just error text. Compare against the *intended* output limit rather than a window estimate.

Related: [[context-overflow-detection]] · [[overflow-recovery]] · [[overflow-message-not-recognized]] · [[length-stop-recovery]] · [[pi--context-overflow-detection|pi]]
