---
type: concept
stage: context
tier: candidate
aliases: [CurrentTimeReminder, "<current_time_reminder>", "<current_time_unavailable>", CurrentTimeReminderDeliveryMode, TimeProvider, current_time_reminder, varlatency, "<current_date>"]
harnesses: [codex]
---
Inject the wall-clock time as a small developer message appended to history just before a model request when due (interval, new context window, or after new user/tool input), instead of putting a date in the cache-sensitive system prompt.

## Why
- A date in the system prompt busts the prompt cache every day ([[volatile-system-prompt-prefix]], pi [[no-date-in-prompt]]); no time at all makes long-running/persistent agents reason from a stale "now".
- Waiting/polling agents (persistent mode, sleep tool) need exact time to judge elapsed time and deadlines.
- Appending a note keeps the prefix stable; debouncing keeps the note from flooding history.

## Design space
- No date/time anywhere ✔ pi ([[no-date-in-prompt]]).
- Date in the system prompt (rejected by pi for cache).
- **Date + timezone in a user-role context block, re-emitted only when the date changes** ✔ codex (`<current_date>`, `<timezone>` in `<environment_context>`, `90cc4e79a2`).
- **Exact time as appended developer note, cadence-gated** ✔ codex (opt-in): due on new context window, interval elapsed (0 = every request), and optionally only after a user message or tool output (`AfterUserOrToolOutput`).
- Time as a tool (`clock` read / interruptible `sleep`) ✔ codex → [[wall-clock-tools]].
- Clock failure: fatal vs one-time "failed to read current time" fragment ✔ codex (`NonfatalClockReadErrors`).
- Pluggable time source (system vs host-provided external clock) ✔ codex.

## Implementations
- [[codex--current-time-reminder|codex]] — `<current_time_reminder>It is YYYY-MM-DD HH:MM:SS UTC.</current_time_reminder>` developer fragment recorded each loop iteration when due; feature `current_time_reminder` under development, default off, forced on for persistent mode.

## Failures
- (none recorded)

## Related
[[env-vars-as-context]] · [[message-role-layering]] · [[world-state-diff-injection]] · [[wall-clock-tools]] · [[persistent-agent-mode]] · [[cache-stable-prompt-prefix]] · [[no-date-in-prompt]]
