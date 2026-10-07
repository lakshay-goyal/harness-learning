---
type: implementation
harness: codex
concept: shell-command-intent-parsing
commit: 622e9e3696
files: [codex-rs/shell-command/src/parse_command.rs:15-54, codex-rs/shell-command/src/parse_command.rs:2229-2240, codex-rs/shell-command/src/parse_command.rs:2290-2470, codex-rs/protocol/src/parse_command.rs:9-30, codex-rs/core/src/tools/events.rs:230, codex-rs/core/src/session/mod.rs:2776-2813, codex-rs/core/src/tasks/user_shell.rs:187, codex-rs/protocol/src/protocol.rs:3597-3654]
---
[[shell-command-intent-parsing]] in [[codex]].

## Mechanism
- `ParsedCommand` enum (`codex-rs/protocol/src/parse_command.rs:9-30`): `Read{cmd, name, path}` ("(Best effort) Path to the file being read"), `ListFiles{cmd, path?}`, `Search{cmd, query?, path?}`, `Unknown{cmd}`.
- `parse_command(command: &[String]) -> Vec<ParsedCommand>` (`codex-rs/shell-command/src/parse_command.rs:54`): one entry per meaningful pipeline/chain segment; doc: "Parses metadata out of an arbitrary command… slightly lossy due to the ~infinite expressiveness of an arbitrary command… The goal of the parsed metadata is to be able to provide the user with a human readable gist" (`:49-53`).
- Wrapper handling: `extract_shell_command` for `bash -lc` / `zsh -c` etc.; `tokenize_powershell_command` "preserving Windows paths and reader aliases" (`:15-44`).
- Recognizers in `summarize_main_tokens` (`:2290+`): `ls`/`eza`/`exa` → ListFiles; `rg`/`rga`/`ripgrep-all` → Search; `cat`/`Get-Content` → Read; `sed -n Np` and similar small formatters folded into reads (`is_small_formatting_command`, tests `:834-836`).
- Mutation guard: `xargs` subcommands `perl`/`ruby` with in-place flag, `sed -i`, `rg --replace` are mutating → not summarized as reads (`:2229-2240`).
- Consumers: `ExecCommandBeginEvent`/`ExecCommandEndEvent.parsed_cmd` (`codex-rs/protocol/src/protocol.rs:3597-3654`) built in `codex-rs/core/src/tools/events.rs:230`; approval requests (`request_command_approval`, `codex-rs/core/src/session/mod.rs:2776-2813`); user `!` commands (`codex-rs/core/src/tasks/user_shell.rs:187`); shell telemetry tagged by command category (`bb947e8e36` 2026-07-13).
- Prompt side: models are told to prefer `rg` / `rg --files` (`codex-rs/core/gpt_5_2_prompt.md:250`; restored after the GPT-5 rewrite dropped it, `90d892f4fd`) — the parser recognizes exactly the commands the prompt recommends.

## Evolution
- 2025-08-11 `7f6408720b` "[1/3] Parse exec commands and format them more nicely in the UI (#2095)".
- 2025-10-07..09 experimental `list_dir` / `grep_files` / `read_file` tools tried (`226215f36d`, `f52320be86`, `0026b12615`), then removed 2026-03-25 / 2026-05-05 (`14c35a16a8`, `178c3b15b4`, `70807730f5`) — parser remains the only source of read/search semantics ([[search-tools]], [[file-read-tool]]).
- 2026-02-10 `d44f4205fb` crate renamed `codex-command` → `codex-shell-command`.
- 2026-07-13 `bb947e8e36` telemetry by command category.

## Quirks
- Source comment: "DO NOT REVIEW THIS CODE BY HAND … The easiest way to iterate is to add unit tests and have Codex fix the implementation." (`codex-rs/shell-command/src/parse_command.rs:44-47`) — a ~2.5k-line agent-maintained parser.
- Parser output is display/telemetry metadata; command safety uses separate canonicalization/heuristics ([[approval-key-canonicalization]], [[dangerous-command-heuristics]]).

## Versus pi
pi gets intent from tool identity (`read`, `grep`, `find`, `ls` — [[pi--search-tools]], [[pi--file-read-tool]]) and only its renderers interpret bash. Axis: [[dedicated-vs-shell-tools]].
