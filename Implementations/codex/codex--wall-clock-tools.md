---
type: implementation
harness: codex
concept: wall-clock-tools
commit: 622e9e3696
files: [codex-rs/core/src/tools/handlers/sleep.rs:26-156, codex-rs/core/src/tools/handlers/current_time.rs:7-80, codex-rs/core/src/tools/spec_plan.rs:1300-1326, codex-rs/core/src/current_time.rs, codex-rs/core/src/config/mod.rs:1328-1343]
---
[[wall-clock-tools]] in [[codex]].

## Mechanism
- Both tools live in namespace `clock` ("Tools for reading and waiting on time.") as `ToolSpec::Namespace` members ([[tool-wire-kinds]]) (`codex-rs/core/src/tools/handlers/sleep.rs:26-27,49`; `codex-rs/core/src/tools/handlers/current_time.rs:7-8,58`).
- `clock.curr_time()` — "Return the current time in UTC."; no params; model-facing body text; code-mode result `{current_time}` with output schema "Current UTC time formatted as YYYY-MM-DD HH:MM:SS UTC." (`current_time.rs:41-80`).
- `clock.sleep(duration_ms)` — "Pause execution for a specified duration. The sleep ends early when new input arrives for the active turn. Returns the elapsed wall-clock time."; param "How long to sleep in milliseconds. Must be between 1 and {MAX_SLEEP_DURATION_MS}."; `deny_unknown_fields` (`sleep.rs:31-60`).
  - Out of range → `RespondToModel("duration_ms must be between 1 and 43200000")`; clock read failure → `RespondToModel("failed to read current time for the sleep tool; the clock provider may be stalled")`; sleep failure → `Fatal("failed to sleep: …")` (`sleep.rs:96-145`).
  - Output "Wall time: {s:.4} seconds\nSleep interrupted by new input." or "…\nSleep completed." (`sleep.rs:150-156`).
- Clock = host-pluggable `TimeProvider` (System / External) with cancellable sleep (`codex-rs/core/src/current_time.rs`; config `codex-rs/core/src/config/mod.rs:1328-1343`).
- Gates (`codex-rs/core/src/tools/spec_plan.rs:1300-1326`): `curr_time` when `Feature::CurrentTimeReminder` or catalog `experimental_supported_tools` contains `clock`; `sleep` when `Feature::SleepTool` (Stable, on, `codex-rs/features/src/lib.rs:1027-1028`) AND (`SleepToolMode::AlwaysOn` OR ModelDriven: reminder config `sleep_tool = true` if reminders on, else model has `clock`). Reminder config default `sleep_tool = false` (`config/mod.rs:1336-1343`).
- Persistent mode force-enables the current-time reminder + sleep tool (`apply_persistent_defaults` in `codex-rs/core/src/session/time_reminder.rs`, M5a) → [[persistent-agent-mode]].
- Related prompt rule in bundled catalog: "Avoid performing blocking sleep or wait calls longer than 60 seconds" (`3380969a29` 2026-07-09, models.json).

## Constants
| name | value | path:line |
|---|---|---|
| `MAX_SLEEP_DURATION_MS` | 12 h (43_200_000 ms) | `codex-rs/core/src/tools/handlers/sleep.rs:28` |
| min sleep | 1 ms | `codex-rs/core/src/tools/handlers/sleep.rs:41-44` |

## Evolution
- 2026-06-15 `08901fc8e1` "[codex] Add interruptible sleep tool (#28429)".
- 2026-06-18 `752ed90d78` current-time reminders (varlatency 2/n) → [[current-time-reminder]].
- 2026-06-19 `73251b2f00` "[codex] add clock current-time tool (#29011)".
- 2026-06-24 `35f5d02464` sleep config nested under current-time reminder.

## Versus pi
pi has neither tool; long waits happen via `bash sleep` with no default timeout ([[pi--shell-execution]]).
