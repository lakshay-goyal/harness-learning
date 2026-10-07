---
type: failure
concepts: [differential-tui-rendering]
harnesses: [pi]
---
**Symptom** — Exiting pi corrupted the user's terminal or parent session: Ctrl+D closed the parent SSH session (#1185); Kitty key-release events leaked into the parent shell (#1204); SIGTERM/SIGHUP left the terminal in raw mode and skipped extension cleanup (#3212, #5724); dead-terminal `EIO` reported as a crash; SIGINT delivered while suspended (#1668).

**Root cause** — Terminal modes (raw mode, Kitty keyboard protocol) and stdin were restored in the wrong order or not on every exit path.

**Fix · [[pi]]**
- `f431f6260` 2026-02-02 — pause stdin before restoring raw mode (`packages/tui/src/terminal.ts:465-473`).
- `9a4d043b2` 2026-02-03 — drain Kitty release events before exit (`packages/tui/src/terminal.ts:79-84,391-440`).
- 0.67.3 / 0.77.0 — `session_shutdown` emitted on SIGTERM/SIGHUP (#3212); 0.79.4 keep handlers until cleanup (#5724); 0.55.2 (#1668) (hashes unverified).
- `4c6b724ea` 2026-10-05 — dead-terminal stdin errors not reported as crashes.

**Lesson** — An agent that owns the terminal owns its restoration on every exit path, including signals and dead terminals.

Related: [[differential-tui-rendering]] · [[extension-event-hooks]] · [[pi--differential-tui-rendering|pi]]
