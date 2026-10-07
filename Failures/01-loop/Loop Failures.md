---
type: group
group: 01-loop
---
Failures whose first concept is in [[Loop]].

## Turn loop & settlement
- [[reentrant-prompt-corrupts-state]] — `prompt()`/`continue()`/`reset()` during a run raced and corrupted agent state.
- [[late-tool-progress-after-settlement]] — Tool progress events arrived after `tool_execution_end`.
- [[listeners-see-stale-agent-state]] — Event handlers read state that did not yet include the event.
- [[proxied-stream-option-loss]] — Proxied/side model calls dropped session, cache, thinking options and namespaces.
- [[retry-wait-race-prompt-returns-early]] — `prompt()` resolved while an auto-retry (with tool calls) was still running.
- [[tool-loadout-stale-within-run]] — Tool changes mid-run not applied to the next request; forced prompt dropped on refresh.
- [[prewarm-blocks-turn-start]] — WebSocket prewarm held `turn/start` up to 5 min; no `turn/started`, interrupt blocked (codex).
- [[pending-input-restarts-failed-turn]] — Pending input/agent mail re-ran a terminally failed turn (codex).
- [[interrupted-message-never-finalized]] — Abort/error left the assistant record open; session stuck "Working…" or busy.

## Queues
- [[steering-skips-pending-tool-calls]] — A mid-run user message skipped the rest of the model's tool batch with fake error results.
- [[queued-messages-stranded-at-run-end]] — Queued messages stuck after auto-compaction or when queued by `agent_end` handlers.
- [[side-phase-input-lost]] — Input typed during compaction/branch summary/tree navigation dropped, stuck or re-typed.

## Autonomous continuation (goals)
- [[goal-continuation-runaway]] — `/goal` kept synthesizing turns on permanent errors, usage exhaustion, repeated blockers, empty answers (codex).
- [[goal-loop-premature-stop]] — "Wait for new input" escape hatch + no-tool-call heuristic stopped goals early (codex).
- [[goal-scope-shrinking]] — Model declared success on a smaller, easier-to-test subset of the objective (codex).
- [[time-pressure-shortcuts]] — Elapsed time in the continuation prompt made the model shortcut work (codex).
- [[steering-message-re-answered]] — Hidden developer-role steering re-answered after new turns and mangled by compaction (codex).

## Cancellation
- [[tool-preflight-ignores-abort]] — After abort, sibling tools kept preparing/confirming and prepared tools still ran.
- [[provider-stream-ignores-abort]] — Esc did not stop a provider stream; mid-stream abort crashed the process.
- [[compaction-cancellation-races]] — Abort missed or raced compaction/summary; RPC abort reported success while compaction ran.
- [[session-switch-leaves-dangling-tool-calls]] — Session switch / tree nav mid-response left tool calls without results.
- [[interrupted-turn-invisible-to-model]] — Interrupt left no model-visible trace; model repeated aborted side effects (codex).
- [[interrupt-kills-background-processes]] — Ctrl+C also killed dev servers / watchers started by the agent (codex).
- [[approval-wait-surfaces-as-rejection-on-interrupt]] — Clearing approvals before cancelling the task looked like a user rejection (codex).
- [[interrupt-rpc-hangs-on-finished-turn]] — `turn/interrupt` on an already-finished turn never answered (codex).

## Retry
- [[retry-counter-accumulates-across-turn]] — Three successful single retries in one run exhausted "3/3".
- [[retry-classifier-regex-sprawl]] — Transient errors ended runs because their text did not match the retry regex.
- [[retry-backoff-hygiene]] — Retry sleeps not abortable, unbounded, or fired immediately on bad Retry-After.
- [[hidden-sdk-retries-double-retry]] — SDK retries under the agent retry burned quota and hid terminal 429s.
- [[deterministic-5xx-retried]] — Deterministic 504 for huge inputs retried pointlessly.
- [[abandoned-attempts-left-in-context]] — Failed retry/recovery attempts stayed in future provider context.
- [[server-retry-advice-ignored]] — Retry-After / "try again in" advice ignored across HTTP, SSE events, WebSocket and fallback (codex).
- [[content-filter-retry-without-guidance]] — Content-filter stops retried verbatim; fix injects model-specific guidance first (codex).

## Stream & truncation integrity
- [[truncated-stream-accepted-as-success]] — Streams without a terminal event persisted as successful partial answers.
- [[length-truncated-tool-calls-executed]] — Tool calls with truncated/unfinalized arguments executed after a length stop.
- [[length-stop-recovery]] — Context-ceiling truncation ended the run silently or was mislabeled overflow.

Related cross-group: [[turn-items-lost-on-abort]] · [[compaction-drops-pending-prompt]] · [[unbounded-hook-continuation-loop]] · [[plugin-hook-wall-clock-timeout]] · [[stream-stall-without-header-timeout]] · [[transport-fallback-after-partial-output]] · [[error-text-breaks-retry-classification]] · [[foreign-sdk-error-shape-skips-retry]] · [[stop-reason-mapping-gaps]] · [[hook-throw-aborts-parallel-batch]] · [[parallel-tool-results-order]] · [[interactive-tool-in-parallel-batch]] · [[non-idempotent-tool-replayed-after-crash]].
- [[step-limit-enforced-only-by-prompt]] — legacy last step says "Tools are disabled" but still sends tools; `toolChoice:"none"` dropped in a refactor (v2 restores it).

## Step budget & repetition
- [[identical-tool-call-loop]] — The model re-issued the same tool call with identical arguments until aborted.
- [[step-budget-not-reset-on-new-input]] — A steered/queued prompt inherited the used step budget and lost its tools immediately.
- [[rewrite-introduces-unrequested-limits]] — A runtime port invented a 25-step hard cap that failed long tasks.
