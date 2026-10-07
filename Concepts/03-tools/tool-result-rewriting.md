---
type: concept
stage: tools
tier: must-have
aliases: [tool_result event, afterToolCall, afterTool, tool-result patch chain, tool.execute.after, PostToolUseFeedbackOutput, post_tool_use_payload]
harnesses: [pi, opencode, codex]
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
- **External-process hooks** (Claude-Code-compatible JSON protocol) with outcomes block / replace model-visible text / add context; success-only (✔ codex PostToolUse) vs in-process field-level patch chain incl. errors (✔ pi).
- Keep the original output for logs/telemetry while the model sees the replacement (✔ codex `PostToolUseFeedbackOutput`).
- One shared mutable output object passed through plugins in order (opencode) vs field-level patches (pi).

## Implementations
- [[pi--tool-result-rewriting|pi]] — agent-core `afterToolCall` field-level merge; coding-agent maps it to chained extension `tool_result` handlers + image normalization.
- [[codex--tool-result-rewriting|codex]] — PostToolUse hooks after successful calls: `should_block` → error result, `feedback_message` → replaces model text, `additional_contexts` → recorded context.
- [[opencode--tool-result-rewriting|opencode]] — plugin `tool.execute.before/after` mutate args/output in place around every tool, MCP tool and code-mode child call.

## Failures
- [[side-door-input-bypasses-hooks]]
- [[tool-result-hook-patches-lost]]
- [[hook-throw-aborts-parallel-batch]]
- [[side-door-input-bypasses-hooks]] (07-safety) — Inputs and tool executions reached the agent through paths that skipped the extension hooks used as the…

## Related
[[tool-call-gate]] · [[extension-event-hooks]] · [[tool-error-as-result]] · [[structured-tool-output]] · [[image-normalization]] · [[context-transform-hook]] · [[turn-lifecycle-hooks]]

## Tradeoffs
- [[lsp-feedback-vs-none]]
