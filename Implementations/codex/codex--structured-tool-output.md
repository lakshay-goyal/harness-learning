---
type: implementation
harness: codex
concept: structured-tool-output
commit: 622e9e3696
files: [codex-rs/tools/src/tool_output.rs:11-60, codex-rs/tools/src/responses_api.rs:33-45, codex-rs/core/src/tools/handlers/shell_spec.rs:197-228, codex-rs/core/src/tools/context.rs:483-560, codex-rs/protocol/src/models.rs:2304-2338, codex-rs/core/src/tools/handlers/current_time.rs:70-80]
---
[[structured-tool-output]] in [[codex]] — one output object, several projections: model-facing response item (plain text), code-mode JSON result, PostToolUse hook payload, lossy log string.

## Mechanism
- `ToolOutput` trait, "Model-facing output contract returned by executable tool runtimes" (`codex-rs/tools/src/tool_output.rs:11-60`):
  - `to_response_item(call_id, payload)` — what the model sees (wire shape follows the call kind).
  - `code_mode_result(payload) -> JsonValue` — what a code-mode script receives.
  - `post_tool_use_input` / `post_tool_use_response` — hook-facing data; `None` = no PostToolUse payload ("not merely that the tool had empty output").
  - `log_output()` — "deliberately lossy diagnostic representation … not the authoritative tool result".
  - `fallback_token_limit_override()` — tool-specific truncation budget carried into history ([[truncation-budget-drift-on-replay]]); `contains_external_context()` — disables memory generation when `memories.disable_on_external_context` is on ([[cross-session-memory]]); `set_handler_duration_ms`.
- **Output schemas are harness-side only**: `ResponsesApiTool.output_schema` is `#[serde(skip)]` — never sent to the provider; used to type code-mode TypeScript declarations (`codex-rs/tools/src/responses_api.rs:33-45`) → [[code-mode]].
  - `exec_command` output schema `{chunk_id?, wall_time_seconds, exit_code?, session_id?, original_token_count?, output}` (required `wall_time_seconds`, `output`; `additionalProperties: false`) (`codex-rs/core/src/tools/handlers/shell_spec.rs:197-228`).
  - `clock.curr_time` → `{current_time}` (`codex-rs/core/src/tools/handlers/current_time.rs:70-80`); `view_image` → `{image_url}`.
- Model-facing text for exec is plain: `Chunk ID / Wall time / Process exited with code N | Process running with session ID N / Original token count / Output:` (`codex-rs/core/src/tools/context.rs:524-548`); legacy JSON-structured shell output removed (`82061660ae` 2026-05-18: "Current shell and apply_patch responses are already plain text for model consumption").
- MCP: if `structuredContent` is present (non-null) it is serialized **as the text output** and content blocks are dropped; else content blocks → text/image items (`codex-rs/protocol/src/models.rs:2304-2338`) → [[mcp-integration]].
- Dynamic (client) tools return text + image content items (`5ea107a088`) → [[client-supplied-dynamic-tools]].
- No terminating-submit control hint on tool results (no `terminate` equivalent found) — turn ends when the model stops calling tools ([[turn-loop]]); unverified beyond the trait.

## Evolution
- 2026-02-04 `5ea107a088` text + image content items for dynamic tool outputs.
- 2026-03-09 `da616136cc` code mode → output schemas for nested typing.
- 2026-05-18 `82061660ae` legacy structured shell output removed.
- 2026-09-09 `aa88a0333c` truncation budget persisted with each tool output.

## Versus pi
pi sends `content` to the model and `structuredContent` (+ `outputSchema`, `details`, `terminate`) to scripts/extensions, with different size limits ([[pi--structured-tool-output]]). codex: same split for scripts, but MCP `structuredContent` *replaces* content for the model, and output schemas never reach the provider.
