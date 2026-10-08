---
type: implementation
harness: codex
concept: extension-event-hooks
commit: 622e9e3696
files: [codex-rs/ext/extension-api/src/contributors.rs:83, codex-rs/ext/items/src/lib.rs:1, codex-rs/hooks/src/engine/dispatcher.rs:115, codex-rs/protocol/src/protocol.rs:1624, codex-rs/hooks/src/engine/discovery.rs:637, codex-rs/hooks/src/engine/command_runner.rs:54, codex-rs/hooks/src/engine/discovery.rs:764]
---
[[extension-event-hooks]] in [[codex]].

Two hook systems: (1) **compile-time extension contributors** (Rust crates under `codex-rs/ext/*`, first-party only) and (2) **external-process hooks** (Claude-Code-compatible command/MCP-tool handlers configured by users, projects, plugins, admins). Loop placement of (2) → [[codex--turn-lifecycle-hooks|turn-lifecycle-hooks]].

## Mechanism
### Extension contributors (in-process, compiled)
- `ExtensionRegistry` with contributor traits (`codex-rs/ext/extension-api/src/contributors.rs:83-430`): thread lifecycle (on_thread_start / ready / resume / idle / stop), turn lifecycle (on_turn_start / item_completed / stop / abort / error), turn input, config changed, token usage, skill invocation, tools, tool dispatch / start / command start / mcp result / finish / timing, approval review (`ApprovalReviewContributor`), turn items, MCP servers, context/world-state sections (`WorldStateSectionContribution`), `SessionIsolation`.
- Consumers: goal, skills, agent message board, git attribution, memories, web search, image generation, guardian, mcp, queue, history-notes, connectors → [[codex--replaceable-builtin-extension|replaceable-builtin-extension]].
- Extension-owned display schema: "Typed display items owned by Codex extensions … sits below `codex-protocol` so core can carry extension items without owning each extension's display schema" (`codex-rs/ext/items/src/lib.rs:1-4`).
- Abort contributors are awaited before `TurnAborted` is emitted (`codex-rs/core/src/tasks/mod.rs:982-1028`, `on_turn_abort` at `:1010`).
- Example `codex-rs/ext/git-attribution`: world-state section `<git_attribution>` telling the model to append `Co-authored-by: Codex <noreply@openai.com>` and PR line "Generated with [Codex](https://openai.com/codex/)." — or explicitly *not* to when disabled; workspace policy fetched with 5 s timeout (500 ms in tests), retry after 30 s (`codex-rs/ext/git-attribution/src/world_state.rs:6-28`; `codex-rs/ext/git-attribution/src/policy.rs:47-50`).

### External hooks (out-of-process)
- Engine literally `ClaudeHooksEngine` (`codex-rs/hooks/src/engine/dispatcher.rs:115`). Events: PreToolUse, PermissionRequest, PostToolUse, PreCompact, PostCompact, SessionStart, SessionEnd, UserPromptSubmit, SubagentStart, SubagentStop, Stop, Interrupt (`codex-rs/protocol/src/protocol.rs:1624-1637`).
- Handler types Command, McpTool, Prompt, Agent — Prompt/Agent rejected at load ("prompt hooks are not supported yet", "agent hooks are not supported yet", `codex-rs/hooks/src/engine/discovery.rs:637-656`) → [[no-prompt-and-agent-hooks]].
- Semantics: JSON on stdin/stdout + exit-code protocol (exit 2 = block with stderr reason); sync handlers for one event run concurrently and results re-ordered by config order (`codex-rs/hooks/src/engine/dispatcher.rs:115-160`); async handlers fire-and-forget through a semaphore of 8 (`codex-rs/hooks/src/engine/command_runner.rs:54`, `:116-140`).
- Timeouts: default 600 s per command hook (`codex-rs/hooks/src/engine/discovery.rs:764`); SessionEnd default 1 s, max 3 s (`codex-rs/hooks/src/events/session_end.rs:20-23`); stdin written concurrently with draining stdout/stderr, timeout covers both (`885113aa1d`) → [[plugin-hook-wall-clock-timeout]].
- Sources & policy: plugin bundles carry `paths.hooks` ([[codex--harness-package-distribution|harness-package-distribution]]); project hooks gated by trust (`codex-rs/config/src/loader/mod.rs:1093-1105`); admins can force `allow_managed_hooks_only` (requirements only). TUI `/hooks`.
- Stop-hook continuations uncapped (relies on `stop_hook_active`) → [[unbounded-hook-continuation-loop]].

## Constants
| name | value | path:line |
|---|---|---|
| default command-hook timeout | 600 s | codex-rs/hooks/src/engine/discovery.rs:764 |
| SessionEnd hook timeout | 1 s default, 3 s max | codex-rs/hooks/src/events/session_end.rs:20-23 |
| `MAX_CONCURRENT_ASYNC_HOOKS` | 8 | codex-rs/hooks/src/engine/command_runner.rs:54 |
| git-attribution policy fetch | 5 s, retry after 30 s | codex-rs/ext/git-attribution/src/policy.rs:47-50 |

## Evolution
- 2026-02-05 `3b54fd7336` hooks implementation; 2026-02-10 `d735df1f50` hooks crate; 2026-03-09 `244b2d53f4` hooks engine; PreToolUse `73bbb07ba8` 2026-03-23, PostToolUse `c4d9887f9a` 2026-03-25; MCP-tool handlers `85fc4def35` 2026-08-15; interrupted-turn hooks `cbfd999db7` 2026-08-25, `c2abf869d5` 2026-08-28.
- 2026-05-11 `d2c3ebac1f` typed extension API (`codex-rs/ext/*`).
- 2026-09-09 `885113aa1d` (#44288) command hooks no longer hang on blocked stdin.

## Versus pi
- [[pi--extension-event-hooks]]: pi exposes 41 typed in-process `pi.on` events to *third-party* TS plugins (transform/veto, chained); codex's in-process contributor API is first-party only (compiled), and third parties hook in through external processes with a Claude-Code-compatible protocol. pi removed hook timeouts; codex keeps a 600 s default and fixed the stdin deadlock instead. See [[extensibility-model]].
