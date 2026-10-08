---
type: implementation
harness: codex
concept: client-server-session-split
commit: 622e9e3696
files: [codex-rs/app-server-protocol/src/rpc.rs:1, codex-rs/app-server/src/main.rs:30, codex-rs/app-server-transport/src/transport/mod.rs:24, codex-rs/app-server-transport/src/transport/mod.rs:120, codex-rs/app-server-protocol/src/protocol/common.rs:499, codex-rs/app-server-protocol/src/protocol/common.rs:1780, codex-rs/app-server-protocol/src/protocol/common.rs:1935, codex-rs/app-server-protocol/src/protocol/v1.rs:43, codex-rs/app-server-client/src/lib.rs:1, codex-rs/app-server-client/src/lib.rs:86, codex-rs/app-server-daemon/src/lib.rs:49, codex-rs/cli/src/main.rs:219]
---
[[client-server-session-split]] in [[codex]].

One protocol ("app-server") between the agent runtime (`ThreadManager` + threads) and **every** first-party surface: VS Code extension, desktop app, TUI, `codex exec`, Python SDK. Runs in-process (TUI/exec), as a stdio child (IDE, Python SDK), or as a long-lived daemon.

## Mechanism
### Wire design (JSON-RPC-ish)
- Inherited from MCP, not chosen fresh: app-server split out of `codex mcp` on 2025-09-30 `d9dbf48828` — `codex mcp` "started a JSON-RPC-ish server that had two overlapping responsibilities … Running the app server used to power experiences such as the VS Code extension".
- "We do not do true JSON-RPC 2.0, as we neither send nor expect the `"jsonrpc": "2.0"` field" (`codex-rs/app-server-protocol/src/rpc.rs:1-2`) → [[no-strict-jsonrpc]]. Message = Request | Notification | Response | Error; ids string or integer.
- **Transports**: `--listen stdio://` (default, `DEFAULT_LISTEN_URL`, `codex-rs/app-server-transport/src/transport/mod.rs:120`), `unix://[PATH]`, `ws://IP:PORT`, `off`; remote-control (pairing/relay) transport (`codex-rs/app-server-transport/src/transport/remote_control`); default `--session-source vscode`; `--strict-config` fails on unknown config keys (`codex-rs/app-server/src/main.rs:30-75`). Incoming app-server/exec-server websockets authenticated by `codex-rs/websocket-auth`. Outgoing `CHANNEL_CAPACITY = 128` (`codex-rs/app-server-transport/src/transport/mod.rs:24`).
- **Capability negotiation in `initialize`**: `experimentalApi` opt-in gates experimental methods/fields (derive macro `ExperimentalApi`); `optOutNotificationMethods` suppresses named notifications; MCP extension negotiation; attestation opt-in (`codex-rs/app-server-protocol/src/protocol/v1.rs:43-70`; `codex-rs/app-server-protocol/src/experimental_api.rs`).
- **Schema**: TS + JSON Schema generated from Rust types (`ts-rs`, `schemars`), precomputed since `acd540f158` 2026-07-30; v1/v2 split 2025-10-30 `cdc3df3790`. ~11.7k lines of TS in repo are mostly generated protocol bindings (M8 repo facts).

### API surface (v2, `codex-rs/app-server-protocol/src/protocol/common.rs:499-1510`)
- 173 client→server methods (full list below): `thread/*` (start, resume, fork, archive, revert, compact/start, goal/*, queue/*, realtime/*, list/search/read, turns/list, items/list, inject_items), `turn/*` (start, steer, interrupt, settings/update), `review/start`, `model/list`, `plugin/*`, `skills/*`, `marketplace/*`, `app/*`, `fs/*`, `command/exec*`, `process/*`, `config/*` (read, value/write, batchWrite, configRequirements/read), `account/*`, `mcpServer/*`, `feedback/upload`, `collaborationMode/list`, `experimentalFeature/*`, `remoteControl/*`, `externalAgentConfig/*`.
- **Server→client requests = approval/UI channel** (`codex-rs/app-server-protocol/src/protocol/common.rs:1780-1851`): `item/commandExecution/requestApproval`, `item/fileChange/requestApproval`, `item/permissions/requestApproval`, `item/tool/requestUserInput`, `mcpServer/elicitation/request`, `item/tool/call` (client-hosted dynamic tools → [[client-supplied-dynamic-tools]]), `account/chatgptAuthTokens/refresh`, `attestation/generate`, `currentTime/read`. Command approval decisions: Accept, AcceptForSession, AcceptWithExecpolicyAmendment, ApplyNetworkPolicyAmendment, Decline, Cancel; unexpected MCP amendment fails closed to Decline (`codex-rs/app-server-protocol/src/protocol/v2/item.rs:66-95`). `serverRequest/resolved` closes the loop across multiple attached clients.
- **Thread/turn/item model**: ThreadStatus = NotLoaded | Idle | SystemError | Active{activeFlags: WaitingOnApproval|WaitingOnUserInput} (`codex-rs/app-server-protocol/src/protocol/v2/thread.rs:1670-1687`); TurnStatus = Completed | Interrupted | Failed | InProgress (`codex-rs/app-server-protocol/src/protocol/v2/turn.rs:33-38`); ThreadItem variants UserMessage, HookPrompt, AgentMessage, FunctionCallOutput, Plan, Reasoning, CommandExecution, FileChange, McpToolCall, DynamicToolCall, CollabAgentToolCall, SubAgentActivity, WebSearch, ImageView, Sleep, ImageGeneration, EnteredReviewMode, ExitedReviewMode, ContextCompaction (`codex-rs/app-server-protocol/src/protocol/v2/item.rs`).
- **Notifications**: thread/started → turn/started → item/started → item/*/delta → item/completed → turn/completed, plus thread/status/changed, thread/tokenUsage/updated, turn/diff/updated, turn/plan/updated, model/rerouted (`codex-rs/app-server-protocol/src/protocol/common.rs:1935-2061`) → [[codex--agent-event-stream|agent-event-stream]].

### Method inventory at `622e9e3696` (`codex-rs/app-server-protocol/src/protocol/common.rs`)
Counts: 173 client→server requests (`:499-1510`; incl. `initialize` and 4 "DEPRECATED APIs below" v1 holdovers `getConversationSummary`, `gitDiffToRemote`, `getAuthStatus`, `fuzzyFileSearch`, `:1468`), 9 server→client requests (`:1780-1851`), 85 server notifications (`:1935-2061`), 1 client notification `initialized` (`:2082-2084`).
| family | methods |
|---|---|
| **requests** | |
| `server/` (1) | diagnostics |
| `userVerification/` (5) | status, enroll, delete, verify, cancel |
| `thread/` (50) | start, resume, fork, archive, delete, unsubscribe, increment_elicitation, decrement_elicitation, name/set, prediction/request, goal/set, goal/get, goal/clear, queue/add, queue/list, queue/update, queue/delete, queue/reorder, queue/start, metadata/update, attachment/add, attachment/list, attachmentOwner/list, attachment/remove, section/move, settings/update, memoryMode/set, unarchive, compact/start, shellCommand, approveGuardianDeniedAction, backgroundTerminals/clean, backgroundTerminals/list, backgroundTerminals/terminate, revert, list, search, searchOccurrences, loaded/list, read, turns/list, items/list, inject_items, realtime/start, realtime/appendAudio, realtime/appendText, realtime/appendSpeech, realtime/stop, timeline/list, realtime/listVoices |
| `memory/` (2) | status, reset |
| `rollout/` (1) | compress |
| `project/` (7) | list, read, create, import, update, move, delete |
| `threadSection/` (4) | list, create, update, delete |
| `skills/` (3) | list, extraRoots/set, config/write |
| `hooks/` (1) | list |
| `marketplace/` (3) | add, remove, upgrade |
| `plugin/` (13) | list, search, installed, reconcile, read, skill/read, share/save, share/updateTargets, share/list, share/checkout, share/delete, install, uninstall |
| `app/` (3) | read, list, installed |
| `fs/` (9) | readFile, writeFile, createDirectory, getMetadata, readDirectory, remove, copy, watch, unwatch |
| `turn/` (4) | start, settings/update, steer, interrupt |
| `review/` (1) | start |
| `model/` (1) | list |
| `account/` (15) | gatewayOAuth/read, gatewayOAuth/login, gatewayOAuth/cancel, login/start, bedrock/discover, bedrock/setup, bedrock/checkGovCloudRequirements, login/cancel, logout, rateLimits/read, rateLimitResetCredit/consume, usage/read, workspaceMessages/read, sendAddCreditsNudgeEmail, read |
| `modelProvider/` (1) | capabilities/read |
| `experimentalFeature/` (2) | list, enablement/set |
| `permissionProfile/` (1) | list |
| `remoteControl/` (7) | enable, disable, status/read, pairing/start, pairing/status, client/list, client/revoke |
| `collaborationMode/` (1) | list |
| `mock/` (1) | experimentalMethod |
| `environment/` (3) | add, info, status |
| `mcpServer/` (5) | oauth/login, resource/read, event/stream/start, event/stream/stop, tool/call |
| `config/` (4) | mcpServer/reload, read, value/write, batchWrite |
| `mcpServerStatus/` (1) | list |
| `windowsSandbox/` (2) | setupStart, readiness |
| `feedback/` (1) | upload |
| `command/` (4) | exec, exec/write, exec/terminate, exec/resize |
| `process/` (4) | spawn, writeStdin, kill, resizePty |
| `externalAgentConfig/` (4) | detect, import, import/recordHistory, import/readHistories |
| `configRequirements/` (1) | read |
| `fuzzyFileSearch/` (3) | sessionStart, sessionUpdate, sessionStop |
| **notifications** | |
| bare (5) | error, warning, guardianWarning, deprecationNotice, configWarning |
| `thread/` (30) | started, status/changed, archived, deleted, unarchived, closed, reverted, name/updated, attachment/updated, goal/updated, prediction/updated, goal/cleared, queue/changed, project/updated, environment/connected, environment/disconnected, settings/updated, tokenUsage/updated, compacted, realtime/started, realtime/itemAdded, realtime/item/started, realtime/item/transcript/delta, realtime/item/completed, realtime/transcript/delta, realtime/transcript/done, realtime/outputAudio/delta, realtime/sdp, realtime/error, realtime/closed |
| `skills/` (1) | changed |
| `project/` (1) | changed |
| `turn/` (5) | started, completed, diff/updated, plan/updated, moderationMetadata |
| `hook/` (2) | started, completed |
| `item/` (14) | started, autoApprovalReview/started, autoApprovalReview/completed, completed, agentMessage/delta, plan/delta, commandExecution/outputDelta, commandExecution/terminalInteraction, fileChange/outputDelta, fileChange/patchUpdated, mcpToolCall/progress, reasoning/summaryTextDelta, reasoning/summaryPartAdded, reasoning/textDelta |
| `autoApprovalReview/` (1) | strictReviewRequired |
| `rawResponseItem/` (1) | completed |
| `rawResponse/` (1) | completed |
| `command/` (1) | exec/outputDelta |
| `process/` (2) | outputDelta, exited |
| `serverRequest/` (1) | resolved |
| `mcpServer/` (3) | oauthLogin/completed, startupStatus/updated, event/stream/notification |
| `account/` (3) | updated, gatewayOAuth/changed, rateLimits/updated |
| `app/` (1) | list/updated |
| `remoteControl/` (1) | status/changed |
| `externalAgentConfig/` (2) | import/progress, import/completed |
| `fs/` (1) | changed |
| `model/` (3) | rerouted, verification, safetyBuffering/updated |
| `modelProvider/` (2) | authRecoveryStarted, authRecoveryCompleted |
| `fuzzyFileSearch/` (2) | sessionUpdated, sessionCompleted |
| `windows/` (1) | worldWritableWarning |
| `windowsSandbox/` (1) | setupCompleted |
- Notable: `thread/increment_elicitation` / `decrement_elicitation` let external helpers "pause timeout accounting while a user approval or other elicitation is pending outside the app-server request flow" (`:586-590`); `thread/approveGuardianDeniedAction` (→ [[codex--llm-approval-reviewer|llm-approval-reviewer]]); `rollout/compress`, `memory/reset`, `userVerification/*`, `threadSection/*` + `project/*` (desktop sidebar organisation), `windowsSandbox/setupStart|readiness` + `windows/worldWritableWarning` (→ [[codex--os-level-sandbox|os-level-sandbox]]).

### Hosting modes
- **In-process facade** `codex-rs/app-server-client` (TUI, exec): runtime startup + initialize handshake, typed dispatch, server-request resolution, "Ordered, lossless event consumption that cannot block request processing"; commands bounded, consumer event queue unbounded (`codex-rs/app-server-client/src/lib.rs:1-17`) → [[unbounded-subscriber-buffering]]. `SHUTDOWN_TIMEOUT` 5 s, `IN_PROCESS_SHUTDOWN_TIMEOUT` 45 s "covers the embedded drain, its analytics flush, and final task join" (`codex-rs/app-server-client/src/lib.rs:86-88`).
- **Daemon** `codex-rs/app-server-daemon`: "Managed app-server lifecycle, serialized across CLI invocations and the updater"; pid/lock files (`daemon.pid`, `daemon-updater.pid`, `daemon.lock`, `settings.json`) under state dir `app-server-daemon/`; `START_TIMEOUT` 10 s, poll 50 ms (`codex-rs/app-server-daemon/src/lib.rs:1`, `:49-60`); control-socket response timeout 2 s (`codex-rs/app-server-daemon/src/client.rs:25`); auto-update loop and thread recovery after restart (`codex-rs/app-server-daemon/src/update_loop.rs`, `thread_recovery.rs`) → [[codex--durable-execution|durable-execution]]. CLI `codex agents` browses sessions on the shared daemon; TUI `/daemon`.
- **Stdio child**: IDE extension and Python SDK spawn `codex app-server --listen stdio://` → [[codex--sdk-embedding|sdk-embedding]].
- **Cloud**: native gRPC ThreadService client for cloud threads (resume/attach; never retried) (`codex-rs/cloud-client/src/lib.rs:1-5`, `b707714ae4` 2026-10-01).

### CLI subcommand surface (folded `cli-subcommand-surface`)
Single multi-call binary multiplexing many roles (`codex-rs/cli/src/main.rs` `enum Subcommand`):

| group | subcommands |
|---|---|
| run the agent | (default TUI), `exec`, `review`, `resume`, `fork`, `cloud`, `queue` (queue a message for an existing session) |
| sessions | `agents` (browse sessions on the shared daemon), `archive`, `unarchive`, `delete`, `migrate-rollouts` (`codex-rs/cli/src/main.rs:219`) |
| servers | `app-server`, `exec-server` ([[codex--remote-execution-env|remote-execution-env]]), `mcp-server` (removed `531f3836a1` 2026-09-05), `remote-control`, `app` (Desktop) |
| config / ecosystem | `login`, `logout`, `mcp`, `plugin`, `features` ([[feature-flag-stages]]), `execpolicy`, `sandbox`, `completion`, `update`, `doctor`, `debug` |
| one-shot utilities | `apply` (git-apply last diff) |
| internal | `responses-api-proxy`, `stdio-to-uds`, `tcp-tunnel` |
- Integration tests use an arg0-dispatch guard + temp `CODEX_HOME` to spawn the multi-call binary (`codex-rs/test-binary-support/lib.rs`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_LISTEN_URL` | `stdio://` | codex-rs/app-server-transport/src/transport/mod.rs:120 |
| outgoing `CHANNEL_CAPACITY` | 128 | codex-rs/app-server-transport/src/transport/mod.rs:24 |
| in-process `SHUTDOWN_TIMEOUT` / `IN_PROCESS_SHUTDOWN_TIMEOUT` | 5 s / 45 s | codex-rs/app-server-client/src/lib.rs:86-88 |
| daemon `START_TIMEOUT` / `START_POLL_INTERVAL` | 10 s / 50 ms | codex-rs/app-server-daemon/src/lib.rs:49-50 |
| daemon control-socket response timeout | 2 s | codex-rs/app-server-daemon/src/client.rs:25 |
| client→server requests / server→client requests / notifications | 173 / 9 / 85 | codex-rs/app-server-protocol/src/protocol/common.rs:499-1510, :1780-1851, :1935-2061 |

## Evolution
- 2025-09-30 `d9dbf48828` `codex mcp` → `codex mcp-server` + `codex app-server`.
- 2025-10-30 `cdc3df3790` protocol v1/v2 split.
- 2026-03-08 `da3689f0ef` `codex exec` on an in-process app server.
- 2026-03-13 `9dba7337f2` / 2026-03-16 `db89b73a9c` TUI moved on top of the app server ("in stages … hybrid approach"); legacy TUI split removed `d65deec617` 2026-03-27.
- 2026-05-08 `0c8d42525e` app-server daemon lifecycle.
- 2026-07-21 `ee71c4a90f` config writes overlapping an exact requirement fail with `configRequirementReadonly`.
- 2026-07-30 `acd540f158` precomputed schema export.
- 2026-08-13 `6fc6b9d6d2` unread events no longer block in-process requests → [[unbounded-subscriber-buffering]].
- 2026-09-05 `531f3836a1` `codex mcp-server` removed in favour of app-server.
- 2026-10-01 `b707714ae4` gRPC cloud thread resume/attach.

## Quirks
- One protocol for in-process and out-of-process clients, so the TUI exercises the same API as third-party IDEs.
- Approvals are server-initiated requests, resolved by whichever attached client answers first (`serverRequest/resolved`).
- Interrupting a finished turn used to hang the RPC → [[interrupt-rpc-hangs-on-finished-turn]].

## Versus pi
- [[pi--client-server-session-split]]: pi's split is experimental (`PI_EXPERIMENTAL=1`), CBOR length-prefixed envelopes over Unix sockets with replicated state and per-session worker processes; codex's is the production path for all surfaces, JSON (no `jsonrpc` field) over stdio/unix/ws, threads hosted in one server process, event-stream reducers instead of replicated state.
- pi exact-equality version check vs codex capability negotiation (`experimentalApi`, notification opt-out) + generated schemas.
