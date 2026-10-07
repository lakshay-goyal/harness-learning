---
type: concept
stage: subagents
tier: variant
aliases: [codex_delegate, run_codex_thread_interactive, run_codex_thread_one_shot, "SubAgentSource::Compact", "SubAgentSource::MemoryConsolidation", SessionIsolation, AgentRunner, spawn_legacy_subagent]
harnesses: [codex]
---
A generic in-process "run a child agent session" primitive (interactive or one-shot) used by internal harness features — review, LLM approval reviewer, memory consolidation, detached skills — rather than by the model.

## Why
- Every internal LLM side task (review, approvals, memory) otherwise re-implements session setup, cancellation, shutdown and event filtering.
- Internal children must not prompt the user and must not accept further input after their single task ([[no-delegate-approvals]]).

## Design space
- **Side LLM call** without tools (pi compaction/summaries, [[auxiliary-model-calls]]) · full child agent session with tools (✔ codex).
- **Mode**: interactive (caller keeps submitting ops) · one-shot: first `TurnComplete`/`TurnAborted` auto-sends `Shutdown`, submit channel pre-closed (✔ codex).
- **Isolation**: inherit parent's extensions/services · isolated empty extension registry (✔ codex `SessionIsolation`).
- **Context**: fresh · fork parent full history then submit a rendered prompt (✔ codex `AgentRunner`).
- **Approvals**: ask user · forced `never`; risky actions go to an automatic reviewer instead (✔ codex).

## Implementations
- [[codex--delegate-session-runner|codex]] — `codex-rs/core/src/codex_delegate.rs` (interactive / one-shot, approval `never` required) + `codex-rs/ext/agent` `AgentRunner` (forked context, `start_turn_if_idle`).

## Failures
- [[review-agent-edits-code]]

## Related
[[review-subagent]] · [[llm-approval-reviewer]] · [[cross-session-memory]] · [[in-process-subagent-threads]] · [[auxiliary-model-calls]] · [[session-fork]] · [[request-attribution-metadata]] · [[builtin-subagents-vs-none]]
