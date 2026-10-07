---
type: implementation
harness: codex
concept: tool-wire-kinds
commit: 622e9e3696
files: [codex-rs/tools/src/tool_spec.rs:20-56, codex-rs/tools/src/responses_api.rs:14-81, codex-rs/tools/src/tool_payload.rs:7-11, codex-rs/tools/src/tool_executor.rs:51-78, codex-rs/core/src/tools/handlers/apply_patch.rs:389-391, codex-rs/core/src/tools/registry.rs:609-624]
---
[[tool-wire-kinds]] in [[codex]].

## Mechanism
- `ToolSpec` (`#[serde(tag="type")]`) "When serialized as JSON, this produces a valid "Tool" in the OpenAI Responses API" (`codex-rs/tools/src/tool_spec.rs:18-56`):
  | variant | wire `type` | used by |
  |---|---|---|
  | `Function(ResponsesApiTool)` | `function` | most built-ins, MCP, dynamic tools |
  | `Namespace(ResponsesApiNamespace)` | `namespace` | `clock.*`, `web.*`, `image_gen.*`, `mcp__<server>__` bundles, multi-agent v1 |
  | `ToolSearch{execution, description, parameters}` | `tool_search` | client-executed discovery ([[deferred-tool-loading]]) |
  | `WebSearch{external_web_access, indexed_web_access, filters, user_location, search_context_size, search_content_types}` | `web_search` | provider-hosted search ([[web-search-tool]]); source TODO "Understand why we get an error on web_search although the API docs say it's supported" |
  | `Freeform(FreeformTool)` | `custom` | `apply_patch` ([[patch-envelope-edit]]), code-mode `exec` ([[code-mode]]) |
- `FreeformTool{name, description, defer_loading?, format:{type:"grammar", syntax:"lark", definition}}` (`codex-rs/tools/src/responses_api.rs:16-30`).
- `ResponsesApiTool{name, description, strict, defer_loading?, parameters, output_schema}`; `strict` carries "TODO: Validation" — no harness check that strict schemas are complete; `output_schema` is `#[serde(skip)]` — never sent, used to type code-mode declarations (`responses_api.rs:33-45`) → [[structured-tool-output]].
- Namespaces: members may be function or custom tools (`responses_api.rs:75-81`; `d4fb78bfc5` 2026-08-04 "Support custom tools in namespaces (#36857)"); default namespace description "Tools in the {ns} namespace." (`responses_api.rs:66-72`).
- `LoadableToolSpec` = Function | Namespace — what a `tool_search_output` can return (`responses_api.rs:50-56`).
- Payload kinds the model sends back: `ToolPayload::{Function{arguments}, ToolSearch{arguments}, Custom{input}}` (`codex-rs/tools/src/tool_payload.rs:7-11`). Handlers declare accepted kinds, e.g. `ApplyPatchHandler::matches_kind` → `Custom` only (`codex-rs/core/src/tools/handlers/apply_patch.rs:389-391`). Payload/kind mismatch in `dispatch_any_with_state` is `Fatal` (ends turn) (`codex-rs/core/src/tools/registry.rs:609-624`) → [[tool-error-as-result]].
- Exposure is orthogonal to wire kind: `ToolExposure::{Direct, Deferred, DeferredModelOnly, DirectModelOnly, CodeModeOnly, Hidden}` (`codex-rs/tools/src/tool_executor.rs:51-78`) → [[deferred-tool-loading]], [[code-mode]].
- MCP specs > `MAX_SERIALIZED_MCP_TOOL_BYTES = 8_000` serialized bytes get an open-object input schema (`responses_api.rs:14,139-151`) → [[tool-schema-normalization]].
- Responses Lite models: tools sent as an `AdditionalTools` developer input item instead of top-level `tools`, `parallel_tool_calls` false (`codex-rs/core/src/client.rs:919-1001`, M2) → [[transcript-carried-system-prompt]].

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_SERIALIZED_MCP_TOOL_BYTES` | 8_000 | `codex-rs/tools/src/responses_api.rs:14` |
| freeform grammar syntax | `lark` | `codex-rs/tools/src/responses_api.rs:16-30` |

## Evolution
- 2025-08-22 `236c4f76a6` first freeform tool (apply_patch) → disabled 2025-08-24 `4157788310` → default 2026-05-08 `cce059467a` ([[grammar-constrained-tool-instability]]).
- 2026-02-11 `42e22f3bde` js_repl freeform runtime (removed `8a559e7938` 2026-04-24); 2026-02-20 `73fd939296` grammar blocks wrapped payload prefixes.
- 2026-03-11 `77b0c75267` client-executed `tool_search` replaces the `search_tool_bm25` function tool.
- 2026-04-24 `0db6811b7c` Bedrock: function-style apply_patch because Bedrock rejects `custom` ([[endpoint-rejects-request-field]]).
- 2026-08-04 `d4fb78bfc5` custom tools inside namespaces; 2026-08-05 `431c78eb9c` deferred custom tools in tool search.

## Versus pi
pi declares only JSON-schema functions and adds a Lark grammar for codemode source via its constrained-sampling layer ([[pi--constrained-tool-sampling]]); codex makes "custom" a first-class tool kind and lets the provider host some tools.
