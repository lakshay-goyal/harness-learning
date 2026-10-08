---
type: implementation
harness: pi
concept: differential-tui-rendering
commit: b30a6dd77
files: [packages/tui/src/tui.ts:449-507, packages/tui/src/tui.ts:990-1043, packages/tui/src/tui-main-screen.ts:9-18, packages/tui/src/tui-main-screen.ts:255-612, packages/tui/src/tui-alt-screen.ts:76-81, packages/tui/src/terminal.ts:8-13, packages/tui/src/terminal.ts:115-116, packages/tui/src/terminal.ts:161-181, packages/coding-agent/src/core/settings-manager.ts:1372]
---
[[differential-tui-rendering]] in [[pi]].

## Mechanism
- Package `@earendil-works/pi-tui`: "Minimal terminal UI framework with differential rendering and synchronized output for flicker-free interactive CLI applications" (`packages/tui/README.md:3`). Home-grown from day 2 (`afa807b20`); why not Ink/blessed is not documented (unverified).
- Component = `render(width) → string[]` lines + optional input handling + `invalidate()` (`packages/coding-agent/docs/tui.md:22-28`); built-in components `docs/tui.md:34-40`. `InteractiveMode` is 7082 lines (`modes/interactive/interactive-mode.ts`).
- Two interchangeable renderers behind one `TUI` interface: `TuiMainScreen` (main screen + native scrollback) and `TuiAltScreen` (alternate screen, app-owned scrolling, mouse, search, scrollbar) (`packages/tui/src/tui.ts:453` `interface TUI`; `tui-main-screen.ts:124`, `tui-alt-screen.ts:202`; `tui/README.md:7-8`; `c13ffe187` #7304). Default `tuiMode:"fullscreen"` (`settings-manager.ts:1372`; `88ff80b98` 2026-10-01); `"regular"` keeps scrollback (`docs/settings.md:94`, `docs/usage.md:86`).
- Main-screen algorithm (`tui-main-screen.ts:255-612`): render all components → composite overlays → extract `CURSOR_MARKER` → per-line reset.
  - **Full redraw** (clear screen + `\x1b[3J` scrollback) on first render, width change (re-wrap), height change (except Termux), clear-on-shrink, or when the first changed line is above the previous viewport (scrolled-off rows can't be touched) (`:331-358,449-455`).
  - Otherwise compute `firstChanged..lastChanged`, move cursor, rewrite only that range (`:363-390,487-490` — "reduces flicker when only a single line changes (e.g., spinner animation)").
- Every write wrapped in synchronized output `CSI ?2026h … ?2026l` so terminals paint atomically (`tui-main-screen.ts:280,302,460,567`; `97c730c87`).
- Output streamed through `BoundedTerminalWriter` in 1 MiB chunks (`MAX_RENDER_WRITE_CHARS`) to avoid V8 max-string crashes on image-heavy frames (`tui-main-screen.ts:9-18`; `6c4f36026`, #8028).
- Scheduling: `requestRender()` coalesces via `process.nextTick`, throttled to `MIN_RENDER_INTERVAL_MS = 16` (~60 fps); user input preempts the throttle; `force` bypasses (`packages/tui/src/tui.ts:507,998-1031`; `6f5f37f85`).
- Other: Kitty/iTerm2 inline images, IME hardware-cursor placement via `CURSOR_MARKER`, CSS-like overlay positioning (`a4ccff382`), bracketed paste, diagnostics `PI_TUI_WRITE_LOG` / `PI_TUI_DEBUG_REDRAW` (`docs/tui.md:48,112`; `tui-main-screen.ts:321-327`).
- Terminal lifecycle: stdin paused before raw-mode restore (`packages/tui/src/terminal.ts:466-474`; `f431f6260`), Kitty key-release events drained before exit (`drainInput`, `packages/tui/src/terminal.ts:79-84,391`; `9a4d043b2`), `session_shutdown` on SIGTERM/SIGHUP, dead-terminal EIO not a crash (`4c6b724ea`).
- Keybindings: `~/.pi/agent/keybindings.json` maps namespaced ids (`app.session.new`, `tui.input.submit`) to key or list; `[]` disables; applied on `/reload` (`docs/keybindings.md:1-27`); syntax `ctrl|shift|alt|super + key`, `super` needs Kitty protocol (`:31-41`); legacy camelCase auto-migrated (`core/keybindings.ts:300-340`; `e3fee7a51`, #2391); Windows/WSL defaults differ to avoid terminal-reserved shortcuts (`packages/coding-agent/src/core/keybindings.ts:62`; `packages/coding-agent/CHANGELOG.md:630`, #8372); `KeybindingsManager` extends pi-tui's (`packages/coding-agent/src/core/keybindings.ts:371`). Configurable since #405 (`packages/coding-agent/CHANGELOG.md:4857`).
- **Input pipeline** (stdin → one event per `handleInput`): `StdinBuffer` (adapted from OpenTUI, MIT) reassembles split CSI/OSC/DCS/APC/SGR-mouse sequences and bracketed paste before dispatch (`packages/tui/src/stdin-buffer.ts:1-26,31-194`; `f3b7b0b17` — keys were dropped when batched with Kitty release events over SSH, #538). Lone-ESC wait separated from incomplete-sequence wait so SSH/`PI_TUI_ESC_TIMEOUT` can rejoin split Alt+Enter without delaying Escape (`terminal.ts:120-130`; `06ed87167` #7899, `2a95ef70d`).
- **Keyboard protocol negotiation**: one burst `CSI >7u` + `CSI ?u` + DA1; a `?<flags>u` reply enables Kitty protocol, DA1 arriving first falls back to xterm `modifyOtherKeys` — no startup timeout (`terminal.ts:12-14,246-290`). Key matching parses Kitty CSI-u, modifyOtherKeys and legacy sequences, masks Caps/Num Lock (`keys.ts:299,587-713`); release/repeat detection `keys.ts:527-560`. Apple Terminal / Windows can't encode Shift+Enter → a native helper reads live Shift state when `\r` arrives (`terminal.ts:352-361`; `c5181a266`, `73dd066ee`).
- **Native helpers** (prebuilt N-API `.node` for darwin/linux-x11/win32 arm64+x64, `packages/tui/native/*/prebuilds/`): async clipboard text/image/file-paths, modifier-key state, Windows VT-input enable (`native-platform.ts:9-23,55-61`); replaced the koffi FFI dep (`4868222e3`, #4480) and the external clipboard dep, running on worker threads (`caf6dfe73`, #9163). Linux keeps CLI tools for `setText` to retain clipboard ownership (`native-platform.ts:16`).
- **Terminal capability detection** (`terminal-image.ts:70-166`): env sniffing (`KITTY_WINDOW_ID`, Ghostty, WezTerm → Kitty graphics; `ITERM_SESSION_ID` → iTerm2); under tmux images off and OSC 8 hyperlinks only if `tmux display-message '#{client_termfeatures}'` confirms forwarding (`:54-81`); overrides `PI_IMAGE_PROTOCOL=kitty|iterm2|none`, `PI_TRUE_COLOR`, `PI_HYPERLINKS` (`:146-165`). Kitty images get ids, placement metadata and are clipped to layout boxes (`:204-422`; `af187eee4`).
- **Width math**: `visibleWidth` uses `Intl.Segmenter` graphemes + `get-east-asian-width`, with an RGI-emoji heuristic so streamed partial emoji still count 2 cells (`utils.ts:1,22-34,179-250`); 512-entry width cache (`utils.ts:51`).
- **Editor component** (`components/editor.ts`, 2472 lines): pastes >10 lines or >1000 chars collapse to atomic `[paste #N +L lines]` markers expanded on submit (`:30-38,1296-1312,1091-1093`); bracketed-paste buffering (`:719-736`); prompt history 100 entries with draft restore (`:344-347,424-489`); Emacs kill ring with accumulate/yank-pop (`kill-ring.ts:1-40`) and structuredClone undo stack incl. paste registry (`undo-stack.ts:1-28`, `editor.ts:224-228`; `4c2d78f6c` #1373); `@`/`#` autocomplete triggers debounced 20 ms (`:252-253`).
- **Autocomplete**: slash commands + file paths; file search shells out to `fd` (respects .gitignore, 100 results) and ranks with subsequence `fuzzyMatch` (lower score better) (`autocomplete.ts:148-194,764-802`; `fuzzy.ts:1-15`).
- **Markdown + LaTeX**: Markdown component tokenizes `$…$`, `$$…$$`, `\[…\]` incl. *pending* (still-streaming) math, rendered by a 1506-line Unicode LaTeX renderer (Greek/symbol tables, matrices) (`components/markdown.ts:26-114`; `latex.ts:1-30`; `05e89b418` 2026-08-05, then `aa601d7ba` `452923b54` `534bcbffb` `fa0e1f48a`).
- **Alt-screen extras**: layout tree (`VStack`/`HStack`/`ScrollView`/`Stack`, `layout.ts`, `ea1e77e2d`), draggable transient scrollbars (`8ac92f831`, `6129a353b`), reusable `MouseRegion` (`71026970a`), transcript search (`alt-screen-search.ts`; `00121ed99` #7913, linearised in `2d4116333`). Assistant messages wrapped in OSC 133 A/B/C shell-integration zones (`packages/coding-agent/src/modes/interactive/components/assistant-message.ts:7-9,86-87`; `4cb1a56b5` #1805); alt-screen jumps between turns by scanning for `133;A` (`tui-alt-screen.ts:74-75,505-515`; `3c717842e`).
- **System theme**: queries OSC 10/11/4 (fg/bg/ANSI palette) in one burst ended by DA1, starts grayscale and recolors when replies land; tokens placed by OKLab lightness + WCAG2 contrast; themes accept `okhsl()`; mode-2031 notifications re-query (`oklab.ts`, `colors.ts`; `bf8e4b953` #10067, 2026-09-26).
- Terminal title via OSC 0, indeterminate progress OSC 9;4;3 kept alive every 1 s, cleared with 9;4;0 (`terminal.ts:8-10,527-550`).

## Component inventory (pi-tui, remaining modules)
| Module | Role | Evidence |
|---|---|---|
| `AltScreenFlashContainer` | transient messages composited by alt-screen renderer | `packages/tui/src/components/alt-screen-flash.ts:12-13` |
| `CancellableLoader` | loader that calls back on Escape (cancel long ops) | `components/cancellable-loader.ts:13-16` |
| `HStack` / `VStack` (`Stack`) | horizontal / vertical layout containers | `components/h-stack.ts:5`, `components/v-stack.ts:3` |
| `LAYOUT_NODE` symbol | marks layout-aware nodes, `Symbol.for` so duplicate package copies agree | `packages/tui/src/layout-node.ts:3` |
| `MouseRegion` | adds mouse handling to a component without changing rendering | `components/mouse-region.ts:11-12` |
| `ScrollView` | scrollable viewport with follow-end option | `components/scroll-view.ts:6-18` |
| `SelectList` / `SettingsList` | filterable pickers (selection resets on filter change) / settings rows | `components/select-list.ts:12,63`, `components/settings-list.ts:7-8` |
| `Spacer`, `TruncatedText` | stateless primitives (no render cache) | `components/spacer.ts:6,18`, `components/truncated-text.ts:7,19` |
| `EditorComponent` | interface custom editors implement (extension `setEditorComponent`, e.g. modal-editor example) | `packages/tui/src/editor-component.ts:11` |
| `isNativeModifierPressed` + native-module-path | native addon probe for physical modifier keys; standalone binaries resolve addon without installed package | `native-modifiers.ts:5`, `native-module-path.ts:8-22` |
| terminal-colors | parse colors the terminal reports for its theme (light/dark detection) | `terminal-colors.ts:1-9` |
| word-navigation | word-boundary cursor moves with pluggable segmenter (Intl) | `word-navigation.ts:9-10` |
- **Render debugging**: hidden `/debug` (not in the slash-command list, also bound via `ui.onDebug`) writes every rendered line with its computed visible width (`[idx] (w=N) "<json-escaped line>"`), the terminal size, and all agent messages as JSONL to `~/.pi/agent/pi-debug.log` (`modes/interactive/interactive-mode.ts:3087,3300-3304,6914-6941`; path `src/config.ts:654-656`) — the width column is what diagnoses wide-char/ANSI overflow bugs. Startup profiling: `PI_TIMING=1` prints per-phase startup timings to stderr (`core/timings.ts:1-50`); `PI_STARTUP_BENCHMARK=1` (interactive only) initializes the TUI, waits 150 ms for terminal query replies (Kitty keyboard, device attributes, cell size), then stops and prints timings (`src/main.ts:932-975`).

## Constants
| name | value | path:line |
|---|---|---|
| `MIN_RENDER_INTERVAL_MS` | 16 ms (~60 fps) | `packages/tui/src/tui.ts:507` |
| `MAX_RENDER_WRITE_CHARS` | 1 MiB | `packages/tui/src/tui-main-screen.ts:9` |
| `DEFAULT_ESCAPE_TIMEOUT_MS` | 10 ms (100 over SSH) | `packages/tui/src/terminal.ts:115-116` |
| stdin `DEFAULT_SEQUENCE_TIMEOUT_MS` / escape | 50 / 10 ms | `packages/tui/src/stdin-buffer.ts:23-24` |
| Kitty flags / fragment timeout | 7 / 150 ms | `packages/tui/src/terminal.ts:12-13` |
| `TERMINAL_PROGRESS_KEEPALIVE_MS` | 1000 | `packages/tui/src/terminal.ts:8` |
| theme query timeout | 100 ms | `coding-agent/src/modes/interactive/theme/theme-controller.ts:25` |
| loader spinner interval | 80 ms | `packages/tui/src/components/loader.ts:12` |
| wheel scroll burst/gesture/reference gap, max auto lines | 5 / 200 / 100 ms, 6 | `packages/tui/src/wheel-scroll.ts:6-11` |
| alt-screen page overlap / wheel multiplier / double-click | 4 / 5 / 500 ms | `packages/tui/src/tui-alt-screen.ts:76-81` |
| offscreen Kitty image cache | 16 imgs / 32 MiB tx / 64 MiB decoded | `packages/tui/src/tui-alt-screen.ts:78-80` |
| width cache | 512 | `packages/tui/src/utils.ts:51` |
| inline image width | 60 cells | `coding-agent/src/core/settings-manager.ts:59` |
| autocomplete visible | 5 (3–20) | `coding-agent/src/core/settings-manager.ts:1522` |
| `tuiMode` default | `"fullscreen"` | `coding-agent/src/core/settings-manager.ts:1372` |
| clipboard image list / PowerShell timeout | 1 s / 5 s | `coding-agent/src/utils/clipboard-image.ts:19-20` |
| large-paste marker threshold | >10 lines or >1000 chars | `packages/tui/src/components/editor.ts:1302` |
| editor prompt history | 100 entries | `packages/tui/src/components/editor.ts:434` |
| attachment autocomplete debounce / triggers | 20 ms / `@` `#` | `packages/tui/src/components/editor.ts:252-253` |
## Evolution
- 2025-08-10 `afa807b20` double-buffer differential rendering + Terminal abstraction + xterm-headless `VirtualTerminal` for tests (flicker; testability).
- 2025-08-11 `386f90fc3` "surgical" 3-strategy diff (surgical/partial/full): ~14 → 1.3 lines per update.
- 2025-11-10 `97c730c87` minimal TUI rewrite, CSI 2026 synchronized output.
- 2025-11-11 `fb4893d8d` clear scrollback on full re-render (stale duplicated history).
- 2026-01-12 `a4ccff382` overlay positioning API.
- 2026-02-02/03 `f431f6260`, `9a4d043b2` SSH session close / Kitty release leak on exit (#1185, #1204).
- 2026-03-20 `16937947b` skip Termux height redraws (#2467); `e3fee7a51` namespaced keybindings (#2391).
- 2026-04-06 `6f5f37f85` 16 ms render throttle under streaming load.
- 2026-07-30 `c13ffe187` alternate-screen renderer (#7304); 2026-08-02 `b70c0f5b4` revert of switchable terminal renderers (#7473).
- 2026-08-11 `06ed87167` lone-ESC timeout tuning (#7899).
- 2026-08-26 `6c4f36026` 1 MiB chunked render writes (#8028).
- 2026-10-01 `88ff80b98` fullscreen default; 2026-10-05 `4c6b724ea` dead-terminal EIO not reported as crash.

## Evidence commits
`afa807b20` `386f90fc3` `97c730c87` `fb4893d8d` `a4ccff382` `f431f6260` `9a4d043b2` `16937947b` `6f5f37f85` `c13ffe187` `b70c0f5b4` `06ed87167` `6c4f36026` `88ff80b98` `4c6b724ea` `e3fee7a51` `f3b7b0b17` `2a95ef70d` `bdb416cbc` `7f30fb613` `c5181a266` `73dd066ee` `4868222e3` `caf6dfe73` `4c2d78f6c` `05e89b418` `00121ed99` `2d4116333` `ea1e77e2d` `8ac92f831` `71026970a` `4cb1a56b5` `3c717842e` `bf8e4b953` `af187eee4`

## Quirks
- TUI fixes are the second-largest fix theme (86 fix commits + 244 `fix(tui)`-scoped; 203 changelog bullets) per fix-mining census.
- Only the active theme file is fs-watched; everything else reloads via `/reload`.
- Custom tool renderers double as HTML export renderers (ANSI → HTML) → [[pi--session-export-share]].

## Failures
[[full-redraw-replays-history]] · [[render-output-exceeds-string-limit]] · [[render-storm-during-streaming]] · [[terminal-state-leaks-on-exit]] · [[terminal-input-sequence-fragmentation]]
