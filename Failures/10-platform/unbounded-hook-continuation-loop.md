---
type: failure
concepts: [extension-event-hooks, run-settlement, persistent-goal-continuation]
harnesses: [pi, codex]
---
**Symptom** — (documented hazard) An extension that unconditionally returns `continue:true` from `turn_end` / `agent_before_settle` makes the agent request another model turn forever.

**Root cause** — The settle boundary lets plugins request exactly ONE more request per boundary (`packages/coding-agent/src/core/extensions/types.ts:944-1003`), but nothing caps how many boundaries chain; pi has no max-turn cap anywhere (`packages/agent/src` has no numeric literal ≥10) → [[no-turn-cap]].

**Fix · [[pi]]** — No code guard; documented warning only (`packages/coding-agent/docs/extensions.md:117`). Continuation is rejected only when context can't continue (last role assistant, nothing queued) (`agent-session.ts:999-1036`).

**Fix · [[codex]]**
- *Stop hooks*: a Stop hook that blocks (JSON `{"decision":"block","reason":…}` or exit 2 with stderr reason) injects its reason as a continuation prompt and the loop continues with `stop_hook_active = true` (`codex-rs/core/src/session/turn.rs:645-668`; aggregation `codex-rs/hooks/src/events/stop.rs:257-445`). No cap on how many times a Stop hook can reopen the turn — the loop relies on the hook honouring `stop_hook_active` (no other guard seen in `codex-rs/core/src/session/turn.rs:618-725`; unverified that none exists elsewhere). The base `run_turn` loop has no counter (`codex-rs/core/src/session/turn.rs:424-837`) → [[no-turn-cap]]. Only exception: memory-consolidation sessions turn a block/stop into an error, "Do not feed managed rejections back into an unattended memory loop" (`codex-rs/core/src/session/turn.rs:629-637`).
- *Autonomous goal continuation* (same class, harness-driven): `/goal` kept synthesizing turns with no possible progress; usage-limit failures replayed failing turns; repeated blockers burned tokens (#22833, #22245, #23067). Fixed by explicit terminal states — `0d344aca9b` 2026-05-18 `blocked` / `usageLimited` goal states + 3-turn Blocked audit; `0cdb1f1c83` 2026-08-25 no-progress classification and equivalent-blocker rule; breaker thresholds 3 empty / 3 exec-failure / 3 blocked turns (`codex-rs/ext/goal/src/accounting.rs:160`, `:229`). Details → [[goal-continuation-runaway]].

**Lesson** — Any mechanism that can extend a run (plugin settle hook, external Stop hook, autonomous goal loop) needs a termination condition the harness owns — explicit terminal states (blocked, out of quota) plus a repetition/no-progress threshold or budget; a flag the hook is trusted to honour (`stop_hook_active`) or a documented warning leaves loops one bug away.

Related: [[extension-event-hooks]] · [[run-settlement]] · [[turn-lifecycle-hooks]] · [[persistent-goal-continuation]] · [[goal-continuation-runaway]] · [[no-turn-cap]] · [[pi--extension-event-hooks|pi]] · [[codex--turn-lifecycle-hooks|codex]] · [[codex--persistent-goal-continuation|codex]]
