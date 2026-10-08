---
type: implementation
harness: pi
concept: extension-event-hooks
commit: b30a6dd77
files: [packages/coding-agent/src/core/extensions/types.ts:1562, packages/coding-agent/src/core/extensions/types.ts:1567-1634, packages/coding-agent/src/core/extensions/types.ts:1375, packages/coding-agent/src/core/extensions/runner.ts:269-271, packages/coding-agent/src/core/extensions/runner.ts:1242-1260, packages/coding-agent/src/core/extensions/runner.ts:1262-1296, packages/coding-agent/src/core/extensions/loader.ts:271-286, packages/coding-agent/src/core/agent-session.ts:1138-1139, packages/coding-agent/docs/extensions.md:97]
---
[[extension-event-hooks]] in [[pi]].

## Mechanism
- Extension = TS/JS module, default export `ExtensionFactory = (pi: ExtensionAPI) => void | Promise<void>` (`packages/coding-agent/src/core/extensions/types.ts:2015`); async factories awaited before startup continues (`packages/coding-agent/docs/extensions.md:56`).
- Factory must not start processes/sockets/timers ("some invocations load extensions without starting a session"); start in `session_start`, clean up in idempotent `session_shutdown` (`docs/extensions.md:58-60`).
- `ExtensionAPI` interface `types.ts:1562`; `pi.on(event, handler)` → 41 typed overloads (`types.ts:1567-1634`; union `ExtensionEvent` `types.ts:1375`), returns an unsubscribe fn (impl `loader.ts:271-286`, added `46c9de402`).
- Handler signature `(event, ctx: ExtensionContext) => R | void | Promise<R|void>` (`types.ts:1557`).
- Dispatch: extension load order then registration order (`docs/extensions.md:97`); each emit snapshots the handler list so unsubscribes don't disturb an in-flight dispatch (`runner.ts:269-271`). Errors caught per handler → `emitError` → RPC `extension_error` event (`docs/json.md:190-193`) — except fail-closed events below.
- Plugins see agent events BEFORE public SDK listeners: "Emit to extensions first, then notify public listeners" (`agent-session.ts:1138-1139`). Extension emits are awaited; public `_emit` is synchronous (`agent-session.ts:1039-1043`).
- All handlers fully sequential and awaited — a slow `provider_stream_event` / `message_update` handler delays the stream (`docs/extensions.md:109-111`).
- No timeouts on handlers (removed `88e39471e`); stuck hooks only escapable by user abort.
- Fail-closed: `tool_call` has no per-handler try/catch — a throw blocks the call (`runner.ts:1242-1260`; `agent-session.ts:676-680`; `packages/agent/src/agent-loop.ts:770-775`; `docs/extensions.md:260`); `user_bash` error/invalid result aborts rather than falling back to local shell (`runner.ts:1281-1291`, `509ee2bd0`).
- `context` sees the conversation WITHOUT system messages; pi re-attaches system prompt + tool declarations after all handlers (`runner.ts:285-293,1298-1327`); `context_with_system` runs after, sees everything, and dropping the leading system message is reported as an error (`runner.ts:1329-1356`, `aef5fc429`).
- Inter-extension pub/sub: `pi.events: EventBus` on Node EventEmitter, handler errors swallowed to console (`types.ts:1887`; `core/event-bus.ts:12-33`); listeners tracked and unsubscribed on invalidation (`loader.ts:162,196-210`, `6ca423447`).

### Full event table (41 `pi.on` overloads)
| # | Event | Fires when | Can mutate / block | Evidence |
|---|---|---|---|---|
| 1 | `project_trust` | startup / cwd switch, before project-local resources load; only global + CLI extensions see it | return `{trusted:"yes"/"no"/"undecided", remember?}`; first non-undecided wins | `types.ts:686-709`, `runner.ts:295-322`, `core/project-trust.ts:54-70` |
| 2 | `resources_discover` | after `session_start` (startup/reload) | return extra `skillPaths/promptPaths/themePaths` (accumulated) | `types.ts:711-722`, `runner.ts:1474-1518` |
| 3 | `mcp_servers_change` | extension registers/unregisters MCP server after bind | notify | `types.ts:730-734`, `runner.ts:459` |
| 4 | `session_start` | startup/reload/new/resume/fork (`reason`, `previousSessionFile`) | notify | `types.ts:741-747`, `agent-session-runtime.ts:218,251,305` |
| 5 | `session_info_changed` | session name changed | notify | `types.ts:750-754`, `agent-session.ts:3943` |
| 6 | `session_before_switch` | before /new or /resume | `{cancel}` short-circuits | `types.ts:757-761,1472`, `runner.ts:1095-1101` |
| 7 | `session_before_fork` | before fork | `{cancel, skipConversationRestore}` | `types.ts:764-768,1476-1479` |
| 8 | `session_before_compact` | before compaction (`reason: manual/threshold/overflow`, `willRetry`, `signal`) | `{cancel}` or custom `compaction: CompactionResult` | `types.ts:771-782,1481-1484`, `agent-session.ts:2793` |
| 9 | `session_compact` | after successful compaction (`fromExtension`) | notify | `types.ts:784-793` |
| 10 | `session_compact_failed` | compaction failed/aborted | notify | `types.ts:795-808`, `a6b1dbceb` |
| 11 | `session_shutdown` | quit/reload/new/resume/fork; SIGTERM/SIGHUP | notify (cleanup awaited) | `types.ts:810-815`, `agent-session.ts:3662` |
| 12 | `session_before_tree` | before /tree navigation | `{cancel, summary, customInstructions, replaceInstructions, label}` | `types.ts:833-837,1486-1500` |
| 13 | `session_tree` | after tree navigation | notify | `types.ts:840-846` |
| 14 | `context` | before every LLM call; conversation without system msgs | return/mutate `messages`; pi re-attaches prompt+tools | `types.ts:871-874`, `runner.ts:1298-1327,285-293` |
| 15 | `context_with_system` | after all `context` handlers; full transcript | returned verbatim; dropped leading system msg = error | `types.ts:881-884`, `runner.ts:1329-1356` |
| 16 | `cache_warming_decision` | idle prompt-cache refresh decision | `{action:"warm"/"stop"}`; last wins | `core/cache-warmer.ts:112-120`, `runner.ts:1121-1141` |
| 17 | `before_provider_request` | provider payload built | return value REPLACES payload (chained) | `types.ts:887-890`, `runner.ts:1361-1390`, `core/sdk.ts:382-386,432` |
| 18 | `before_provider_headers` | headers assembled pre-HTTP | mutate `headers` in place; `null` deletes; return ignored | `types.ts:897-900`, `runner.ts:1392-1418`, `244f1deaf` |
| 19 | `after_provider_response` | HTTP status+headers received, before stream consumed | notify | `types.ts:903-907`, `core/sdk.ts:387-395`, `d131fcd4b` |
| 20 | `provider_stream_event` | each raw provider stream event pre-normalization | notify; awaited in stream order | `types.ts:910-916`, `docs/extensions.md:109-111`, `002fc8385` |
| 21 | `before_agent_start` | prompt submitted, before loop | chain `systemPromptOptions`; return `systemPrompt` (full override) and/or inject `message` | `types.ts:919-929,1466-1470`, `runner.ts:1420-1472` |
| 22 | `agent_start` | loop starts | notify | `types.ts:932`, `agent-session.ts:1284-1286` |
| 23 | `agent_end` | one low-level run ends (retry/compaction may follow) | notify | `types.ts:937-940` |
| 24 | `agent_before_settle` | final actionable boundary | chain `entries` (custom/custom_message/context_edit/compaction drafts) + `continue:true` for ONE more request | `types.ts:944-1003`, `runner.ts:1029-1078` |
| 25 | `agent_settled` | no retry/compaction/queued continuation left | notify | `types.ts:1005-1007`, `agent-session.ts:1083`, `e9fa5a68a` |
| 26 | `ui_prompt_start` | pi starts blocking on an extension dialog | notify | `types.ts:1012-1018`, `runner.ts:582-609`, `ccfe79ed2` |
| 27 | `ui_prompt_end` | dialog resolved | notify | `types.ts:1020-1026` |
| 28 | `turn_start` | each turn (`turnIndex`, `timestamp`) | notify | `types.ts:1028-1032` |
| 29 | `turn_end` | each turn end with boundary state | same boundary semantics as #24 | `types.ts:1035-1042,1417` |
| 30 | `message_start` | user/assistant/toolResult message starts | notify | `types.ts:1045-1048` |
| 31 | `message_update` | assistant streaming delta | notify | `types.ts:1051-1055` |
| 32 | `message_end` | message finalized | return replacement `message` (same role enforced, chained) | `types.ts:1058-1061,1461-1464`, `runner.ts:1144-1181`, `40c6eabb8` |
| 33 | `tool_execution_start` | tool begins (incl. nested, `parentToolCallId`) | notify | `types.ts:1064-1071` |
| 34 | `tool_execution_update` | partial result | notify | `types.ts:1074-1082` |
| 35 | `tool_execution_end` | tool done (`durationMs`) | notify | `types.ts:1085-1097` |
| 36 | `model_select` | model set/cycle/restore | notify | `types.ts:1104-1109`, `agent-session.ts:2463` |
| 37 | `thinking_level_select` | thinking level changed | notify | `types.ts:1112-1116` |
| 38 | `tool_call` | after arg validation, before execute | mutate `event.input` in place (NOT re-validated); `{block, reason, terminate}`; first block short-circuits; throw = block | `types.ts:1220-1235,1424-1434`, `runner.ts:1242-1260`, `agent-session.ts:658-682`, `packages/agent/src/agent-loop.ts:728-775` |
| 39 | `tool_result` | after execute (also errors) | patch `content/details/structuredContent/isError/usage`, chained; replacing content drops structuredContent unless also returned | `types.ts:1237-1310,1453-1459`, `runner.ts:1183-1240` |
| 40 | `user_bash` | user `!cmd` / `!!cmd` (excludeFromContext) | `{operations}` (custom executor) or `{result}` (full replacement); `undefined` falls through; error/invalid → ABORT | `types.ts:1123-1132,1436-1449`, `runner.ts:1262-1296`, `509ee2bd0` |
| 41 | `input` | raw user input (source interactive/rpc/extension, `streamingBehavior`) | `{action:"continue"}` / `{action:"transform", text, images}` (chained) / `{action:"handled"}` short-circuits | `types.ts:1141-1157`, `runner.ts:1520-1558`, `agent-session.ts:1915-1934` |

Sub-groupings (registry merges): provider-payload hooks = #17–#20 (below [[unified-provider-api]]); compaction hook = #8 (see [[auto-compaction]]); context hooks = #14–#15 ([[context-transform-hook]]); tool gate = #38 ([[tool-call-gate]]); result rewriting = #39 ([[tool-result-rewriting]]); system prompt hook = #21 ([[system-prompt-override]]); settle boundary = #24/#25/#29 ([[run-settlement]]).

### Example extensions (`packages/coding-agent/examples/extensions/`, 79 entries) → hooks used
| Example | What | Hooks / API |
|---|---|---|
| `hello.ts` | minimal tool | `registerTool` |
| `permission-gate.ts` | confirm `rm -rf`/`sudo`/`chmod 777`; block w/o UI | `tool_call`, `ctx.ui.confirm` |
| `protected-paths.ts` | block write/edit to `.env`, `.git/`, `node_modules/` | `tool_call` |
| `confirm-destructive.ts` | confirm clear/switch/fork | `session_before_switch`, `session_before_fork` |
| `dirty-repo-guard.ts` | block session changes with uncommitted changes | `session_before_switch`, `session_before_fork` |
| `git-checkpoint.ts` | `git stash create` per turn; restore on /fork | `turn_start`, `tool_result`, `agent_settled`, `session_before_fork` |
| `auto-commit-on-exit.ts` | commit on shutdown with last reply as message | `session_shutdown` |
| `git-merge-and-resolve.ts` † | fetch+merge upstream after each run | `agent_end` |
| `custom-compaction.ts` | whole-conversation summary compaction | `session_before_compact` |
| `trigger-compact.ts` | compact >100k tokens + `/trigger-compact` | `turn_end`, `ctx.compact()` |
| `handoff.ts` | `/handoff <goal>` new focused session ("compacting is lossy") | `registerCommand`, `newSession` |
| `pirate.ts` / `prompt-customizer.ts` † | system prompt mods | `before_agent_start` (+ `registerCommand` in pirate) |
| `claude-rules.ts` | list `.claude/rules/` for on-demand reading | `session_start` + `before_agent_start` |
| `inline-bash.ts` | expand `!{cmd}` in prompts | `input` |
| `input-transform.ts` † / `input-transform-streaming.ts` | transform input; skip work for mid-stream steering | `input` (`streamingBehavior`) |
| `interactive-shell.ts` | vim/htop with TUI suspended | `user_bash` |
| `bash-spawn-hook.ts` † | rewrite bash command/cwd/env before spawn | `registerTool(createBashTool({spawnHook}))` (override built-in) |
| `ssh.ts` | delegate all tools to remote host | `registerTool(createBashTool…)` overrides + `user_bash` + `before_agent_start` + `registerFlag` |
| `gondolin/` | route built-in tools + `!` into micro-VM | tool overrides + `user_bash` + `before_agent_start` + `session_shutdown` |
| `sandbox/` | OS sandbox via `@anthropic-ai/sandbox-runtime` | bash tool override + `user_bash` + `registerFlag` |
| `provider-payload.ts` † | log/replace provider payload | `before_provider_request`, `after_provider_response` |
| `debug-provider.ts` | raw stream capture into TUI-only entries | `provider_stream_event`, `message_end`, `turn_*`, `appendEntry` |
| `project-trust.ts` | custom trust decisions | `project_trust`, `session_start` |
| `dynamic-resources/` | supply skills/prompts/themes | `resources_discover` |
| `file-trigger.ts` | watch file, inject contents | `session_start` + `sendMessage` |
| `event-bus.ts` | inter-extension pub/sub | `pi.events`, `session_start`, `registerCommand` |
| `model-status.ts` | status on model change | `model_select` |
| `notify.ts` | OSC 777 notification when done | `agent_settled` |
| `status-line.ts` / `titlebar-spinner.ts` / `working-indicator.ts` | progress UI | `turn_start/turn_end`; `agent_start/agent_settled`; `session_start` |
| `plan-mode/` | read-only plan mode, bash allowlist, `[DONE:n]` | `tool_call`, `context`, `before_agent_start`, `turn_end`, `agent_end`, `setActiveTools`, `appendEntry`, `registerFlag` |
| `subagent/` | single/parallel/chain sub-agents via `pi` subprocess | `registerTool` |
| `todo.ts` | todo tool, branch-safe state in tool details | `registerTool`, `session_start` + `session_tree` reconstruct |
| `jev-router.ts` | virtual model plan→implement routing | `registerVirtualModel` |
| `preset.ts` | named presets `--preset`/`/preset` | `registerFlag`, `setModel`, `setActiveTools`, `before_agent_start`, `appendEntry` |
| `custom-provider-anthropic/`, `custom-provider-gitlab-duo/` | custom providers w/ OAuth | `registerProvider` |
| `reload-runtime.ts` | `/reload-runtime` + reload tool | `ctx.reload()`, `registerCommand`, `registerTool`, `sendUserMessage` |
| `shutdown-command.ts` | `/quit` | `ctx.shutdown()` |
| `send-user-message.ts` | inject user message | `sendUserMessage` |
| UI-only (`custom-footer`, `custom-header`, `modal-editor`, `rainbow-editor`, `widget-placement`, `overlay-*`, `doom-overlay/`, `snake`, `space-invaders` †, `tic-tac-toe`, `mac-system-theme`, `hidden-thinking-label`, `minimal-mode`, `built-in-tool-renderer`, `message-renderer`, `entry-renderer`, `border-status-editor` †, `github-issue-autocomplete`, `qna`, `question`, `questionnaire`, `timed-confirm`, `bookmark`, `session-name`, `commands` †, `tools`, `system-prompt-header` †, `summarize`, `structured-output`, `truncated-tool`, `tool-override`, `dynamic-tools`, `with-deps/`, `working-message-test` †, `rpc-demo`) | see [[pi--extension-ui-primitives]] / [[pi--plugin-tools]] | — |
Hook columns verified by grepping `pi.on("…"` in each example at HEAD. Source: `packages/coding-agent/examples/extensions/` catalogue; † = not in `examples/extensions/README.md`. Absent-feature → example map: sub-agents→`subagent/`; plan mode→`plan-mode/`; approvals→`permission-gate`/`protected-paths`/`confirm-destructive`/`dirty-repo-guard`; sandbox→`sandbox/`/`gondolin/`/`ssh`; todos→`todo`; checkpoints→`git-checkpoint`/`auto-commit-on-exit`; routing→`jev-router`/`preset`; ask-user→`question`/`questionnaire`. No examples for web search/fetch, LSP, RAG, background bash.

## Constants
| name | value | path:line |
|---|---|---|
| hook timeout | none (removed) | `88e39471e` |
| events | 41 overloads | `types.ts:1567-1634` |
| continuation per `agent_before_settle` | exactly one request | `types.ts:1000-1003` |

## Evolution
- 2025-12-09 `04d59f31e` hooks system (loader/runner/types, `HookUIContext` per mode, `hookTimeout`), based on PR #147.
- 2025-12-27 `77fe3f1a1` `context` event (non-destructive per-LLM-call transform); 2025-12-28 `57146de20` `before_agent_start`.
- 2025-12-31 `88e39471e` removed hook timeouts — inconsistently applied, broke legit slow ops (LLM calls, user prompts); use Ctrl+C.
- 2026-01-05 `c6fc08453` merge hooks + custom-tools into unified `extensions/`; `--hook/--tool` → `-e`; `hookMessage`→`custom` role; session v3 migration (#454).
- 2026-01-07 `cb3ac0ba9` shared `ExtensionRuntime` with throwing stubs instead of per-extension closures.
- 2026-01-15 `3e5d91f28` `input` event (#761).
- 2026-02-06 `2668326e0` `tool_result` patches chain (last-handler-wins lost changes, #1280).
- 2026-02-12 `ff5148e7c` message/tool_execution events forwarded to extensions (#1375).
- 2026-03-03 `7df89066d` `session_directory` event, later removed in `9f9277ccd` (`packages/coding-agent/CHANGELOG.md:2683`).
- 2026-03-14 `63ac2df24` tool interception moved from wrappers into agent-core `beforeToolCall/afterToolCall`; sequential preflight, parallel exec default (#2113).
- 2026-04-03 `9f9277ccd` `session_switch`/`session_fork` folded into `session_start{reason}` (`packages/coding-agent/CHANGELOG.md:2681`). Earlier rename `session_before_branch`→`session_before_fork` (`packages/coding-agent/CHANGELOG.md:4310`).
- 2026-04-16 `d131fcd4b` `after_provider_response` (#3128); 2026-04-22 `4e919868f` chained system prompt in `before_agent_start` (#3539); 2026-04-30 `40c6eabb8` `message_end` replacements.
- 2026-06-08 `718215bd9` `project_trust` decisions (trust gating `89a92207f` 2026-06-05).
- 2026-07-02 `675756154` abort stuck `context` hooks (#6234) → reverted 2026-07-07 `2b00dade7` (reason unverified).
- 2026-07-06 `244f1deaf` `before_provider_headers` (#6350); 2026-07-09 `e9fa5a68a` `agent_settled` (#6363).
- 2026-08-17 `a6b1dbceb` `session_compact_failed` (#8241); 2026-08-27 `ccfe79ed2` `ui_prompt_start/end` (#8355).
- 2026-09-16 `509ee2bd0` `user_bash` fails closed (#9068); 2026-09-17 `46c9de402` `pi.on` returns unsubscribe (#9630).
- 2026-09-21 `aef5fc429` `context` hides system msgs; new `context_with_system` (#9789, #9822).
- 2026-09-23 `002fc8385` `provider_stream_event` (#9901).

## Evidence commits
`04d59f31e` `88e39471e` `c6fc08453` `cb3ac0ba9` `3e5d91f28` `2668326e0` `ff5148e7c` `63ac2df24` `9f9277ccd` `4e919868f` `40c6eabb8` `718215bd9` `675756154` `2b00dade7` `244f1deaf` `e9fa5a68a` `a6b1dbceb` `ccfe79ed2` `509ee2bd0` `46c9de402` `aef5fc429` `002fc8385` `d131fcd4b` `6ca423447`

## Quirks
- `tool_call` input mutation is NOT re-validated against the schema (`types.ts:1220-1235`) — a plugin can hand the tool malformed args.
- `cache_warming_decision` is last-wins while most transforms chain (`runner.ts:1121-1141`).
- Unconditional `continue:true` in `turn_end`/`agent_before_settle` can infinite-loop — documented hazard only (`docs/extensions.md:117`) → [[unbounded-hook-continuation-loop]].
- Lifecycle handlers calling `newSession/fork/switchSession/reload` can deadlock; "only safe in user-initiated commands" (`types.ts:398-439`; `docs/extensions.md:214-215`).
- Does a `tool_call` throw reach the RPC `extension_error` event, given it bypasses `emitError`? (unverified)
- Why the abortable-context-hook fix was reverted (`2b00dade7`) — commit gives no reason (unverified).
- Extensions run in-process with full OS permissions, no sandbox (`docs/extensions.md:5`) → [[no-sandbox]].

## Durable variant (packages/durable)
- pi-durable `Registry` of named `Extension`s contributes tools, prompt sections, hooks (`beforeTool`/`afterTool`, `beforeCompact`, `onYield`), wraps and tasks; reload in place, stored by name (`packages/durable/README.md:126-145,264-274`; `spec.md:3254-3291`).
- `beforeTool` chain can block or rewrite args and args ARE re-validated after hooks (`packages/durable/src/harness/tool.ts:55-90`) — unlike stable `tool_call`.
- `beforeCompact` can decline or supply a summary (`harness/compaction.ts:128-133`).
- Spec footgun: guard extensions omitted from an array selection silently disappear (`spec.md:4701-4703`).

## Failures
[[plugin-hook-wall-clock-timeout]] · [[tool-result-hook-patches-lost]] · [[pre-tool-hook-sees-stale-state]] · [[hook-error-fails-open]] · [[unbounded-hook-continuation-loop]] · [[context-handler-drops-system-state]] · [[side-door-input-bypasses-hooks]] · [[hook-throw-aborts-parallel-batch]]
