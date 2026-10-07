---
type: implementation
harness: pi
concept: headless-rpc-mode
commit: b30a6dd77
files: [packages/coding-agent/src/modes/rpc/rpc-types.ts:22-74, packages/coding-agent/src/modes/rpc/rpc-types.ts:252-297, packages/coding-agent/src/modes/rpc/jsonl.ts:4-58, packages/coding-agent/src/modes/rpc/rpc-mode.ts:138-309, packages/coding-agent/src/modes/rpc/rpc-client.ts, packages/coding-agent/src/core/output-guard.ts:20-70, packages/coding-agent/src/main.ts:112-123, packages/coding-agent/src/main.ts:651-655, packages/coding-agent/src/modes/print-mode.ts:139-148, packages/coding-agent/docs/rpc.md:3-93]
---
[[headless-rpc-mode]] in [[pi]].

## Mechanism
- Mode selection `resolveAppMode` (`packages/coding-agent/src/main.ts:112-123`): `--mode rpc` → rpc; `--mode json` → json; `--print` OR non-TTY stdin OR non-TTY stdout → print; else interactive.
- **stdout takeover**: non-interactive modes call `takeOverStdout()` — every `process.stdout.write` (stray `console.log` from extensions, package managers) goes to stderr; a raw writer is kept for protocol output (`main.ts:651-655`; `core/output-guard.ts:45-70`; `f1fe49a64`, #2482). Raw writes honor backpressure with ENOBUFS/EAGAIN/EWOULDBLOCK retry (`output-guard.ts:20-43`; `d0d1d8edc`). Not a safety feature — protocol integrity only.
- **RPC** (`--mode rpc`): long-lived JSONL over stdin/stdout (`docs/rpc.md:3-33`). `rpc-mode.ts` dispatch handles all 33 command types (33 `case "…"` arms).
  - Strict LF framing, no Node `readline` (splits on U+2028/U+2029) (`modes/rpc/jsonl.ts:4-58`, `serializeJsonLine` `:10`; `e3adaf1bd`, #1911).
  - Optional `id` correlation; malformed JSON → `{command:"parse", success:false}` (`docs/rpc.md:37-85`).
  - `prompt` response = accepted with `disposition: started|queued|handled`, NOT completed; clients wait for `agent_settled` (`docs/rpc.md:60-69`; `core/agent-session.ts:310-311`) → [[run-settlement]].
  - Closing stdin = orderly shutdown (`docs/rpc.md:93`).
  - RPC-only events: `bash_execution_update`, `extension_error` (`docs/json.md:182-190`).
  - TS client `RpcClient` exported (`modes/rpc/rpc-client.ts`); examples `examples/rpc-client.ts` (one prompt via child), `examples/rpc-extension-ui.ts` (TUI chat over RPC); `examples/extensions/rpc-demo.ts` exercises all extension-UI methods.
- **33 commands** (`modes/rpc/rpc-types.ts:22-74`):
  | group | commands |
  |---|---|
  | prompting / queue | `prompt` (+images, `streamingBehavior`), `steer`, `follow_up`, `abort`, `clear_queue` |
  | session | `new_session`, `switch_session`, `fork`, `clone`, `get_fork_messages`, `set_session_name`, `export_html` |
  | state / introspection | `get_state`, `get_session_stats`, `get_entries`, `get_tree`, `get_messages`, `get_last_assistant_text`, `get_commands` |
  | model / thinking | `set_model`, `cycle_model`, `get_available_models`, `set_thinking_level`, `cycle_thinking_level`, `get_available_thinking_levels` |
  | queue modes | `set_steering_mode`, `set_follow_up_mode` |
  | compaction / retry | `compact`, `set_auto_compaction`, `set_auto_retry`, `abort_retry` |
  | user shell | `bash`, `abort_bash` |
- **Extension UI sub-protocol**: stdout `extension_ui_request` methods `select|confirm|input|editor|notify|setStatus|setWidget|setTitle|set_editor_text`; stdin `extension_ui_response {value|confirmed|cancelled}`; agent-side timeout auto-resolves (`rpc-types.ts:252-297`; `docs/rpc-extension-ui.md:10,25`). RPC forwards dialogs/notify/status/string widgets/title/editor text and stubs component factories, `custom()`, theme switching (`modes/rpc/rpc-mode.ts:138-309`) → [[pi--extension-ui-primitives]].
- RPC `steer`/`follow_up` now funnel through `input` hooks (`faa9863cb`, #8718); RPC `bash` through `user_bash` (`5d548ae96`, #7214) → [[side-door-input-bypasses-hooks]].
- **Print** (`-p`): runs prompts, writes final assistant text, exit 1 on error/aborted stopReason (`modes/print-mode.ts:139-148`); piped stdin prepended to first prompt (`docs/cli.md:34-37`).
- **JSON** (`--mode json`): see [[pi--agent-event-stream]].
- Detached bash children killed on SIGHUP/SIGTERM/exit in interactive/print/rpc (`9b7948c4c`) → [[process-tree-kill]].
- **CLI parsing details** (`packages/coding-agent/src/cli/args.ts`): piped stdin is read (except in RPC, where stdin is the protocol) and **downgrades interactive to print** even on a TTY stdout (`main.ts:891-898`); `@path` args → `fileArgs` (`args.ts:247-248`) → [[pi--xml-prompt-boundaries|xml-prompt-boundaries]]; any unrecognized `--name[=value]` is kept in `unknownFlags` (next non-`-`/`@` token taken as its value) for extension-registered flags such as `--plan`, which `--help` lists under "Extension CLI Flags" (`args.ts:249-262,272-283`); unknown short `-x` is an error (`:263-264`). `--offline` = `PI_OFFLINE=1` + `PI_SKIP_VERSION_CHECK=1` before anything else runs (`main.ts:576-580`). Subcommands `install/remove/update [self]/list/config/auth/mcp` dispatch before the agent starts (help `args.ts:289-298`). Full flag table: [[pi#Coverage index]].

## Constants
| name | value | path:line |
|---|---|---|
| RPC commands | 33 | `rpc-types.ts:22-74` |
| record separator | `\n` only | `jsonl.ts:4-58` |

## Evolution
- 2025-12-09 `3559a43ba` RPC mode rewritten with typed protocol + `RpcClient` (headless embedding).
- 2026-03-07 `e3adaf1bd` strict LF JSONL framing (#1911).
- 2026-03-20 `21ef72e9c` protect rpc stdout (#2388); 2026-03-22 `f1fe49a64` stdout takeover in print/json (#2482).
- 2026-05-24 `d0d1d8edc` respect stdout backpressure.
- 2026-07-28 `5d548ae96` RPC `bash` no longer bypasses `user_bash` (#7214).
- 2026-08-03 `a4475344f` linear JSON streaming / delta-only `message_update` on the wire (PR #7394, issue #7290).
- 2026-09-08 `faa9863cb` RPC inputs through extension hooks (#8718).
- Other RPC fixes: 2026-06-18 `51f752358` unknown-command errors lacked request `id` so clients hung (#5868); 2026-09-25 `92e8d4f02` `RpcClient` skipped listeners on unsubscribe (#9990); 2026-05-24 `ce0e801d8` retry stdout backpressure. (#4897 linkage unverified.)

## Evidence commits
`3559a43ba` `e3adaf1bd` `21ef72e9c` `f1fe49a64` `d0d1d8edc` `ce0e801d8` `51f752358` `5d548ae96` `a4475344f` `faa9863cb` `92e8d4f02`

## Quirks
- `hasUI` is true in RPC, so plugins may open dialogs a headless client must answer or time out (`core/extensions/types.ts:330`).
- `prompt` returning on acceptance surprises clients expecting completion.

## Durable variant (packages/durable)
- Experimental replacement transport: length-prefixed CBOR envelopes over Unix sockets / Radius relay with Chord services instead of JSONL verbs → [[pi--client-server-session-split]].

## Failures
[[headless-protocol-stream-corruption]] · [[quadratic-event-stream-output]] · [[side-door-input-bypasses-hooks]]
