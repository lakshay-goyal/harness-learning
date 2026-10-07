---
type: tradeoff
concepts: [differential-tui-rendering, terminal-scrollback-tui]
---
**Axis** — Who owns transcript history on screen: the app (re-rendered and diffed every frame, often on the alternate screen) vs the terminal (finalized history appended once to native scrollback; app renders only a live area).

| aspect | pi | codex |
|---|---|---|
| framework | home-grown line-component TUI `@earendil-works/pi-tui` | ratatui 0.30.2 with forked `Terminal` (`codex-rs/Cargo.toml:436`, `codex-rs/tui/src/custom_terminal.rs:1-6`) |
| history | re-rendered + line-diffed; `TuiAltScreen` app-owned scroll/search default since 2026-10-01 (`settings-manager.ts:1372`, `88ff80b98`) | inserted once into scrollback via escape sequences (`codex-rs/tui/src/insert_history.rs:1-5`) |
| live area | full component tree each frame, CSI 2026 synchronized output, 16 ms throttle (`packages/tui/src/tui.ts:507`) | small inline viewport, ratatui cell-buffer diff |
| alt screen | default `fullscreen`; `regular` keeps scrollback | opt-in: `/tui` Scrollback vs Fullscreen (`2f34d236f5`), transcript overlay (`codex-rs/tui/src/app_backtrack.rs:168`), `--no-alt-screen` (`codex-rs/tui/src/cli.rs:76-80`) |
| copy / search | app transcript search, mouse regions | terminal's native search/selection; `/raw` copy-friendly mode, `/copy` |
| failure surface | full redraw replays history ([[full-redraw-replays-history]]), render storms, string-limit overflow | finalized cells immutable (no re-wrap/collapse); untrusted escapes must be filtered before trusted hyperlinks |
| experiments | switchable renderers reverted (`b70c0f5b4`) | tui2 frontend retired (`0c8828c5e2` → `a489b64cb5`); TUI rebuilt on app-server (`db89b73a9c`) |

**When each wins**
- **App-owned diffed transcript (pi)**: rich interactive transcript — collapsible tool output, re-wrap on resize, in-app search, plugin-rendered components, mouse.
- **Native scrollback (codex)**: long sessions at low CPU, terminal-native copy/search, output survives exit and plays well with tmux/SSH; simpler renderer since history is write-once.
