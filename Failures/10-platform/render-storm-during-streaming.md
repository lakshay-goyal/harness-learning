---
type: failure
concepts: [differential-tui-rendering]
harnesses: [pi]
---
**Symptom** — During fast token streaming the TUI rendered on every event, burning CPU and delaying input handling (input latency, notably on Windows — `packages/tui/CHANGELOG.md:225`).

**Root cause** — `requestRender()` triggered an immediate render per event with no frame budget.

**Fix · [[pi]]** — `6f5f37f85` 2026-04-06: renders coalesced via `process.nextTick` and throttled to `MIN_RENDER_INTERVAL_MS = 16` (~60 fps); user input preempts the throttle with an immediate render; `force` bypasses (`packages/tui/src/tui.ts:503-507,990-1043`).

**Lesson** — Streaming UIs need a frame budget with input priority; render at display rate, not event rate.

Related: [[differential-tui-rendering]] · [[agent-event-stream]] · [[pi--differential-tui-rendering|pi]]
