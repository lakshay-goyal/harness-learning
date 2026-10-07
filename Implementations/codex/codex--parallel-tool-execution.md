---
type: implementation
harness: codex
concept: parallel-tool-execution
commit: 622e9e3696
files: [codex-rs/core/src/session/turn.rs:1576, codex-rs/core/src/session/turn.rs:2460-2488, codex-rs/core/src/session/turn.rs:2580, codex-rs/core/src/session/turn.rs:2774-2784, codex-rs/core/src/session/turn.rs:3156-3173, e95abcdf49:codex-rs/core/src/tools/parallel.rs:46-66, e95abcdf49:codex-rs/core/src/tools/parallel.rs:196-218, codex-rs/tools/src/tool_executor.rs:122-124, codex-rs/core/src/tools/registry.rs:532-535, codex-rs/core/src/tools/handlers/mcp.rs:142-150, codex-rs/core/src/client.rs:1001]
---
[[parallel-tool-execution]] in [[codex]].

## Mechanism
- Request flag: every prompt sets `parallel_tool_calls: true` (`codex-rs/core/src/session/turn.rs:1576`); forced false for Responses Lite models (`codex-rs/core/src/client.rs:1001`). Per-model `supports_parallel_tool_calls` removed from the catalog by `86b1123ff6` 2026-08-14 "Enable parallel tool calls for all model prompts".
- **Tools start while the response is still streaming**: each `OutputItemDone` tool call becomes a spawned future pushed into a `FuturesOrdered` (`turn.rs:2580,2774-2784`); after the stream ends `drain_in_flight` awaits them **in model order** and records outputs to history (`turn.rs:2460-2488,3156-3165`). Token-count event emitted only after tools resolve (so a paused `request_user_input` doesn't show progress, `turn.rs:3167-3173`). A failed in-flight future during drain hits `error_or_panic` ("in-flight tool future failed during drain") rather than failing the turn (`turn.rs:2480-2483`).
- **Concurrency gate** = per-sampling-request `Arc<RwLock<()>>`: tools whose runtime `supports_parallel_tool_calls()` take a **read** lock, all others the **write** lock (exclusive) (`e95abcdf49:codex-rs/core/src/tools/parallel.rs:46-66,207-217`). A non-parallel tool waits for running readers and blocks later ones — a reader/writer barrier inside one batch, not per-file serialization ([[per-file-mutation-queue]] absent).
- Each call `tokio::spawn`ed under `AbortOnDropHandle`; readiness wait (`wait_until_ready`, e.g. MCP server startup) and lock acquisition are cancellable even for tools that must finish (`parallel.rs:196-218`) → [[abort-propagation]].
- Per-tool flag, **default false** (`codex-rs/tools/src/tool_executor.rs:122-124`). Opt-in true: `exec_command`, `write_stdin`, `view_image`, `tool_search`, MCP resource tools (`codex-rs/core/src/tools/handlers/unified_exec/exec_command.rs:142`, `codex-rs/core/src/tools/handlers/view_image.rs:81`, `codex-rs/core/src/tools/handlers/tool_search.rs:197`, `codex-rs/core/src/tools/handlers/mcp_resource/read_mcp_resource.rs:35`). Explicit false: plugin install tools. Default (exclusive): `apply_patch`, `update_plan`, `request_user_input`.
- Registry: parallel iff `exposure != Hidden && runtime.supports_parallel_tool_calls()` (`codex-rs/core/src/tools/registry.rs:532-535`).
- MCP tools parallel only when annotation `readOnlyHint == true` or server config `supports_parallel_tool_calls` opts in; cached (possibly stale) catalogs clear the read-only hint until the live server starts (`codex-rs/core/src/tools/handlers/mcp.rs:142-150`, tests `:880-915`) → [[unannotated-mcp-tools-serialized]], [[tool-safety-annotations]].
- Code-mode nested calls go through the same gate (`ToolCallSource::CodeMode`) → [[nested-tool-calls]].
- Approvals keyed by request/call id, not turn (`c4b771a16f`) → [[approval-scope-too-broad-in-parallel-batch]].
- Prompt side: "Parallelize tool calls whenever possible… Use `multi_tool_use.parallel` to parallelize tool calls and only this." (`238ce7dfad` 2025-12-11) → newest catalog text "Batch independent searches and reads in one functions.exec using await Promise.allSettled([...])" (code-mode era, M5a).

## Constants
| name | value | path:line |
|---|---|---|
| `parallel_tool_calls` request flag | always true (false for Responses Lite) | `codex-rs/core/src/session/turn.rs:1576`; `codex-rs/core/src/client.rs:1001` |
| per-tool default | not parallel | `codex-rs/tools/src/tool_executor.rs:122-124` |
| analytics tool-call ids per response | 256 | `codex-rs/core/src/session/turn.rs:2588` |

## Evolution
- 2025-10-05 `dc3c6bf62a` "feat: parallel tool calls" (landed off) → 2025-10-07 `f2555422b9` "Simplify parallel (#4829)" introduces the RwLock gate → 2025-11-18 `f5d9939cda` enabled → 2025-11-19 `4985a7a444` instruction-injection fix → 2025-12-16 `d802b18716` "fix parallel tool calls" → `5f80ad6da8`/`649badd102` chat-completions parallel calls → 2026-02-10 `c4b771a16f` approvals by request id ([[parallel-tools-reveal-latent-bugs]]).
- 2025-10-05 `f3b4a26f32` `read_file` dropped for gpt-5-codex, commit noting "will do the same for parallel tool call".
- 2026-05-12 `862b2122ee` "tools: remove is_mutating dispatch gating" ("That second hook no longer carried its weight").
- 2026-05-21 `c83ba22359` read-only MCP tools parallel.
- 2026-08-14 `86b1123ff6` parallel for all model prompts.

## Versus pi
pi runs a whole batch concurrently after a sequential preflight; one sequential tool serializes the whole batch; per-file queue for edit/write ([[pi--parallel-tool-execution]], [[pi--per-file-mutation-queue]]). codex mixes read/write locks per tool inside one batch, starts tools during streaming, and has no file-level lock ([[no-per-file-mutation-queue]]; shell commands are parallel-safe even though they can mutate files).
