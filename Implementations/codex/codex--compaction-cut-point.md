---
type: implementation
harness: codex
concept: compaction-cut-point
commit: 622e9e3696
files: [codex-rs/core/src/compact.rs:63, codex-rs/core/src/compact.rs:583, codex-rs/core/src/compact.rs:686, codex-rs/core/src/compact.rs:707, codex-rs/core/src/compact_remote_v2.rs:73, codex-rs/core/src/compact_remote_v2.rs:545, codex-rs/core/src/compact_remote_v2.rs:610, codex-rs/core/src/compact_remote_v2_images.rs:24, codex-rs/core/src/compact_remote_history.rs:39]
---
[[compaction-cut-point]] in [[codex]] — a keep-set *filter* rather than a positional cut.

## Mechanism
- **Local** (`codex-rs/core/src/compact.rs:686-764`): walk real user messages newest→oldest, approx-token count each; a message that does not fit is truncated to `TruncationPolicy::Tokens(remaining)` and becomes text-only (media dropped, "never clone discarded media"), then stop. Non-text-only messages are rebuilt as text, except text-only messages that fit keep their original parts (`:707-728`, `bd3d4d1436`). Prior summaries excluded by prefix match (`:583-603`). Everything else (assistant, reasoning, tool calls/outputs) dropped.
- **Remote v2** (`codex-rs/core/src/compact_remote_v2.rs:545-590`): retained = real user messages and hook prompts; client-authored developer messages only behind feature `RetainClientDeveloperMessages`; inter-agent AgentMessages except descendant progress (`MESSAGE` / `CHANNEL_POST`), `FINAL_ANSWER` completions, and messages > 10,000 tokens. Budget 64,000 tokens newest-first; the boundary message is middle-truncated by text; truncated retained messages `mark_retained_sources_incomplete()` (`:610-703`).
- **Images (remote)**: with `CompactionImageBudget` images are charged at the image estimate, kept atomic with their open/close label tags, and no older backfill happens once a boundary image does not fit (`codex-rs/core/src/compact_remote_v2_images.rs:24-98`). Image-resize notices stay attached to their message as a group (`codex-rs/core/src/compact_remote_history.rs:39-66`).
- Because tool calls are never kept, no call/result adjacency rule is needed; orphan repair still runs at prompt build (`for_prompt` → `normalize_history`, → [[transcript-replay-repair]]), and pair-aware deletion removes a dropped item's counterpart during compaction-overflow trimming (`codex-rs/core/src/context_manager/history.rs:680-692`).

## Constants
| name | value | path:line |
|---|---|---|
| `COMPACT_USER_MESSAGE_MAX_TOKENS` (local keep budget) | 20_000 | `codex-rs/core/src/compact.rs:63` |
| `RETAINED_MESSAGE_TOKEN_BUDGET` (remote) | 64_000 | `codex-rs/core/src/compact_remote_v2.rs:73` |
| `MAX_RETAINED_AGENT_MESSAGE_TOKENS` | 10_000 | `codex-rs/core/src/compact_remote_v2.rs:74` |

## Evolution
- 2025-09-12 `ea225df22e` all user messages quoted into one bridge message.
- 2025-09-22 `c415827ac2` (#4068) kept user messages truncated ("any future /compact task … fail" with a massive earlier message).
- 2025-10-31 `611e00c862` user messages re-inserted as real items.
- 2025-11-14 `0b28e72b66` previous summaries filtered out of the kept set.
- 2026-05-21 `94442b7f95` remote retained budget 64k.
- 2026-08-18 `711a5f8b3a`, 2026-09-22 `c117207a6f` drop descendant progress / channel posts.
- 2026-08-23 `6677fd827d` "Budget retained images during remote compaction (#40280)": "Remote compaction's retained-message budget counted text but not images, so image-heavy history could retain more context than the budget represented."
- 2026-09-25 `bd3d4d1436` (#48115) preserve original parts of kept text-only messages → [[compaction-loses-modality-or-structure]].

## Versus pi
- [[pi--compaction-cut-point]]: backward token walk over *all* message kinds, snapped to a legal non-toolResult boundary, split-turn handling. pi's 20000 keep-recent number was taken from codex `compact.rs`; codex applies it to user words only, so assistant/tool context survives only through the summary.
