---
type: failure
concepts: [extension-event-hooks, turn-lifecycle-hooks]
harnesses: [pi, codex]
---
**Symptom** — Hook handlers that legitimately waited (an LLM call inside a hook, a confirmation dialog waiting on the human) were killed by the hook timeout, so plugins failed mid-action.

**Root cause** — The original hooks system (`04d59f31e`, 2025-12-09) imposed a `hookTimeout` on every handler; it was applied inconsistently and cannot distinguish "stuck" from "waiting on a human/model".

**Fix · [[pi]]** — `88e39471e` 2025-12-31 "Remove hook execution timeouts": no wall-clock limit; user aborts with Ctrl+C/Esc instead. Corollary: `675756154` 2026-07-02 tried to make stuck `context` hooks abortable (#6234) and was reverted 5 days later by `2b00dade7` 2026-07-07 (reason not stated — unverified). At HEAD handlers are fully sequential and awaited, so a stuck hook still stalls the run (`packages/coding-agent/docs/extensions.md:109-111`).

**Fix · [[codex]]** — *Variant: the timeout did not cover the phase that blocked.* Command hooks were given input on stdin before their output was drained; with large input and a hook that never reads stdin, pipe buffers filled and the turn hung indefinitely, and stdin writes were outside the hook timeout. `885113aa1d` 2026-09-09 "Prevent command hooks from hanging on blocked stdin (#44288)": write stdin concurrently with draining stdout/stderr and apply the timeout to both (`codex-rs/hooks/src/engine/command_runner.rs`). Codex keeps wall-clock limits — default 600 s per command hook (`codex-rs/hooks/src/engine/discovery.rs:764`), SessionEnd 1 s default / 3 s max (`codex-rs/hooks/src/events/session_end.rs:20-23`), outcome "hook timed out after {n}s" (`codex-rs/hooks/src/engine/command_runner.rs:276-318`) — because its hooks are external processes, not in-process code awaiting a human.

**Lesson** — Pick one coherent liveness policy per hook kind: in-process hooks that may await humans/models get no wall clock but a reliable user abort (pi); subprocess hooks get a timeout that covers *every* I/O phase, with stdin written concurrently with draining output (codex) — a timeout that misses a blocking phase is no timeout.

Related: [[extension-event-hooks]] · [[turn-lifecycle-hooks]] · [[abort-propagation]] · [[pi--extension-event-hooks|pi]] · [[codex--extension-event-hooks|codex]] · [[codex--turn-lifecycle-hooks|codex]]
