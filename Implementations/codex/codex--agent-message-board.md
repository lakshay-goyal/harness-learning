---
type: implementation
harness: codex
concept: agent-message-board
commit: 622e9e3696
files: [codex-rs/ext/agent-message-board/src/tools/spec.rs:10, codex-rs/ext/agent-message-board/src/local.rs:52, codex-rs/ext/agent-message-board/src/tools.rs:39, codex-rs/core/src/agent_message_board.rs:1, codex-rs/core/src/agent_message_board.rs:204, codex-rs/agent-message-board-client/src/protocol.rs:22]
---
[[agent-message-board]] in [[codex]].

## Mechanism
- **Tools** (9, in the multi-agent namespace): create_channel, get_channels, list_threads, search_posts, read_thread, read_post, subscribe, unsubscribe, post (`codex-rs/ext/agent-message-board/src/tools/spec.rs:10-20`). Descriptions (`:25-141`): "Create a channel where all agents in this collaboration can read and post messages. You are subscribed to new top-level posts by default."; "Posting subscribes you to the thread unless you previously…"; agents addressed by "An absolute agent path or a reference relative to you"; paging "Opaque next_cursor… Concurrent posts may shift pages; omit the cursor to refresh."; defaults limit_chars 20000, max_chars_per_post 1000, max results 20, ≤ 256 recipients; overridable via `description_override` (`:131`).
- **Backends**: local SQLite `agent_message_board_1.sqlite` (tables channels / posts / subscriptions / subscription_opt_outs, idempotent `request_id`) (`codex-rs/ext/agent-message-board/src/local.rs:52-85`); in-memory (training/ephemeral); remote HTTP with bearer token from env var — "Runtime credentials are scoped to one session; tools never see them" (`codex-rs/agent-message-board-client/src/lib.rs:1-2`; `codex-rs/core/src/agent_message_board.rs:60-110`).
- **Gating**: `Feature::AgentMessageBoard` AND `Feature::MultiAgentV2`; ephemeral sessions never open local SQLite (`codex-rs/core/src/agent_message_board.rs:62-69`) → [[feature-flag-stages]].
- **Notifications**: `InterAgentCommunication{trigger_turn:false}` delivered only to the recipient's *current* turn — "This adapter never starts or restores recipients and never queues a notification for an idle agent" (`codex-rs/core/src/agent_message_board.rs:1-4`, `:204-262`) → [[subagent-result-mailbox]]. Remote notices > 1 024 bytes rejected as untrusted ("remote board notice exceeds the context budget"); post stays readable via tools (`:233-245`).

## Constants
| name | value | path:line |
|---|---|---|
| post / channel name / read caps | `MAX_POST_BYTES` 64 KiB / `MAX_CHANNEL_BYTES` 128 / `MAX_READ_CHARS` 20 000 | `codex-rs/ext/agent-message-board/src/local.rs:52-54` |
| tool response cap | `MAX_RESPONSE_BYTES` 8 000 | `codex-rs/ext/agent-message-board/src/tools.rs:39` |
| remote client body / response | 512 KiB / 1 MiB | `codex-rs/agent-message-board-client/src/protocol.rs:22-23` |
| remote notice cap | 1 024 bytes | `codex-rs/core/src/agent_message_board.rs:233-245` |
| tool defaults | limit_chars 20 000, max_chars_per_post 1 000, max results 20, ≤ 256 recipients | `codex-rs/ext/agent-message-board/src/tools/spec.rs:25-141` |

## Evolution
- 2026-09-21 `abbdde95b5`, `293de27177`, `8f2c15c398` interfaces + SQLite + paging; `695ab0a0b4` "Add collaboration tools for the agent message board (#46985)".
- `68e1a421f5` remote notice context-budget cap.

## Versus pi
- pi has no multi-agent coordination substrate (not in findings).
