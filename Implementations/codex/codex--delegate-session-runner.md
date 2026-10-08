---
type: implementation
harness: codex
concept: delegate-session-runner
commit: 622e9e3696
files: [codex-rs/core/src/codex_delegate.rs:52, codex-rs/core/src/codex_delegate.rs:64, codex-rs/core/src/codex_delegate.rs:215, codex-rs/ext/agent/src/lib.rs:33]
---
[[delegate-session-runner]] in [[codex]].

## Mechanism
- **Interactive delegate** `run_codex_thread_interactive`: shares the parent's skills service, environment manager, auth/models managers; extensions either inherited or an empty registry when `SessionIsolation::Isolated`; WebSockets only if the parent's client has them (`codex-rs/core/src/codex_delegate.rs:52-110`).
- **Approval rule**: "Codex delegates require approval policy `never`" (`codex-rs/core/src/codex_delegate.rs:64-73`) → [[no-delegate-approvals]]; risky actions of V2 workers go to the automatic reviewer instead ([[llm-approval-reviewer]]).
- **One-shot** `run_codex_thread_one_shot`: child cancel token; first `TurnComplete`/`TurnAborted` auto-sends `Op::Shutdown`; returned submit channel is pre-closed so callers cannot submit more ops (`codex-rs/core/src/codex_delegate.rs:215-322`). Used by `/review` ([[review-subagent]]).
- **Sources**: `SubAgentSource::{Review, Compact, ThreadSpawn, MemoryConsolidation, Other}` tag every delegate; surfaced as `x-openai-subagent` header ([[request-attribution-metadata]]); memory consolidation sessions turn Stop-hook block/stop into errors (`codex-rs/core/src/session/turn.rs:629-637`) → [[cross-session-memory]].
- **`AgentRunner::start`** (`codex-rs/ext/agent/src/lib.rs:33-101`): fork parent full history via `ThreadManager::spawn_legacy_subagent`, submit the prompt with `start_turn_if_idle`, return (thread_id, turn_id) — "Agent discovery owns rendering prompt, including any selected skill references. The runtime only starts that prompt in isolated forked context." (`6629e08702`). Used by detached review; refused for paginated-history threads (`codex-rs/app-server/src/request_processors/turn_processor.rs:1464-1500`).

## Evolution
- 2025-10-29 `13e1d0362d` "Delegate review to codex instance (#5572)".
- 2025-11-28 `aaec8abf58` "feat: detached review".
- 2026-07-14 `6629e08702` `ext/agent` `AgentRunner`.

## Versus pi
- pi runs side LLM work (compaction, branch summary) as tool-less side calls, not as child agent sessions ([[pi--abort-propagation]]); codex uses full child sessions with tools for review, approvals and memory.
