---
type: concept
stage: tools
tier: candidate
aliases: [tool_result event, afterToolCall, afterTool, tool-result patch chain]
harnesses: [pi]
---
Post-execution hooks that patch a tool's result (content, details, structured content, error flag, usage, terminate) before it is appended — for redaction, augmentation, normalization — composed as a chain.

## Why
- Policy/redaction and normalization (e.g. image resizing) need one choke point after every tool, including extension and MCP tools.
- With multiple handlers, last-writer-wins drops earlier patches ([[tool-result-hook-patches-lost]]).
- A throwing post-hook must not abort sibling tools ([[hook-throw-aborts-parallel-batch]]).
- Replacing model-facing content while keeping a stale structured value would desynchronize model vs scripts.

## Design space
- Wrap each tool in decorators (pi before 2026-03) vs **loop-level after-hook** (pi `afterToolCall`).
- Merge semantics: whole-result replace vs **field-level patch, chained across handlers** (pi).
- Hook errors → error result (pi) vs propagate.
- Hooks see errors too (pi) vs success only.
- Harness-owned normalizations live in the same hook (pi: tool-result image normalization) → [[image-normalization]].

## Implementations
- [[pi--tool-result-rewriting|pi]] — agent-core `afterToolCall` field-level merge; coding-agent maps it to chained extension `tool_result` handlers + image normalization.

## Failures
- [[tool-result-hook-patches-lost]]
- [[hook-throw-aborts-parallel-batch]]

## Related
[[tool-call-gate]] · [[extension-event-hooks]] · [[tool-error-as-result]] · [[structured-tool-output]] · [[image-normalization]] · [[context-transform-hook]]
