---
type: implementation
harness: pi
concept: abort-propagation
commit: b30a6dd77
files: [packages/coding-agent/src/modes/interactive/interactive-mode.ts:4721, packages/coding-agent/src/core/agent-session.ts:2433, packages/agent/src/agent.ts:341, packages/agent/src/agent.ts:507, packages/agent/src/agent-loop.ts:403, packages/agent/src/agent-loop.ts:575, packages/agent/src/agent-loop.ts:613, packages/ai/src/api/anthropic-messages.ts:887, packages/ai/src/utils/abort.ts:13, packages/durable/docs/spec.md:1935]
---
[[abort-propagation]] in [[pi]].

## Mechanism (end to end, stable loop)
1. **UI**: Esc while `session.isStreaming` → `restoreQueuedMessagesToEditor({abort:true})` → `clearAllQueues()` (queued text restored to editor) → `void session.abort()` (`packages/coding-agent/src/modes/interactive/interactive-mode.ts:3050-3052,4721-4740`). Esc during user `!` bash → `abortBash()` only (`:3053-3054`). Extension `ctx.abort()` routes through the same handler (`interactive-mode.ts:1968-1970,2250-2252`; `b94482762` #4276).
2. **`AgentSession.abort()`** (`packages/coding-agent/src/core/agent-session.ts:2433-2443`): set `_agentRunAbortRequested` if run active → `abortRetry()` (cancels backoff sleep) → `abortCompaction()` → `abortBranchSummary()` → flag abort during `agent_before_settle` → `agent.abort()` → `await waitForIdle()` (idle = no run && not compacting, `:1464-1466`). User `!` bash NOT aborted here (`abortBash`, `:3904-3908`; concurrent bash controllers `2efa728d2`); `dispose()` aborts everything incl. bash (`:1392-1413`; `5b31ffd74` #5029).
3. **`Agent.abort()`** → `activeRun.abortController.abort()` (`packages/agent/src/agent.ts:341-343`). **One `AbortController` per run** (`agent.ts:512-517`); `agent.signal` exposed to extensions (`:336-338`; `7d4faa080` #2660) and passed to every awaited listener (`agent.ts:605-611`).
4. **Provider stream**: signal in `streamFn` options (`agent-loop.ts:403-407`). Adapters catch and set `stopReason = signal.aborted ? "aborted" : "error"`, push `error` event with partial output and strip scratch (`partialJson`, `index`) (e.g. `packages/ai/src/api/anthropic-messages.ts:887-897`; abort checked per chunk read `:482-484`). StreamFn contract: must not throw after returning; encode failures in stream (`packages/agent/src/types.ts:27-31`) → [[errors-as-stream-events]]. Proxy: abort listener cancels reader → `reason:"aborted"` (`packages/agent/src/proxy.ts:136-144,195-210,233-243`). Google: `config.abortSignal`, pre-aborted signal throws at build (`google-generative-ai.ts:415-420`). Mistral/Codex SSE parsers cancel reader on abort (`mistral-conversations.ts:452-455`; `openai-codex-responses.ts:805-808`, `a36a132c7`).
5. **Partial message**: loop keeps partial in `context.messages` while streaming (`agent-loop.ts:416-441`); on `error` replaces with `response.result()` (partial content + `stopReason:"aborted"`), emits `message_end` (`:443-456`) → persisted by session (`agent-session.ts:1143-1163`); `pending` partials never persisted (`docs/message-types.md:145`); loop hard-exits (`agent-loop.ts:245-256`) → [[partial-message-persistence]].
6. **Running tools**: same signal into `tool.execute` (`agent-loop.ts:832-835`); bash kills process tree (`core/tools/bash.ts:141-149`) → [[process-tree-kill]]. Sequential: `break` after current call (`agent-loop.ts:575-577`). Parallel: stop preparing further calls (`:613-615`, `641-643`); prepared-but-unstarted thunks return "Operation aborted" without executing (`:619-628`; `afda4d620` #8936); `beforeToolCall` result discarded if signal aborted during hook (`:738-744`, `757-760`; `b94482762`). Calls never prepared get **no** toolResult → pi-ai synthesizes "No result provided" at next request ([[transcript-replay-repair]]).
7. **After tool abort** (inferred): no signal check between batch and next request → next provider call fails immediately as `aborted` → persisted → hard exit (`agent-loop.ts:183-242`; e2e only covers abort during streaming, `packages/agent/test/e2e.test.ts:103-126,208-218`) (unverified).
8. **Session post-run**: `_handlePostAgentRun` sees `_agentRunAbortRequested` → `_finishCancelledRetry()` emits `auto_retry_end{success:false, finalError:"Retry cancelled"}` (`agent-session.ts:3745-3755`, `1857-1860`); `_checkCompaction` skips aborted messages post-run (`:2954-2955`) but the **pre-prompt** check considers them (`:2051-2056`).
9. **Replay**: aborted assistant stays in JSONL, skipped by `transformMessages` (`packages/ai/src/api/transform-messages.ts:195-203`); pi-ai README says aborted messages may be continued — works only because the partial is not replayed (`packages/ai/README.md:1162-1185`).
10. **Session replacement** (`/new`, `/resume`, `/fork`) calls `session.abort()` first so the aborted turn incl. tool results lands in the outgoing session (`agent-session-runtime.ts:167-178`; `cefa40ed8` #7022).
- **Classification by signal, not text**: abort detection moved from `message === "Compaction cancelled" || AbortError` to `signal.aborted`; summarization auth lookup abortable; `throwIfAborted()` checkpoints in auto-compaction (`agent-session.ts:3115-3119,3144,3174`; `de2de549b` #9340 #9777).
- **pi-ai abort utilities**: `raceWithAbortSignal(op, signal)` stops waiting but keeps observing the abandoned promise (no unhandled rejection) (`packages/ai/src/utils/abort.ts:13-50`); `operationSignal()` = never-aborting local signal so internals always get a concrete signal (`abort.ts:8-11`); `combineAbortSignals` with listener cleanup (`abort-signals.ts:6-41`). Provider retry sleep abortable → `AbortError("Request aborted")` (`provider-retry.ts:71-97,119`); `retryAssistantCall` normalizes abort during backoff to `{stopReason:"aborted"}` minus errorMessage (`retry.ts:204-208,229-236`; `243f64be5`).
- **Deliberately uncancellable**: OAuth refresh ignores caller signal once started (rotated refresh token must be persisted), bounded by 15 s (`packages/ai/src/auth/resolve.ts:104-155`) → [[subscription-oauth-auth]].
- Subagent example: signal → `SIGTERM`, `SIGKILL` after 5 s, then throws "Subagent was aborted" (`examples/extensions/subagent/index.ts:410-424`) → [[pi--subagent-as-subprocess|subagent-as-subprocess]].
- Chord (experimental runtime) uses Go-style `Context`: `BACKGROUND_CONTEXT`, `withCancel`, `withAbortSignal` (`AbortSignal.any`), `withoutAbortSignal` ("Intended for mandatory cleanup only"), `awaitWithContext` rejects the waiter only (`packages/chord/src/context/index.ts:55-116`); only the abort signal crosses RPC (`packages/client/src/client.ts:458`; server rebuilds `withAbortSignal(controller.signal, TODO_CONTEXT)`, `packages/server/src/server.ts:331`; `cancel` frame aborts the request's controller iff target matches, answers `{code:"cancelled"}`, `server.ts:298-304,396-398`). Keyed service `observe` handlers each get a fresh `withCancel(BACKGROUND_CONTEXT)`; closing an instance cancels only that task (`packages/chord/src/services/instances.ts:80-142`).
- pi-env daemon: each request's `Control` registered **before** it runs so a cancel right behind isn't lost; `cancel` aborts by default, `{mode:"kill"}` kills and lets the command settle (`packages/env/daemon/src/main.rs:394-397,426-431`; `packages/env/docs/protocol.md:45-50`); cancel replies scheduled ahead of bulk output (`main.rs:347-360`) → [[remote-execution-env]].

## Constants
| name | value | path:line |
|---|---|---|
| AbortControllers per run | 1 | `packages/agent/src/agent.ts:512` |
| OAuth refresh bound (uncancellable) | 15 s | `packages/ai/src/auth/resolve.ts:110-114,140` |
| subagent kill escalation | SIGTERM → SIGKILL after 5000 ms | `examples/extensions/subagent/index.ts:411-416` |
| process kill escalation (exec) | SIGTERM → SIGKILL after 5000 ms | `packages/coding-agent/src/core/exec.ts:61` |

## Evolution
- 2026-01-08 `a65da1c14` Esc not interrupting during "Working…" (Gemini CLI stream ignored signal; escape handler not restored).
- 2026-03-28 `7d4faa080` abort signal exposed to extensions (#2660).
- 2026-05-19 `b94482762` stop tool preflight after extension abort (#4276).
- 2026-05-28 `5b31ffd74` dispose aborts agent/compaction/summary/retry/bash (#5029).
- 2026-05-29 `a36a132c7` abort Codex SSE body reads.
- 2026-06-30 `2117b61c6` undici mid-stream client `error` event without listener crashed process.
- 2026-07-21 `243f64be5` aborted retry attempts reported unsuccessful.
- 2026-07-23 `7af8533c6` provider retries abortable (#6980).
- 2026-07-28 `cefa40ed8` abort + persist before tree navigation / session switch (#7022).
- 2026-08-03 `e56893f4c` manual vs auto compaction race (#7370).
- 2026-09-01 `afda4d620` stop prepared tools after preflight abort (#8936).
- 2026-09-03 `bea67d90d` session abort cancels compaction; idle includes compaction (#8920).
- 2026-09-07 `e687434a6` reject tree nav during compaction (#9179).
- 2026-09-19 `de2de549b` close compaction cancellation races (#9340, #9777).

## Evidence commits
`a65da1c14` `7d4faa080` `b94482762` `5b31ffd74` `a36a132c7` `2117b61c6` `243f64be5` `7af8533c6` `cefa40ed8` `e56893f4c` `afda4d620` `bea67d90d` `e687434a6` `de2de549b` `2efa728d2` `ee398cb6d`

## Quirks
- `ee398cb6d` (2026-09-01) "recover mini operations and forward aborts" — remote presentations had to forward abort explicitly.
- Mistral `AbortSignal.timeout(timeoutMs ?? 60_000)` spans the whole streaming request → generations >60 s cut and classified `error`, not `aborted` (`packages/ai/src/api/mistral-conversations.ts:307-308`) (whether coding-agent sets `timeoutMs`: unverified).
- pi-messages error event is a fresh empty message, dropping already-streamed partial content (`packages/ai/src/api/pi-messages.ts:323-345`).
- Bedrock may bypass the abortable retry wrapper (unverified).

## Durable variant (packages/durable)
- Abort protocol: commit `abortRequested` → signal + join → wait for owned work → abort invocation → abort handler commits terminal (`packages/durable/docs/spec.md:1935-1941`). Abort flows down the ownership tree; **bottom-up** abort order so handlers see final child outcomes; background tasks are boundaries (`spec.md:2089-2176`) → [[pi--task-owned-subagent|task-owned-subagent]].
- `Conversation.abort()` withdraws queued inputs (writes stay), aborts every task of current work, resolves once idle; `{background:true}` also reaches background tasks (`packages/durable/README.md:416,426`).
- Generation checks `runtime.signal.throwIfAborted()` before classifying a response (`generation.ts:426`). Abort cascades derived in a separate reconcile commit, re-run at open (`spec.md:1990-1998`).
- Tool crash ≠ abort: interrupted non-replay-safe tool → "Tool X was interrupted and may have partially run" (`harness/tool.ts:93-110`) → [[crash-safe-tool-replay]].

## Failures
[[tool-preflight-ignores-abort]] · [[provider-stream-ignores-abort]] · [[compaction-cancellation-races]] · [[session-switch-leaves-dangling-tool-calls]] · [[retry-backoff-hygiene]] · [[ownership-cancellation-races]]
