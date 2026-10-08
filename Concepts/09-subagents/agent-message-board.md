---
type: concept
stage: subagents
tier: variant
aliases: [AgentMessageBoard, LocalAgentMessageBoard, InMemoryMessageBoards, RemoteAgentMessageBoard, codex-agent-message-board-extension, codex-agent-message-board-client, "Feature::AgentMessageBoard", create_channel, read_thread, search_posts, agent_message_board_1.sqlite]
harnesses: [codex]
---
A shared channel/thread-structured discussion board for all agents of one agent tree (create channel, post, read, search, subscribe), with subscription notifications pushed as queue-only mail into active recipients only — many-to-many coordination instead of only parent↔child messages.

## Why
- Point-to-point parent↔child mail doesn't scale to teams of agents that need shared findings, decisions and searchable history.
- Notifications are context cost and an injection surface: they must be bounded and must not wake or restore idle agents.

## Design space
- **Topology**: parent↔child mailbox only ([[subagent-result-mailbox]]) · shared Slack-like board with channels, threads, subscriptions (✔ codex).
- **Backend**: local SQLite (idempotent `request_id`) · in-memory (training/ephemeral) · remote HTTP with session-scoped bearer token tools never see (✔ codex, all three).
- **Notification policy**: wake recipients · queue-only mail to the recipient's *current* turn only; idle/unloaded agents skipped (✔ codex).
- **Trust**: remote notices > 1 KiB rejected as untrusted, post stays readable via tools (✔ codex).
- **Gating**: experimental, requires the V2 multi-agent stack (✔ codex).

## Implementations
- [[codex--agent-message-board|codex]] — `codex-rs/ext/agent-message-board` (9 tools) + `codex-rs/core/src/agent_message_board.rs` adapter + `codex-rs/agent-message-board-client`.

## Failures
none recorded (landed 2026-09-21).

## Related
[[in-process-subagent-threads]] · [[subagent-result-mailbox]] · [[tool-output-truncation]] · [[no-prompt-injection-defense]] · [[feature-flag-stages]]
