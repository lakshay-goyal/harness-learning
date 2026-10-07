---
type: implementation
harness: codex
concept: cross-session-prompt-history
commit: 622e9e3696
files: [codex-rs/message-history/src/lib.rs:1, codex-rs/message-history/src/lib.rs:51, codex-rs/message-history/src/lib.rs:89, codex-rs/message-history/src/lib.rs:191, codex-rs/message-history/src/lib.rs:278]
---
[[cross-session-prompt-history]] in [[codex]].

## Mechanism
- File `~/.codex/history.jsonl`, one `{"session_id","ts","text"}` per line (`codex-rs/message-history/src/lib.rs:1-15`).
- **Atomic append**: single `write(2)` with `O_APPEND` (atomic up to `PIPE_BUF`) plus advisory `File::try_lock` with 10 retries × 100 ms (`codex-rs/message-history/src/lib.rs:51-59`, `:89-103`).
- **Cap**: when over `history.max_bytes`, drop oldest lines down to 80% "to avoid trimming again immediately on the next write" (`codex-rs/message-history/src/lib.rs:55-56`, `:191-194`).
- **Paging identity**: inode (Unix) / creation time (Windows) + newline count (`codex-rs/message-history/src/lib.rs:278-284`).
- Disabled with `history.persistence = none`.

## Constants
| name | value | path:line |
|---|---|---|
| `HISTORY_SOFT_CAP_RATIO` | 0.8 | codex-rs/message-history/src/lib.rs:56 |
| `MAX_RETRIES` / `RETRY_SLEEP` | 10 / 100 ms | codex-rs/message-history/src/lib.rs:58-59 |

