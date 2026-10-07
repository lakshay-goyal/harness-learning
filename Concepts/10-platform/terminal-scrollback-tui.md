---
type: concept
stage: architecture
tier: candidate
aliases: [codex-tui, insert_history, custom_terminal, history_cell, bottom_pane, chatwidget, native-scrollback-history, inline-viewport-tui, fullscreen_transcript, --no-alt-screen]
harnesses: [codex]
---
Terminal UI that keeps only the live area (streaming cell, composer, popups) in an inline viewport and pushes each finalized history cell into the terminal's own scrollback via escape sequences, so history is never re-rendered by the app; a full-screen transcript is an opt-in overlay/mode.

## Why
- Re-rendering long transcripts each frame costs CPU and flickers; once text scrolls off the visible screen it cannot be edited anyway ([[full-redraw-replays-history]]).
- Native scrollback gives users the terminal's own search, selection, copy and scroll for free, and survives app exit.
- Cost: finalized cells are immutable (no re-wrap on resize, no later collapse/expand), and untrusted content written as raw escapes needs filtering.

## Design space
- **Where history lives**: app-owned buffer redrawn by diff ([[differential-tui-rendering]], pi) vs **terminal scrollback, append-only** (codex default inline mode) vs alternate screen with app-owned scrolling (pi default since 2026-10-01; codex "Fullscreen" mode / transcript overlay).
- **Live area rendering**: cell-buffer diff of a small inline viewport (codex: ratatui-derived `custom_terminal`) vs line diff (pi).
- **Mode switch**: per-launch flag/config (codex `--no-alt-screen`, `tui.fullscreen_transcript` via `/tui`, restart required) vs live toggle.
- **Escape safety**: filter control characters in untrusted content before adding trusted escapes (codex hyperlinks).
- **Copy ergonomics**: raw scrollback mode for copy-friendly selection (codex `/raw`).

## Implementations
- [[codex--terminal-scrollback-tui|codex]] — `codex-rs/tui` on ratatui 0.30.2 with a forked `Terminal`; `insert_history` writes finalized cells into scrollback; Scrollback vs Fullscreen modes; the TUI is an app-server client.

## Failures
- (none mined specific to the mechanism)

## Related
[[differential-tui-rendering]] · [[agent-event-stream]] · [[client-server-session-split]] · [[session-export-share]] · [[tui-rendering-strategy]]
