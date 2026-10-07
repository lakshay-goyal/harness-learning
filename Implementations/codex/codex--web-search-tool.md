---
type: implementation
harness: codex
concept: web-search-tool
commit: 622e9e3696
files: [codex-rs/core/src/tools/hosted_spec.rs:14-48, codex-rs/core/src/tools/spec_plan.rs:677-706, codex-rs/core/src/tools/spec_plan.rs:1104-1112, codex-rs/ext/web-search/src/tool.rs:41-44, codex-rs/ext/web-search/web_run_description.md, codex-rs/protocol/src/config_types.rs:376-382, codex-rs/tools/src/tool_spec.rs:33-53]
---
[[web-search-tool]] in [[codex]].

## Mechanism
### Hosted (server-executed) `web_search`
- `hosted_model_tool_specs` (`codex-rs/core/src/tools/spec_plan.rs:677-706`): returns nothing for Responses Lite ("accepts schemas for client-executed tools, not hosted Responses tools"); emits hosted search only when the standalone `web.run` tool is NOT registered and `provider.capabilities().web_search` is true.
- `create_web_search_tool` (`codex-rs/core/src/tools/hosted_spec.rs:14-48`) maps `WebSearchMode`:
  | mode | `external_web_access` | `indexed_web_access` |
  |---|---|---|
  | Cached (default, `codex-rs/protocol/src/config_types.rs:376-382`) | false | — |
  | Indexed | true | true ("restricts live fetches to indexed URLs") |
  | Live | true | — |
  | Disabled / none | tool omitted | |
  plus `filters`, `user_location`, `search_context_size` from `web_search_config`; `search_content_types ["text","image"]` when model `web_search_tool_type = TextAndImage`.
- Wire `ToolSpec::WebSearch` with source TODO "Understand why we get an error on web_search although the API docs say it's supported" (`codex-rs/tools/src/tool_spec.rs:33-53`) → [[tool-wire-kinds]].
- `WebSearchMode::restrict_to` intersects requested mode with a ceiling (Disabled wins) (`config_types.rs:384-388`); requirements expose `allowed_web_search_modes` (M7) → [[layered-settings]].
- UI only: `codex-rs/core/src/web_search.rs` formats action details (search / open_page / find); `open_page` is a search action, not a fetch tool (`codex-rs/protocol/src/models.rs:1966`, M8).

### Standalone client-executed `web.run` (extension)
- Namespace `web`, tool `run`, description = `web_run_description.md` (7,507 bytes, "Tool for accessing the internet." + command examples) (`codex-rs/ext/web-search/src/tool.rs:41-44`).
- Commands: `search_query`, `image_query`, `open`, `click`, `find`, `screenshot`, `finance`, `weather`, `sports`, `time` (description file); proxied through the model provider's HTTP client to the OpenAI backend; result payload size metric `codex.web_search.results.payload_bytes` (`tool.rs:45`).
- Enabled when `provider.capabilities().web_search && (model.use_responses_lite || Feature::StandaloneWebSearch)` (`spec_plan.rs:1104-1112`); presence suppresses the hosted tool.
- Its schema is exempt from lossy compaction (`9fe55d68e6`) → [[tool-schema-normalization]].

### Sibling extension: image generation
- `image_gen.imagegen` extension: model `gpt-image-2`, ≤5 edit images, ≤32 MiB output (`codex-rs/ext/image-generation/src/tool.rs:58-63`); gated on `Feature::ImageGeneration` + provider `image_generation` capability + image-input model (`spec_plan.rs:772-800`).

### Off for side agents
- Review sub-agent disables web search (`codex-rs/core/src/tasks/review.rs:99-135`, M1) → [[review-subagent]].

## Constants
| name | value | path:line |
|---|---|---|
| default `WebSearchMode` | `Cached` | `codex-rs/protocol/src/config_types.rs:378-379` |
| `web.run` description size | 7,507 bytes | `codex-rs/ext/web-search/web_run_description.md` |
| image gen model / max edit images | `gpt-image-2` / 5 | `codex-rs/ext/image-generation/src/tool.rs:58-59` |
| image gen output cap | 32 MiB | `codex-rs/ext/image-generation/src/tool.rs:58-63` |

## Evolution
- 2025-08-23 `363636f5eb` "Add web search tool (#2371)" (hosted Responses tool).
- 2026-05-11 `d2c3ebac1f` typed extension API → 2026-05-26 `a22706dfae` "standalone websearch extension (#23823)".
- 2026-05-27 `9fe55d68e6` don't compact standalone web-search schema.

## Versus pi
pi deliberately ships no web tools ([[no-web-tools]]); codex has them on by default (Cached mode) whenever the provider supports hosted search.
