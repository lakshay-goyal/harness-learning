---
type: implementation
harness: pi
concept: tool-call-gate
commit: b30a6dd77
files: [packages/agent/src/agent-loop.ts:708, packages/agent/src/types.ts:319, packages/coding-agent/src/core/agent-session.ts:652, packages/coding-agent/src/core/extensions/runner.ts:1242, packages/coding-agent/src/core/extensions/types.ts:1220, packages/coding-agent/docs/extensions.md:260, packages/coding-agent/examples/extensions/permission-gate.ts:11, packages/coding-agent/examples/extensions/protected-paths.ts:11, packages/durable/src/harness/tool.ts:55]
---
[[tool-call-gate]] in [[pi]].

## Mechanism
- **Core seam (agent-core)**: `beforeToolCall(ctx, signal)` contract (`packages/agent/src/types.ts:319-326`) runs inside `prepareToolCall` after `prepareArguments` shim + schema `validateToolArguments` (`packages/agent/src/agent-loop.ts:726-727`), so the hook sees validated args plus the current context.
  - Returns `{block:true, reason?, terminate?}` → immediate error tool result `reason || "Tool execution was blocked"`, optional `terminate` hint (`agent-loop.ts:745-755`; terminate-on-block `1eb988cfe` #7715). Blocked = ordinary `isError` result → model sees the reason and can adapt ([[tool-error-as-result]]).
  - Signal re-checked right after the hook; if aborted → "Operation aborted" and the hook's decision is discarded (`agent-loop.ts:738-744`; `b94482762`, #4276: aborted runs kept showing confirmations for sibling calls).
  - Any throw in validation/hook → error result with the thrown message (`agent-loop.ts:770-776`) ⇒ **fail-closed**.
  - Parallel mode: preflight (incl. hook) runs **sequentially** for all calls before concurrent execution (`agent-loop.ts:586-660`) → confirmations are asked one at a time, in source order.
  - Hook must honour the signal itself (`packages/agent/src/types.ts:319-326`).
- **Coding-agent binding**: `_installAgentToolHooks` sets `agent.beforeToolCall` once (`packages/coding-agent/src/core/agent-session.ts:652-655`); `_beforeToolCall` fast-paths when no `tool_call` handlers exist (`:663-665`), else `runner.emitToolCall({type:"tool_call", toolName, toolCallId, parentToolCallId?, input})` (`:667-674`); `Error` re-thrown as is, non-Error wrapped `Extension failed, blocking execution: …` (`:676-680`). Doc: "A `tool_call` handler failure blocks the tool as a fail-safe" (`packages/coding-agent/docs/extensions.md:260`).
- **Dispatch** `emitToolCall` (`packages/coding-agent/src/core/extensions/runner.ts:1242-1259`): iterate a snapshot of handlers in extension load order; first result with `block` returns immediately (later handlers skipped); non-blocking results overwrite each other (last wins); **no per-handler try/catch** → one throw skips remaining handlers and propagates as block.
- **Arg rewriting**: `event.input` is mutable in place; "Later `tool_call` handlers see earlier mutations. No re-validation is performed after mutation." (`packages/coding-agent/src/core/extensions/types.ts:1220-1225`). Typed per built-in tool (`BashToolCallEvent` … `CustomToolCallEvent`, `:1226-1235`).
- **State barrier**: `Agent.processEvents` awaits listeners, so assistant `message_end` (incl. JSONL persistence) completes before preflight; hook sees session state containing the assistant message (`packages/agent/src/agent.ts:565-612`; `63ac2df24`, #2113).
- **Nested calls gated**: tools calling tools via `ctx.executeTool()` (codemode) run the same hooks with `parentToolCallId` and ids `<parent>/<n>` (`agent-session.ts:730-764`; `packages/coding-agent/docs/extensions.md:148`) → a permission extension also covers code-mode-issued calls. See [[nested-tool-calls]], [[code-mode]]. MCP tools too: "Every MCP call passes through Pi's tool pipeline … permission gates, therefore apply to MCP tools" (`packages/coding-agent/docs/mcp.md:252`).
- **User shell path** (`!cmd`/`!!cmd`, not a model tool): separate `user_bash` event; handler returns `{operations}` (custom executor) or `{result}`; invalid result or throw → `emitError` + rethrow → command aborted, never falls back to local shell (`runner.ts:1261-1290`; `509ee2bd0`, #9662 fixing #9068).
- **Other input paths funneled**: RPC `steer`/`follow_up` run `input` handlers (`faa9863cb`, #8718); RPC bash runs `user_bash` (`5d548ae96`, #7214).
- **No policy in core**: "Pi does not include a built-in permission system" (`README.md:90`); "does not ask for approval before every tool call" (`packages/coding-agent/docs/security.md:3`). `ctx.hasUI` true in TUI and RPC (`packages/coding-agent/src/core/extensions/types.ts:330`), so RPC hosts receive confirmations via the extension-UI subprotocol.
- **Example policies** (`packages/coding-agent/examples/extensions/`):
  - `permission-gate.ts`: regex list `rm -r[f]/--recursive`, `sudo`, `chmod|chown …777` (`:11`), only `bash` (`:14`); no UI ⇒ block "Dangerous command blocked (no UI for confirmation)" (`:20-22`); else `ui.select` Yes/No (`:25-29`).
  - `protected-paths.ts`: substring list `.env`, `.git/`, `node_modules/` (`:11`), only `write`/`edit` (`:14`) — bash can still write them (inferred; tool-name gating only).
  - `confirm-destructive.ts` (session actions via `before_*`), `dirty-repo-guard.ts`, `plan-mode/` (bash allowlist enforced by `tool_call` block, `plan-mode/index.ts:168`; `DESTRUCTIVE_PATTERNS`/`SAFE_PATTERNS` `plan-mode/utils.ts:7,44,97-99`).
  - `sandbox/index.ts:8-10` notes bash could be sandboxed "via `tool_call` input mutation without replacing the tool".
  - Docs snippet using [[tool-safety-annotations]] to confirm what Codex would ask about (`packages/coding-agent/docs/extensions.md:166-177`).
- No hook timeout: removed (`88e39471e`) because timeouts broke legitimate LLM calls / dialogs → a gate may wait on a human indefinitely, bounded only by abort.

## Constants
| name | value | path:line |
|---|---|---|
| default block reason | `"Tool execution was blocked"` | packages/agent/src/agent-loop.ts:746 |
| non-Error wrap | `Extension failed, blocking execution: …` | packages/coding-agent/src/core/agent-session.ts:679 |
| aborted-during-hook result | `"Operation aborted"` | packages/agent/src/agent-loop.ts:741 |
| hook timeout | none (removed) | `88e39471e` |

## Evolution
- 2025-11-12 `b172beb92` README "Security (YOLO by default)": no gate; "Use at your own risk".
- 2025-12-09 `04d59f31e` hooks system (with `hookTimeout`). 2025-12-17 `3424550d2` "No permission popups. Security theater." 2025-12-31 `88e39471e` hook timeouts removed.
- 2026-01-03/04 `57bba4e32`/`059292ead` → `91fae8b2f`: hook API for dynamic tool control, driven by plan-mode example.
- 2026-01-05 `c6fc08453` hooks + custom tools merged into unified extensions (#454).
- 2026-02-06 `2668326e0` `tool_result` patches chain instead of last-wins (#1280) (post-call counterpart).
- 2026-03-14 `63ac2df24` interception moved from tool wrappers into agent-core `beforeToolCall/afterToolCall`; sequential preflight (#2113).
- 2026-04-17 `e9808b585` restore `afterToolCall` error overrides (#3051).
- 2026-05-19 `b94482762` stop preflight after extension abort (#4276).
- 2026-07-28 `5d548ae96` RPC bash no longer bypasses `user_bash` (#7214).
- 2026-08-06 `1eb988cfe` blocked calls may terminate the batch (#7715).
- 2026-09-08 `faa9863cb` input handlers for queued (steer/follow-up) messages (#8718).
- 2026-09-10 `c7eee0195` agent-runtime design doc: "permissions and approval policy are plugin territory (`before_tool` can block or rewrite args and may wait for a person)" (doc later deleted).
- 2026-09-16 `509ee2bd0` `user_bash` fails closed (#9662).
- 2026-09-29 `8562bcf66` tool `annotations` + `ctx.executeTool` nested calls through the gate.

## Evidence commits
`b172beb92`, `3424550d2`, `88e39471e`, `c6fc08453`, `63ac2df24`, `b94482762`, `1eb988cfe`, `5d548ae96`, `faa9863cb`, `509ee2bd0`, `8562bcf66`, `c7eee0195`

## Quirks
- Mutated args are not re-validated: a policy that rewrites `command` can produce schema-invalid input that reaches `execute` (`packages/coding-agent/src/core/extensions/types.ts:1222-1224`).
- Non-blocking handler results are last-wins (`runner.ts:1250-1254`) while `tool_result` patches chain — asymmetric.
- Whether a throwing `tool_call` handler surfaces as an `extension_error` RPC event (it bypasses `emitError`) — unverified.
- Gate keys on `toolName`; a renamed/overriding tool (`tool-override.ts`) or MCP tool slips past name-based example policies.

## Durable variant (packages/durable)
- `pi.tool` task `call` phase: resolve tool, validate, run `beforeTool` chain (block or rewrite args), **re-validate**, then commit intent `{phase:"execute", arguments, replay}` before executing (`packages/durable/src/harness/tool.ts:55-90`). Gate decision is therefore durable: a crash after the commit never re-asks.

## Failures
- [[hook-error-fails-open]] · [[side-door-input-bypasses-hooks]] · [[pre-tool-hook-sees-stale-state]]
