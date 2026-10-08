---
type: implementation
harness: codex
concept: message-role-layering
commit: 622e9e3696
files: [codex-rs/core/src/client.rs:931, codex-rs/core/src/client.rs:996, codex-rs/core/src/context/base_instructions.rs, codex-rs/core/src/session/mod.rs:4212, codex-rs/core/src/session/mod.rs:4370, codex-rs/context-fragments/src/fragment.rs:30, codex-rs/protocol/src/protocol.rs:122]
---
[[message-role-layering]] in [[codex]].

## Mechanism
- **Base instructions**: since 2026-10-05 `c9253c4977` "Send base instructions as Responses input messages (#51156)" the resolved model prompt is spliced in front of `input` as a `developer` message with id `msg_<uuidv5(uuidv5(OID, thread_id), text)>` and content kind `model.base_instructions` (`codex-rs/core/src/client.rs:908-940`; `codex-rs/core/src/context/base_instructions.rs`). `ResponsesApiRequest` has no `instructions` field anymore (`codex-rs/core/src/client.rs:996-1011`). Changed instructions force a full request without `previous_response_id`. With Responses Lite the tool catalog is likewise a `developer`-role `AdditionalTools` prefix item (`codex-rs/core/src/client.rs:915-926`).
- **Developer-role fragments** (policy/capability): `<permissions instructions>`, `<collaboration_mode>`, multi-agent mode/role, apps/plugins/skills instructions, environments, `<model_switch>` (always inserted first in the developer bundle, `codex-rs/core/src/session/mod.rs:4370-4375`), personality (retired), `<current_time_reminder>`, content-filter guidance, token budget `<context_window>`, `<persistent_mode>`, `<realtime_conversation>`, memory read-path, user-config `developer_instructions`.
- **User-role fragments** (context, not commands): AGENTS.md (`# AGENTS.md instructions for <dir>` + `<INSTRUCTIONS>`), `<environment_context>`, `<user_shell_command>`, `<turn_aborted>`, `<subagent_notification>`, compaction summary (SUMMARY_PREFIX), `<codex_internal_context …>` goal context, `<external_*>` hook additional context.
- **Initial-context assembly order** (`codex-rs/core/src/session/mod.rs:4212-4480`): one aggregated developer message (all developer sections) → separate developer messages (multi-agent role, standalone, token budget) → multi-agent mode → one aggregated contextual-user message → guardian policy as its own developer message → managed developer instructions. Adjacent mergeable fragments of the same role are packed; standalone/prefix items stay separate (`codex-rs/core/src/context_manager/updates.rs:1-62`).
- **Markers as classifier**: every fragment implements `type_markers()` (open/close) and carries a machine `ContentItemKind` (e.g. `agents_md.instructions`, `model.base_instructions`, `current_time.reminder`, `compaction.summary`) in `internal_chat_message_metadata_passthrough`, stripped for non-OpenAI providers (`codex-rs/context-fragments/src/fragment.rs:30-64`; `codex-rs/core/src/client.rs:942-952`). Tag constants `codex-rs/protocol/src/protocol.rs:122-145`.
- Config switches per block: `include_permissions_instructions`, `include_apps_instructions`, `include_collaboration_mode_instructions`, `include_environment_context` (`codex-rs/config/src/config_toml.rs:255-265`).

## Evolution
- 2025-08-04 `063083af15` "[prompts] Better user_instructions handling (#1836)": first XML wrapper `<user_instructions>` because "Our recent change in #1737 can sometimes lead to the model confusing AGENTS.md context as part of the message"; added prompt lines "`user_instructions` are not part of the user's request, but guidance for how to complete the task." / "Do not cite `user_instructions` back to the user unless a specific piece is relevant." (removed in the 2025-08-07 rewrite `81b148bda2`).
- 2025-08-06 `3e8bcf0247` `<environment_context>` as a user message (cwd + sandbox).
- 2026-01-12 `87f7226cca` sandbox/approval text moved out of every static prompt into a developer "permissions" message re-injected on change → [[permission-state-prompt]].
- 2026-01-22 `8b3521ee77` personality switch as developer message (retired `132c739171`).
- 2026-06-22 `3b32d861c5` environment context migrated to world state → [[codex--world-state-diff-injection]].
- 2026-10-05 `c9253c4977` base instructions as developer input message.

## Quirks
- Base prompt says AGENTS.md is "included with the developer message" but it is a `user`-role fragment (`codex-rs/protocol/src/prompts/base_instructions/default.md:17-27`).
- Same role (user) carries both real user text and harness context; only markers distinguish them — hence `matches_marked_text` (case-insensitive start AND end after trim, `codex-rs/context-fragments/src/fragment.rs:116-130`).

## Versus pi
- pi puts all harness text into one XML-sectioned system prompt and appends deltas as system/custom messages ([[pi--minimal-system-prompt]], [[transcript-carried-system-prompt]]); codex distributes it across developer/user roles by authority and appends role-typed deltas.
