---
type: concept
stage: tools
tier: candidate
aliases: [clock.curr_time, clock.sleep, SleepToolMode, SleepHandler, CurrentTimeHandler, sleep tool, interruptible sleep]
harnesses: [codex]
---
Namespaced clock tools for long-running or waiting agents: read the current wall-clock time, and sleep for a bounded duration that ends early when new user input arrives.

## Why
- An agent waiting on something external (CI, a deploy, a background process) otherwise busy-polls with shell `sleep` (uninterruptible, burns tool calls) or ends its turn.
- The model has no reliable sense of time; time in the system prompt breaks caching ([[no-date-in-prompt]], [[current-time-reminder]]).
- A blocking sleep must not make the agent deaf to the user — it has to wake on new input ([[steering-queue]]).

## Design space
- **No clock tools; date in prompt or none** (pi).
- Time injected as appended history reminders (codex [[current-time-reminder]]) vs **pull tool** (codex `clock.curr_time`) vs both.
- Sleep: shell `sleep` (uninterruptible) vs **harness sleep tool interrupted by new input** (codex `clock.sleep`) vs wait primitives inside other tools (codex `write_stdin` empty polls, `wait_agent`).
- Bound: max duration (codex 12 h) and prompt guidance against long blocking waits (codex catalog: "Avoid performing blocking sleep or wait calls longer than 60 seconds", `3380969a29`).
- Gate: always-on vs model-driven (codex `SleepToolMode`, model catalog advertising `clock`) vs implied by an autonomy mode (codex persistent mode force-enables reminder + sleep → [[persistent-agent-mode]]).
- Pluggable clock source (system vs host-provided) so embedders/tests control time (codex `TimeProvider`).

## Implementations
- [[codex--wall-clock-tools|codex]] — `clock` namespace: `curr_time()` → "YYYY-MM-DD HH:MM:SS UTC"; `sleep(duration_ms)` 1 ms..12 h, wakes on new turn input, reports wall time.

## Failures
- (none recorded)

## Related
[[current-time-reminder]] · [[persistent-agent-mode]] · [[persistent-goal-continuation]] · [[shell-execution]] · [[elicitation-pause]] · [[tool-wire-kinds]] · [[steering-queue]]
