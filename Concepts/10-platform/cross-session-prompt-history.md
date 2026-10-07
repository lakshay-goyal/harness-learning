---
type: concept
stage: state
tier: candidate
aliases: [message-history, history.jsonl, HistoryPersistence, append_entry, history.max_bytes, prompt-recall-history]
harnesses: [codex]
---
A global append-only file of every user prompt submitted across all sessions, used for shell-like Up-arrow recall in the composer, independent of conversation transcripts.

## Why
- Users re-run or tweak earlier prompts from other sessions; transcripts are per session and expensive to scan.
- Multiple concurrent harness processes append to one file — writes must not interleave or corrupt.
- Unbounded growth needs a cap that does not trim on every write.

## Design space
- **Source**: derive from session transcripts vs **dedicated prompt-only file** (codex `~/.codex/history.jsonl`).
- **Concurrency**: single `O_APPEND` write ≤ `PIPE_BUF` + advisory lock with retries (codex) vs lockfile vs DB.
- **Cap**: none vs byte cap with soft-trim to a ratio (codex 80%) vs entry count.
- **Opt-out**: `history.persistence = none` (codex).
- **Paging identity**: inode/creation-time + line count (codex).

## Implementations
- [[codex--cross-session-prompt-history|codex]] — `codex-rs/message-history`: `{"session_id","ts","text"}` lines, O_APPEND + `try_lock` 10×100 ms, trim oldest to 80% of `max_bytes`.

## Failures
- (none mined)

## Related
[[session-tree]] · [[sqlite-session-index]] · [[layered-settings]] · [[terminal-scrollback-tui]]
