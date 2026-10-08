---
type: implementation
harness: codex
concept: current-time-reminder
commit: 622e9e3696
files: [codex-rs/core/src/context/current_time_reminder.rs:20, codex-rs/core/src/session/time_reminder.rs:17, codex-rs/core/src/session/time_reminder.rs:46, codex-rs/core/src/session/time_reminder.rs:96, codex-rs/core/src/session/turn.rs:497, codex-rs/core/src/config/mod.rs:1328, codex-rs/features/src/lib.rs:1861, codex-rs/core/src/context/world_state/environment.rs:143]
---
[[current-time-reminder]] in [[codex]].

## Mechanism
- **Fragment**: developer role, tag `<current_time_reminder>`, body `It is {%Y-%m-%d %H:%M:%S UTC}.` (`codex-rs/core/src/context/current_time_reminder.rs:20-42`); content kind `current_time.reminder`. Clock failure → `<current_time_unavailable>failed to read current time</current_time_unavailable>`.
- **Call site**: every loop iteration before world state is rendered (`codex-rs/core/src/session/turn.rs:497-502`), i.e. before each sampling request, not just per user turn.
- **Due logic** (`codex-rs/core/src/session/time_reminder.rs:46-92`): new context window (after compaction) ⇒ always; `reminder_interval_seconds == 0` ⇒ always; else elapsed ≥ interval; delivery mode `AfterUserOrToolOutput` additionally requires a user message or tool output recorded since the last inference (`note_recorded_items`, `:49`); `AnyInference` does not.
- **Clock**: host-pluggable `TimeProvider` (`CurrentTimeSource::{System, External}`) with cancellable `sleep` used by the optional `clock.sleep` tool (`codex-rs/core/src/current_time.rs`, `codex-rs/core/src/config/mod.rs:1328-1343`) → [[codex--wall-clock-tools]]. Read failure is Fatal unless `NonfatalClockReadErrors`, then a one-time unavailable fragment (`codex-rs/core/src/session/time_reminder.rs:96-146`).
- **Persistent mode**: `apply_persistent_defaults` turns the reminder on with the sleep tool unless the user set it (`codex-rs/core/src/session/time_reminder.rs:17`) → [[codex--persistent-agent-mode]].
- **Always-on alternative**: `<current_date>` + `<timezone>` inside the user-role `<environment_context>` (date only, UTC fallback), re-emitted as a world-state diff when the date changes (`codex-rs/core/src/context/world_state/environment.rs:143-148`; example `codex-rs/core/src/context/world_state/environment_render_tests.rs:87-123`).

## Constants
| name | value | path:line |
|---|---|---|
| `reminder_interval_seconds` default | 1 s (0 = every request) | `codex-rs/core/src/config/mod.rs:1336-1343` |
| delivery mode default | `AnyInference` | `codex-rs/core/src/config/mod.rs:1336-1343` |
| sleep tool default | `false` | `codex-rs/core/src/config/mod.rs:1336-1343` |
| feature stage | `Stage::UnderDevelopment`, default off | `codex-rs/features/src/lib.rs:1861-1865` |

## Evolution
- 2026-02-26 `90cc4e79a2` `<current_date>` + `<timezone>` in `<environment_context>` (date only).
- 2026-06-18 `752ed90d78` "current time reminders impl for system clock (varlatency 2/n) (#28824)": "record UTC developer reminders in history immediately before due model requests… force a refresh after compaction"; same day `73251b2f00` clock tool.
- 2026-06-23 `9fe689783d` "debounce current-time reminders by elapsed time (#29659)".
- 2026-06-25 `cc78903379` interval may be 0 (#30029).
- 2026-08-13 `3ba52d6075` tagged with XML markers.

## Versus pi
- pi: [[no-date-in-prompt]] — no date or time at all. codex keeps time out of the cached base prompt too, but re-adds date in user-role context and (opt-in) exact time as appended developer history.
