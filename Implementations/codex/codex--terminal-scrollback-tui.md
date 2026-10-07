---
type: implementation
harness: codex
concept: terminal-scrollback-tui
commit: 622e9e3696
files: [codex-rs/tui/src/insert_history.rs:1, codex-rs/tui/src/custom_terminal.rs:1, codex-rs/Cargo.toml:436, codex-rs/tui/src/cli.rs:76, codex-rs/tui/src/app_backtrack.rs:168, codex-rs/tui/src/slash_command.rs:109, codex-rs/tui/src/app.rs:1193]
---
[[terminal-scrollback-tui]] in [[codex]].

## Mechanism
- "Codex uses the terminal scrollback itself for finalized chat history, so inserting a history cell is an escape-sequence operation rather than a normal ratatui render. Untrusted content follows ratatui's control-character filtering before semantic hyperlinks add trusted escapes." (`codex-rs/tui/src/insert_history.rs:1-5`).
- `codex-rs/tui/src/custom_terminal.rs` is derived from `ratatui::Terminal` (MIT header, `:1-6`); ratatui pinned `0.30.2` (`codex-rs/Cargo.toml:436`). Inline viewport restored when not in alt screen (`codex-rs/tui/src/app.rs:1193`).
- **Modes**: `--no-alt-screen` "Runs the TUI in inline mode, preserving terminal scrollback history" (`codex-rs/tui/src/cli.rs:76-80`); `/tui` picker Scrollback vs Fullscreen saves `tui.fullscreen_transcript`, restart required (`2f34d236f5` 2026-09-20; `codex-rs/tui/src/slash_command.rs:111`); transcript overlay enters the alternate screen (`codex-rs/tui/src/app_backtrack.rs:168`, leaves `:196`). `/raw` toggles raw scrollback mode for copy-friendly selection (`codex-rs/tui/src/slash_command.rs:110`). Which mode is the default: unverified.
- **Structure**: `app.rs` (event loop, an app-server client since `9dba7337f2`), `chatwidget.rs` + `chatwidget/` (transcript cells), `bottom_pane/` (chat_composer, approval_overlay, request_user_input, file_search_popup, command_popup, footer, mcp_server_elicitation, pending_thread_approvals), `history_cell/`, `markdown_render.rs` / `markdown_stream.rs`, `multi_agents.rs` ("/subagents picker entries, and the fast-switch keyboard shortcuts"), `resume_picker.rs`, `pager_overlay.rs`, `keymap/`. Only in-repo `AGENTS.md` is `codex-rs/tui/src/bottom_pane/AGENTS.md`.
- **Slash commands** (`SlashCommand`, `#[strum(serialize_all = "kebab-case")]`, `codex-rs/tui/src/slash_command.rs:11`; descriptions `:91-160`): model, daybreak, ide, permissions, keymap, vim, `setup-default-sandbox` (variant `ElevateSandbox`, "set up elevated agent sandbox"), experimental, `approve` (variant `AutoReview`, "approve one retry of a recent auto-review denial"), memories, skills, import, hooks, review, rename, new, archive, delete, resume, fork, worktree, app ("continue this session in the Desktop app", macOS/Windows only `:310`), init, compact, recap, plan, voice, goal, agents ("open the agent command center"), side / btw ("start a side conversation in an ephemeral fork"), copy, export, raw, tui, diff, mention, status, daemon, warnings, cd, pwd (alias `cwd`), usage, debug-config ("show config layers and requirement sources"), title, statusline, theme, pets (alias `pet`), mcp, apps, plugins, logout, quit, exit, feedback, rollout, ps, stop (alias `clean`), clear, test-approval, `subagents` (variant `MultiAgents`), `debug-m-drop` / `debug-m-update` ("DO NOT USE"). `rollout` and `test-approval` visible only in debug builds (`:312`). `docs/slash_commands.md` is a 3-line stub.
- File @-mention picker uses `codex-rs/file-search` (fuzzy filename search, born `296996d74e` 2025-06-25) — not a code index ([[no-codebase-index]]).

## Constants
| name | value | path:line |
|---|---|---|
| ratatui | 0.30.2 | codex-rs/Cargo.toml:436 |
| recap history turns (display) | 8 | codex-rs/tui/src/app/recap_history.rs:13 |

## Evolution
- 2025-04-16 Era 0: Ink TUI in the TypeScript CLI; 2025-04-28 `cca1122ddc` Rust TUI becomes default.
- 2025-12-09 `0c8828c5e2` tui2 alternative frontend → retired `a489b64cb5` 2026-01-21.
- 2026-03-13 `9dba7337f2` / 2026-03-16 `db89b73a9c` TUI on top of app-server; legacy split removed `d65deec617` 2026-03-27.
- 2026-08-07 `2801d12661` `/export` Markdown.
- 2026-09-20 `2f34d236f5` `/tui` mode picker.

## Versus pi
- [[pi--differential-tui-rendering]]: pi line-diffs every component frame (CSI 2026, 16 ms throttle) and since 2026-10-01 defaults to an alt-screen app-owned transcript; codex writes finalized history once into native scrollback and diffs only the small live viewport. See [[tui-rendering-strategy]].
