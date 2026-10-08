---
type: implementation
harness: codex
concept: subagent-result-mailbox
commit: 622e9e3696
files: [codex-rs/core/src/agent/control/completion.rs:27, codex-rs/core/src/context/inter_agent_completion_message.rs:23, codex-rs/core/src/session_prefix.rs:9, codex-rs/core/src/agent/types.rs:54, codex-rs/core/src/agent/control/delivery.rs:12, codex-rs/core/src/agent/control/mailbox.rs:1, codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs, codex-rs/core/src/config/mod.rs:259]
---
[[subagent-result-mailbox]] in [[codex]].

## Mechanism
1. **Completion notice**: on a child's terminal turn, `notify_parent_of_terminal_turn` builds `InterAgentCommunication{author: child_path, recipient: parent_path, trigger_turn: false}` and sends it to the parent thread (`codex-rs/core/src/agent/control/completion.rs:27`, `:112-126`); best-effort, counter `codex.multi_agent.result_delivery{outcome=queued|failed}` (`:129-137`).
2. **Envelope**: `Message Type: FINAL_ANSWER\nTask name: …\nSender: …\nPayload:\n<final message>`, stored with role **assistant**, content kind `multi_agent.inter_agent_completion_message` (`codex-rs/core/src/context/inter_agent_completion_message.rs:23-45`). Final answer passed untruncated; only error text truncated to `ERROR_MAX_TOKENS` = 900 (1 000 − 100 envelope reserve) plus next-action hint "This agent's turn failed. If you still need this agent, use the available collaboration tools to give it another task." (`codex-rs/core/src/session_prefix.rs:9-13`, `:9-36`). Running / Interrupted / PendingInit produce no message.
3. **Guardian-denial case**: child ending with `TooManyDenials` → "Tell the user this agent stopped after repeated Guardian denials. Do not resume this agent or retry its blocked work until the user explicitly confirms that it should continue." (`codex-rs/core/src/session_prefix.rs:14`, `:38-46`; `codex-rs/core/src/agent/control/completion.rs:89-96`) → [[delegated-authorization-provenance]].
4. **Delivery modes**: `send_message` = `QueueOnly` (`InterAgentMessageType::Message`, does not start an idle agent); `followup_task` = `TriggerTurn` (`NewTask`, wakes the target) (`codex-rs/core/src/agent/types.rs:54-60`; `codex-rs/core/src/agent/control/delivery.rs:12-39`; `codex-rs/core/src/tools/handlers/multi_agents_v2/message_tool.rs` header).
5. **Store**: per-thread `Vec<StoredMail>` + `watch::Sender<bool>`, retained in memory while the recipient session is unloaded (`codex-rs/core/src/agent/control/mailbox.rs:1-60`) → [[subagent-eviction-loses-mail]].
6. **Pickup**: `submission_loop` selects on the mailbox watch and starts a turn when the thread is durably asleep (`codex-rs/core/src/session/handlers.rs:450-479`); `start_task` drains mail into the new turn's pending input (`codex-rs/core/src/tasks/mod.rs:287-425`); in-turn, mail drains with steers at the loop top; queue-only child mail does not force a resample and defers to the next turn unless other same-turn input exists (`codex-rs/core/src/session/input_queue.rs:323-344`); mid-stream preemption after reasoning/commentary (`codex-rs/core/src/session/turn.rs:2747-2811`) → [[follow-up-queue]].
7. **`wait_agent` V2**: only `timeout_ms`; waits for *any* mailbox activity, a user steer, or timeout; returns `{message: "Wait completed." | "Wait interrupted by new input." | "Wait timed out.", timed_out}` — content arrives through the mailbox, not the tool output (`codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs`, `WaitAgentResult::from_outcome`). Pending mail checked first (`has_pending_mailbox_items`, `subscribe_activity` returns `pending_activity`) → [[wait-misses-already-queued-result]]. Steer wakes waiters (`WaitOutcome::Steered`) → [[blocking-wait-ignores-user-steer]]. Too-small timeouts clamped up to the minimum and reported; > max is an error.

## Constants
| name | value | path:line |
|---|---|---|
| wait default / min / max | 30 000 / 10 000 / 3 600 000 ms | `codex-rs/core/src/config/mod.rs:259-261` |
| hard min/max wait bounds | 0 / 3 600 000 ms | `codex-rs/core/src/config/mod.rs:264-266` |
| completion message cap | 1 000 tokens, 100 envelope reserve → error text 900 | `codex-rs/core/src/session_prefix.rs:9-12` |
| final answer cap | none | `codex-rs/core/src/session_prefix.rs:9-36` |

## Evolution
- 2026-01-26 `375a5ef051` `MIN_WAIT_TIMEOUT_MS` = 10 000 ("Minimum wait timeout to prevent tight polling loops from burning CPU", `codex-rs/core/src/tools/handlers/multi_agents_common.rs:1-4`).
- 2026-03-30 `213756c9ab` mailbox concept.
- 2026-04-22 `639382609f` "fix: wait_agent timeout for queued mailbox mail (#18968)".
- 2026-06-15 `ee40dddbf6` "core: let steer interrupt wait_agent".
- 2026-07-15 `c28770a42f` final-answer boundary for queued agent mail.
- 2026-07-27 `8a1c941439` "Recommend longer waits in the v2 wait_agent schema".
- 2026-08-07 `4d7e3e90d9` clamp short timeouts to the minimum (report, don't reject).
- 2026-09-22 `acc20df49f` eviction vs queued messages; 2026-10-01 `960e878df4` mail preserved across session eviction.
- 2026-10-06 `c2ae67d769` `codex.multi_agent.wait.duration_ms{outcome=mailbox|steered|timed_out}` histogram.

## Quirks
- Completion arrives as an *assistant*-role message authored by the child path — the parent reads it like its own prior output with an envelope.
- `wait_agent` is deliberately content-free so results have exactly one channel.

## Versus pi
- pi example subagent returns the child's final text as the blocking tool result, capped at 50 KiB ([[pi--subagent-as-subprocess]]); pi-durable background tasks report as follow-up messages ([[pi--task-owned-subagent]]). Codex: always asynchronous, typed mailbox, uncapped final answers.
