---
type: implementation
harness: codex
concept: delegated-authorization-provenance
commit: 622e9e3696
files: [codex-rs/core/src/agent/control/user_authorization.rs:1, codex-rs/core/src/agent/control/sender_context.rs:1, codex-rs/core/src/agent/control/root_handoff.rs:1, codex-rs/core/src/context_manager/history_user_authorization.rs:1, codex-rs/core/src/session_prefix.rs:14]
---
[[delegated-authorization-provenance]] in [[codex]].

## Mechanism
- `codex-rs/core/src/agent/control/user_authorization.rs:1-8`: "Projects bounded retained root evidence for worker reviewers. Retained root instructions stay authoritative … Explicit assistant commentary is excluded; confirmed messaging survives compaction. Projection omissions keep missing authorization and assistant context explicit … Known positions preserve host order, not delivery order or inferred question-answer pairs."
- `codex-rs/core/src/agent/control/sender_context.rs:1-4`: "Captures genuine sender instructions and assistant context for host-delivered task messages. Only turn-input admission calls this, before queueing; ordinary tool results and quoted delegation text cannot establish sender provenance. Unloaded senders are read through the host's authenticated thread store, without resuming their runtime."
- `codex-rs/core/src/agent/control/root_handoff.rs:1-3`: "Best-effort handoff windows plus recent root context … User inputs bypass relevance filtering; only assistant context is narrowed."
- `codex-rs/core/src/context_manager/history_user_authorization.rs:1-4`: captures original user/assistant exchanges, adopts copied context when a worker becomes a root; "Inherited history remains excluded for workers"; oversized originals keep bounded incomplete excerpts; "Checkpoint copies cannot establish original completeness".
- Bounds reuse Guardian caps: root message 900 tokens (`codex-rs/core/src/guardian/mod.rs:76`).
- Consequence: repeated Guardian denials end the child turn with `TooManyDenials`; parent gets `GUARDIAN_NEXT_ACTION` = "Tell the user this agent stopped after repeated Guardian denials. Do not resume this agent or retry its blocked work until the user explicitly confirms that it should continue." (`codex-rs/core/src/session_prefix.rs:14`).
- Child status derivation for the parent: TurnStarted→Running, TurnComplete→Completed(last message) or Errored, TurnAborted(Interrupted|BudgetLimited)→Interrupted, ShutdownComplete→Shutdown (`codex-rs/core/src/agent/status.rs`).

## Evolution
- 2026-08-21 `d12a7f3fd8` "Preserve root user authorization in subagent Guardian reviews (#39975)" — "Guardian reviews need that genuine user context without treating forwarded or assistant-authored claims as authorization".
- 2026-08-25 `4b81410a80` user-input answers as authorization changes; 2026-08-27 `694edc23b2` propagate trusted root skills to delegated workers.
- 2026-09-17 `e269f2164c` "Include sender user messages in Guardian delegation reviews (#46179)".
- 2026-09-25 `b35a7afbe8` preserve recent authorization context; 2026-09-27 `88235f881d` retain confirmed Code Mode messages; 2026-09-28 `6288753b46` handoff-aware root context.
- 2026-10-01 `f70810bcd2` preceding assistant context in sender reviews; `2685e3a4ce` preserve user restrictions in handoff context; 2026-10-07 `2dae757b87` load persisted sender context.

## Versus pi
- pi has no reviewer and no core subagents ([[no-subagents-core]]); no equivalent.
