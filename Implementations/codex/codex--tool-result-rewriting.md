---
type: implementation
harness: codex
concept: tool-result-rewriting
commit: 622e9e3696
files: [codex-rs/core/src/tools/registry.rs:238-263, codex-rs/core/src/tools/registry.rs:735-815, codex-rs/hooks/src/engine/dispatcher.rs:115-160, codex-rs/protocol/src/protocol.rs:1624-1646]
---
[[tool-result-rewriting]] in [[codex]] — via user-configured **PostToolUse hooks** (external processes, Claude-Code-compatible protocol), not in-process callbacks.

## Mechanism
- After a **successful** handler run with a `post_tool_use_payload`, `run_post_tool_use_hooks(tool_use_id, tool_name, matcher_aliases, tool_input, tool_response)` runs (cancellable) (`codex-rs/core/src/tools/registry.rs:735-760`). Failed tool calls do not reach PostToolUse.
- Outcomes:
  - `additional_contexts` → recorded into the conversation as extra context (`record_additional_contexts`, `registry.rs:762-770`).
  - `should_block` → the result is rejected (not the already-completed execution): `RespondToModel(feedback_message || "PostToolUse hook blocked the tool result")` (`registry.rs:772-795`) → [[tool-error-as-result]].
  - `feedback_message` without block → model-visible output **replaced** by `PostToolUseFeedbackOutput{original, model_visible}`: logs/telemetry/token-limit overrides keep the original, the response item sent to the model is the feedback text (`registry.rs:238-263,796-806`).
- No field-level patch chain: one aggregated outcome per call; synchronous handlers for an event run concurrently (`FuturesUnordered`) and results are re-ordered by config order (`codex-rs/hooks/src/engine/dispatcher.rs:115-160`) → [[turn-lifecycle-hooks]].
- Handler types Command / McpTool supported; Prompt / Agent skipped at load ("prompt hooks are not supported yet") (`codex-rs/protocol/src/protocol.rs:1641-1646`; `codex-rs/hooks/src/engine/discovery.rs:637-656`).
- Default function tools participate in hooks (`5c20513a1b` 2026-05-23 "Default function tools into tool hooks"); `apply_patch` exposes `{command: <patch>}` as hook input ([[patch-envelope-edit]]).
- Harness-owned result rewriting is separate and fixed: truncation ([[tool-output-truncation]]), MCP image omission for text-only models, abort synthesis.

## Evolution
- 2026-03-23 `73bbb07ba8` PreToolUse (shell-only, non-streaming); 2026-03-25 `c4d9887f9a` PostToolUse (shell-only).
- 2026-05-23 `5c20513a1b` default function tools into hooks. 2026-06-15 `d7f298fe20` blocking PostToolUse in code mode.
- 2026-08-15 `85fc4def35` MCP tool hook handlers.

## Versus pi
pi: in-process `afterToolCall` with field-level merge chained across extension `tool_result` handlers, also on errors ([[pi--tool-result-rewriting]]); its failure [[tool-result-hook-patches-lost]] (last handler won) maps to codex's single aggregated outcome. codex: external-process hooks, success-only, block / replace-text / add-context.
