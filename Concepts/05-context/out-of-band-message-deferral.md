---
type: concept
stage: context
tier: must-have
aliases: [_pendingBashMessages, _pendingCustomMessages, _pendingNextTurnMessages, "triggerTurn: false", "deliverAs: nextTurn", "!cmd during streaming", inject_no_new_turn, PendingTurnInput]
harnesses: [pi, codex]
---
Messages produced while a run is streaming but not addressed to the current turn (user shell commands, plugin notes) are buffered and appended only at a turn boundary, so they never land between an assistant tool call and its results.

## Why
- tool_use → tool_result adjacency is a provider invariant; a note appended mid-turn sits between call and result and providers that validate order reject the replayed history — on every later request (pi: [[side-channel-message-splits-tool-pair]]).
- Message events must not describe messages the session log doesn't yet contain (UI/log consistency).
- "Don't trigger a turn" must not silently steer the running turn either (`47b5119d0`).

## Design space
- Append immediately (broken) vs **buffer to turn end** ✔ pi custom messages (`turn_end`) vs buffer to run end ✔ pi user `!` bash messages (flushed when the run settles).
- Alternative: inject via steering queue (becomes model input at next boundary) — pi does this only when `triggerTurn ≠ false`.
- Attach to next user prompt (`deliverAs: "nextTurn"`) ✔ pi.
- Repair at replay instead (provider boundary holds back mid-transcript system messages until results are emitted, pi `transformMessages`) — complementary, see [[transcript-replay-repair]].
- Excluded-from-context variant: `!!cmd` runs without ever reaching the model ✔ pi.
- **One generic injection path: while a turn is active, out-of-band items go into the turn's pending-input queue (consumed at the next sampling boundary, like steering); when idle they are recorded directly without starting a turn** ✔ codex (`inject_no_new_turn`; user `!cmd` output, hook context).

## Implementations
- [[pi--out-of-band-message-deferral|pi]] — `_pendingCustomMessages` flushed at `turn_end`; `_pendingBashMessages` flushed when the run settles; `_pendingNextTurnMessages` attached to next prompt; delivery matrix in `sendCustomMessage`.
- [[codex--out-of-band-message-deferral|codex]] — `inject_no_new_turn`: pending turn input if a turn is running, else record without a new turn; user shell output as `<user_shell_command>` fragment.

## Failures
- [[side-channel-message-splits-tool-pair]]
- Cross-group: [[side-task-result-invisible-to-parent]] (09-subagents)

## Tradeoffs
- [[mid-run-user-input]]

## Related
[[steering-queue]] · [[follow-up-queue]] · [[transcript-replay-repair]] · [[message-conversion-layer]] · [[run-settlement]] · [[extension-event-hooks]] · [[user-shell-escape]]
