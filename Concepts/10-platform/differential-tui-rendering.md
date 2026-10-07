---
type: concept
stage: architecture
tier: candidate
aliases: ["pi-tui", "TuiMainScreen", "TuiAltScreen", "CSI 2026", "synchronized output", "tuiMode fullscreen", "alt-screen-transcript", "MIN_RENDER_INTERVAL_MS"]
harnesses: [pi]
---
Terminal UI that re-renders components to lines, diffs against the previous frame and rewrites only changed rows inside synchronized-output brackets, with a fallback full redraw and an optional alternate-screen transcript with app-owned scrolling.

## Why
- Streaming tokens + spinners at 60 fps flicker and burn CPU if every frame is a full repaint.
- Native scrollback can't be edited once scrolled off — rows above the viewport force full redraws that can replay history ([[full-redraw-replays-history]]).
- Large image-heavy frames exceed JS string limits ([[render-output-exceeds-string-limit]]); unthrottled render requests storm the terminal ([[render-storm-during-streaming]]).
- The agent owns the terminal and must restore it on every exit path ([[terminal-state-leaks-on-exit]]).

## Design space
- **Framework**: React-for-terminal (Ink) / blessed vs home-grown line-component framework (pi; rationale not documented, unverified).
- **Diff granularity**: full repaint vs changed-line range (pi) vs cell-level.
- **Atomicity**: none vs DEC mode 2026 synchronized output (pi).
- **Screen**: main screen + native scrollback (pi `tuiMode:"regular"`) vs alternate screen with app-owned scroll/search/mouse (pi default since 2026-10-01).
- **Scheduling**: render on every event vs coalesced + throttled frame budget with input preemption (pi 16 ms).
- **Output size**: single write vs bounded chunked writer (pi 1 MiB).
- **Testability**: headless virtual terminal (pi xterm-headless `VirtualTerminal`).
- codex: different strategy — finalized history inserted once into native terminal scrollback via escape sequences, only the small inline live area diffed (ratatui cell buffer); see [[terminal-scrollback-tui]] / [[codex--terminal-scrollback-tui|codex]]. Fullscreen/alt-screen transcript is an opt-in mode (`/tui`, `--no-alt-screen`).

## Implementations
- [[pi--differential-tui-rendering|pi]] — `@earendil-works/pi-tui`: line-diff main-screen renderer + alt-screen renderer behind one `TUI` interface, CSI 2026, 16 ms throttle, 1 MiB chunked writes.

## Failures
- [[full-redraw-replays-history]]
- [[render-output-exceeds-string-limit]]
- [[render-storm-during-streaming]]
- [[terminal-state-leaks-on-exit]]
- [[terminal-input-sequence-fragmentation]] (10-platform) — Keystrokes dropped or misread: over SSH, key presses batched with Kitty release events in one stdin chunk…

## Related
[[extension-ui-primitives]] · [[agent-event-stream]] · [[layered-settings]] · [[session-export-share]] · [[tui-rendering-strategy]]
