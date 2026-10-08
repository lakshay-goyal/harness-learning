---
type: failure
concepts: [shell-execution]
harnesses: [codex]
---
**Symptom** — In the Windows unified-exec experiment, `exec_command` yielded (returned a session id) while commands were still starting, so the model spent extra tool/model cycles polling: tool calls per turn +20.7 %, blended tokens per turn +4.1 %, output tokens per turn +4.0 %, latency per turn +8.3 %; turns −6.6 %, DAU −1.0 %, hourly active users −3.0 % — while per-response tokens and latency went *down* (`e7a9988d1a` body).

**Root cause** — Shell-wrapped commands on Windows pay a large PowerShell startup/teardown tax before the command runs: ~740 ms p50 / 800 ms p90 (Windows PowerShell) and ~930 ms p50 / 980 ms p90 (`pwsh`) over direct exec, eating short yield windows.

**Fix · [[codex]]**
- `e7a9988d1a` 2026-06-15 "Add Windows unified exec yield floor (#27086)" — `WINDOWS_INITIAL_EXEC_YIELD_TIME_FLOOR_MS = 2_000` applied before the [250, 30000] clamp (commit body: "Windows-only 2s floor").
- `fd41e813cb` 2026-07-27 "Raise the Windows exec yield floor to 10 seconds" — floor 10_000 (`codex-rs/core/src/unified_exec/mod.rs:74,218-224`); Windows description: "Effective range on Windows is 10000-30000 ms" (`codex-rs/core/src/tools/handlers/shell_spec.rs:30-31`).
- Related tuning: `32b1795ff4` 2026-01-14 min 5 s for empty `write_stdin` polls ("After evals, 0 impact on performance"); `547f462385` 2026-02-19 empty-poll max 30 s → 5 min.

**Lesson** — A yield/poll shell needs a platform-tuned minimum wait; measure turn-level metrics (tool calls per turn), not per-response cost, or the model silently burns turns polling.

Related: [[shell-execution]] · [[codex--shell-execution|codex]] · [[windows-process-tree-and-shells]] · [[stale-background-process-status]]
