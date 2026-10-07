---
type: concept
stage: loop
tier: candidate
aliases: [ElicitationService, ElicitationRegistration, subscribe_elicitation_pause_state, out-of-band elicitation count, stopwatch pausing, wait_until_clear]
harnesses: [codex]
---
While any user elicitation (approval prompt, question, out-of-band client dialog) is outstanding, the session is marked paused so time-based tool limits stop counting user think-time and results wait until the user has answered.

## Why
- Tool timeouts and yield windows measured in wall time expire while the human reads an approval dialog → commands killed or returned "still running" because the user was slow.
- Concurrent elicitations (parallel tools, MCP servers, client dialogs) need ref-counting or the first answer un-pauses the others.

## Design space
- **Signal**: per-tool timer suspension · session-wide ref-counted paused flag on a watch channel, flipped on 0↔1 transitions (✔ codex).
- **Consumers**: shell timeouts subscribe to the flag (✔ codex unified exec) · result delivery waits until clear (✔ codex code-mode cells).
- **Out-of-band sources**: client-side dialogs counted via a thread API (✔ codex `increment/decrement_out_of_band_elicitation_count`).
- **None**: pi has no permission prompts, so no pause is needed ([[no-permission-prompts]]).

## Implementations
- [[codex--elicitation-pause|codex]] — `ElicitationService` ref-count → `watch<bool>` paused; unified exec and code mode consume it.

## Failures
none recorded.

## Related
[[approval-policy-modes]] · [[structured-user-question-tool]] · [[shell-execution]] · [[code-mode]] · [[mcp-integration]] · [[tool-call-gate]]
