---
type: implementation
harness: codex
concept: tool-output-truncation
commit: 622e9e3696
files: [codex-rs/models-manager/models.json:15, codex-rs/models-manager/src/model_info.rs:32, codex-rs/models-manager/src/model_info.rs:126, codex-rs/core/src/context_manager/history.rs:514, codex-rs/core/src/context_manager/history.rs:566, codex-rs/utils/output-truncation/src/lib.rs:17, codex-rs/utils/output-truncation/src/lib.rs:23, codex-rs/utils/output-truncation/src/lib.rs:44, codex-rs/utils/output-truncation/src/lib.rs:164, codex-rs/utils/string/src/truncate.rs:156, codex-rs/core/src/unified_exec/head_tail_buffer.rs:11, codex-rs/core/src/unified_exec/mod.rs:79, codex-rs/core/src/tools/context.rs:483, codex-rs/core/src/tools/context.rs:524, codex-rs/core/src/tools/registry.rs:220]
---
[[tool-output-truncation]] in [[codex]].

## Mechanism
- **Policy per model** from the catalog: `truncation_policy {mode: tokens|bytes, limit}`; all shipped models `tokens/10000` (`codex-rs/models-manager/models.json:15-18`); unknown models `bytes/10_000` (`codex-rs/models-manager/src/model_info.rs:126`). User `tool_output_token_limit` overrides the limit, keeping the model's mode (bytes = tokens × 4) (`codex-rs/models-manager/src/model_info.rs:32-43`).
- **Choke point = history recording**: `record_item_with_metadata` truncates every `FunctionCallOutput` / `CustomToolCallOutput` with `policy × 1.2` ("20% allowance for serialization and headers") unless the item carries `history_truncation_token_limit` metadata, which is used as-is ("The override already includes the tool's serialization allowance") (`codex-rs/core/src/context_manager/history.rs:566-574`; `codex-rs/utils/output-truncation/src/lib.rs:17-21`). Tools set the override per result (`codex-rs/core/src/tools/registry.rs:220-224`).
- **Rollout keeps full payloads**: "Tool output truncation applies only to live history, preserving full rollout payloads" (`codex-rs/core/src/context_manager/history.rs:514-515`); resume replays through the same function, so the persisted per-item limit (if present) is re-applied, else the current model's policy (`replay_annotated_item`, `:531-550`) → [[truncation-budget-drift-on-replay]]. Model is never pointed at the full payload ([[no-tool-output-spill-file]]).
- **Shape**: head + tail kept, middle replaced by `…N tokens truncated…` / `…N chars truncated…` (`codex-rs/utils/string/src/truncate.rs:156-176`); formatted variant prefixes `Warning: truncated output (original token count: N)\nTotal output lines: L` (`codex-rs/utils/output-truncation/src/lib.rs:23-41`).
- **Multi-part outputs**: text/audio items share one budget in order; overflow items replaced by "[omitted N text items ...]" / "[omitted N audio items ...]"; images always kept, not charged (`codex-rs/utils/output-truncation/src/lib.rs:164-250`).
- **MCP results**: oversized serialized result → single text preview preserving `isError`; budget scaled iteratively because JSON escaping inflates the preview (`codex-rs/utils/output-truncation/src/lib.rs:44-89`).
- **Exec (unified exec)**: (1) collection `HeadTailBuffer` capped at `UNIFIED_EXEC_OUTPUT_MAX_BYTES` = 1 MiB, half head / half tail, dropped middle `... N bytes omitted ...` (`codex-rs/core/src/unified_exec/head_tail_buffer.rs:11-19`; `codex-rs/core/src/unified_exec/mod.rs:80`, `:231-233`); (2) model budget `max_output_tokens` (default 10000, `codex-rs/core/src/unified_exec/mod.rs:79`) capped by the model policy, whichever is smaller (`codex-rs/core/src/tools/context.rs:483-491`). Result text `Chunk ID … Wall time … Process exited with code N | Process running with session ID N … Original token count: N … Output:` (`:524-548`); code-mode callers get structured JSON with `output_schema` (`codex-rs/core/src/tools/handlers/shell_spec.rs:197-226`).
- **User shell commands** (`!cmd`) are truncated with the model's policy before entering history as `<user_shell_command>` (`codex-rs/core/src/user_shell_command.rs:9-40`) → [[user-shell-escape]].
- **Persistence caps** (separate policy): persisted MCP results / command output in rollout events capped at 64 KiB with "\n... command output truncated for persistence ...\n" (`codex-rs/rollout/src/policy.rs:16-19`); event copies of MCP results 1 MiB (`codex-rs/core/src/mcp_tool_call.rs:125`) → [[unbounded-payload-in-transcript]].

## Constants
| name | value | path:line |
|---|---|---|
| catalog truncation policy | tokens 10_000 per model | `codex-rs/models-manager/models.json:15-18` |
| unknown-model policy | bytes 10_000 | `codex-rs/models-manager/src/model_info.rs:126` |
| serialization allowance | × 1.2 | `codex-rs/utils/output-truncation/src/lib.rs:17-21` |
| `UNIFIED_EXEC_OUTPUT_MAX_BYTES` (collection) | 1 MiB head/tail | `codex-rs/core/src/unified_exec/mod.rs:80` |
| exec `max_output_tokens` default | 10_000 | `codex-rs/core/src/unified_exec/mod.rs:79` |
| bytes per token | 4 | `codex-rs/utils/string/src/truncate.rs:4` |
| persisted MCP / command output | 64 KiB | `codex-rs/rollout/src/policy.rs:16-17` |

## Evolution
- 2025-08-12 `90d892f4fd` prompt states "10 kilobytes or 256 lines"; removed 2025-12-12 `570eb5fe78` → [[prompt-states-stale-harness-limits]].
- 2025-10-04 `d7acd146fb` "fix: exec commands that blows up context window. (#4706)" — some exec paths skipped truncation ("with 76% context window left" users got `input exceeded context window`) → [[tool-output-bypasses-truncation]].
- 2025-11-13 `9890ceb939` "Avoid double truncation" (history gets 10 % headroom over the tool constant — today the ×1.2 allowance); 2025-11-19 `d62cab9a06` "don't truncate at new lines".
- 2025-11-18 `3de8790714` "Add the utility to truncate by tokens": `TruncationPolicy` per model family.
- 2025-12-23 `fb24c47bea` "limit output size for exec command in unified exec" (issues #8197/#8358/#7585: monorepo sessions crashing) → [[bash-output-integrity]].
- 2026-04-30 `3516cb9751` truncate large MCP outputs in rollouts.
- 2026-05-18 `82061660ae` legacy JSON-structured shell output removed ("already plain text for model consumption").
- 2026-07-10 `6138909d6e` "Keep unified exec output collection bounded" (repeated drains accumulated into an uncapped buffer).
- 2026-09-09 `aa88a0333c` "Preserve tool output truncation budgets across resume and fork" (`history_truncation_token_limit`).
- 2026-10-02 `820f85cf59` / `bee28e8a06` 64 KiB previews in paginated history.

## Versus pi
- [[pi--tool-output-truncation]]: per-tool direction (head/tail/middle), dual line+byte limits, actionable continuation notices, spill file. codex: one per-model token budget, middle elision everywhere, enforced once at history recording, full output only in the rollout (no model-readable spill).
