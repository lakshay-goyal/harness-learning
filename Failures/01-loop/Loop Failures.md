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

## Queues
- [[steering-skips-pending-tool-calls]] — A mid-run user message skipped the rest of the model's tool batch with fake error results.
- [[queued-messages-stranded-at-run-end]] — Queued messages stuck after auto-compaction or when queued by `agent_end` handlers.
- [[side-phase-input-lost]] — Input typed during compaction/branch summary/tree navigation dropped, stuck or re-typed.

## Cancellation
- [[tool-preflight-ignores-abort]] — After abort, sibling tools kept preparing/confirming and prepared tools still ran.
- [[provider-stream-ignores-abort]] — Esc did not stop a provider stream; mid-stream abort crashed the process.
- [[compaction-cancellation-races]] — Abort missed or raced compaction/summary; RPC abort reported success while compaction ran.
- [[session-switch-leaves-dangling-tool-calls]] — Session switch / tree nav mid-response left tool calls without results.

## Retry
- [[retry-counter-accumulates-across-turn]] — Three successful single retries in one run exhausted "3/3".
- [[retry-classifier-regex-sprawl]] — Transient errors ended runs because their text did not match the retry regex.
- [[retry-backoff-hygiene]] — Retry sleeps not abortable, unbounded, or fired immediately on bad Retry-After.
- [[hidden-sdk-retries-double-retry]] — SDK retries under the agent retry burned quota and hid terminal 429s.
- [[deterministic-5xx-retried]] — Deterministic 504 for huge inputs retried pointlessly.
- [[abandoned-attempts-left-in-context]] — Failed retry/recovery attempts stayed in future provider context.

## Stream & truncation integrity
- [[truncated-stream-accepted-as-success]] — Streams without a terminal event persisted as successful partial answers.
- [[length-truncated-tool-calls-executed]] — Tool calls with truncated/unfinalized arguments executed after a length stop.
- [[length-stop-recovery]] — Context-ceiling truncation ended the run silently or was mislabeled overflow.

Related cross-group: [[error-text-breaks-retry-classification]] · [[foreign-sdk-error-shape-skips-retry]] · [[stop-reason-mapping-gaps]] · [[hook-throw-aborts-parallel-batch]] · [[parallel-tool-results-order]] · [[interactive-tool-in-parallel-batch]] · [[non-idempotent-tool-replayed-after-crash]].
