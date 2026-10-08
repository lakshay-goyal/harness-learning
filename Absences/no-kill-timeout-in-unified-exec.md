---
type: absence
harnesses: [codex]
---
# no-kill-timeout-in-unified-exec

Model commands have no wall-clock kill timeout in the default mode: a long command yields a session id and keeps running until the session ends.

**What's missing**
- `codex-rs/core/src/session/handlers.rs:305-312`; yield windows `MIN_YIELD_TIME_MS = 250`, `MIN_EMPTY_YIELD_TIME_MS = 5_000`, `MAX_YIELD_TIME_MS = 30_000`, `DEFAULT_MAX_BACKGROUND_TERMINAL_TIMEOUT_MS = 300_000` (`codex-rs/core/src/unified_exec/mod.rs:73-78`).

**Evidence of decision**
- "No timeout mode" experiment added and reverted the same day (`9719dc502c` / `928be5f515` 2026-02-19).
- Yield tuning: `0792a7953d` 2025-11-13 default yield 10 s exec / 250 ms write_stdin; `32b1795ff4` 2026-01-14 clamp min yield for empty write_stdin ("After evals, 0 impact on performance"); `547f462385` 2026-02-19 empty-poll max 30 s → 5 min.

**Implication**
- Maps loosely to [[no-bash-default-timeout]] but for a different reason: the yield model returns control to the model, so blocking is impossible without a kill; runaway processes are bounded by session end and interrupt.

Related: [[shell-execution]] · [[no-bash-default-timeout]] · [[no-background-bash]] · [[process-tree-kill]] · [[elicitation-pause]] · [[Absences]]
