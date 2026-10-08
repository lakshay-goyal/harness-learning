---
type: implementation
harness: codex
concept: session-export-share
commit: 622e9e3696
files: [codex-rs/tui/src/slash_command.rs:109, codex-rs/tui/src/app/transcript_export.rs:1, codex-rs/tui/src/app/transcript_export.rs:227, codex-rs/tui/src/app/transcript_export.rs:297, codex-rs/feedback/src/lib.rs:53]
---
[[session-export-share]] in [[codex]].

Export only (no share/hosting): TUI `/export` writes a Markdown transcript; the raw rollout path is the other "export".

## Mechanism
- `/export` — "export the conversation as markdown" (`codex-rs/tui/src/slash_command.rs:109`); "Complete, Markdown-preserving conversation exports" loaded through the app server (`ThreadItem`/`Turn`, history hydration) (`codex-rs/tui/src/app/transcript_export.rs:1-30`); visible items filtered (`visible_export_items`, `:202`), rendered with the same history cells as the TUI (`render_markdown_transcript`, `:227`), written relative to cwd (`write_transcript`, `:297`); destination picker + filename prompt in the composer.
- Related TUI commands: `/copy` "copy the last response or part of it" (`codex-rs/tui/src/slash_command.rs:108`), `/raw` "toggle raw scrollback mode for copy-friendly terminal selection" (`:110`), `/rollout` "print the rollout file path" (`:156`).
- Full-fidelity artifact = the rollout JSONL itself ([[codex--session-tree|session-tree]]); `/feedback` uploads it as a Sentry attachment ([[codex--install-telemetry|install-telemetry]]).
- No HTML viewer, no gist/hosted share link found (unverified beyond the slash-command list).

## Evolution
- 2026-08-07 `2801d12661` (#37358) "Add Markdown conversation export to the TUI".

## Versus pi
- [[pi--session-export-share]]: pi exports self-contained HTML (with plugin renderers) or JSONL and shares via hosted artifact / private gist; codex exports plain Markdown locally and has no share flow — smaller XSS surface ([[exported-session-xss]] does not apply).
