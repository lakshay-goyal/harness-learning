---
type: implementation
harness: codex
concept: xml-prompt-boundaries
commit: 622e9e3696
files: [codex-rs/context-fragments/src/fragment.rs:30, codex-rs/context-fragments/src/fragment.rs:116, codex-rs/protocol/src/protocol.rs:122, codex-rs/core/src/context/environment_context.rs:194, codex-rs/core/src/context/world_state/environment_render_tests.rs:87, codex-rs/core/src/context/user_instructions.rs, codex-rs/core/src/client.rs:942, codex-rs/core/src/context/compaction_summary.rs:17]
---
[[xml-prompt-boundaries]] in [[codex]].

## Mechanism
- **Two layers**: (1) base prompt uses Markdown `#`/`##` headings, no XML (`codex-rs/protocol/src/prompts/base_instructions/default.md`); (2) every harness-injected fragment is XML-fenced. ~60 fragment types implement `type_markers()` (`git grep -n 'fn type_markers' codex-rs/core/src/context` → 63 hits), each tagged with a `ContentItemKind` (e.g. `agents_md.instructions`, `model.base_instructions`, `current_time.reminder`, `compaction.summary`, `images.resize_notice`, `generic.turn_aborted`, `user.text`) carried in `internal_chat_message_metadata_passthrough` and stripped for non-OpenAI providers (`codex-rs/context-fragments/src/fragment.rs:30-64`; `codex-rs/core/src/client.rs:942-952`).
- Tag constants: `<user_instructions>`, `<environment_context>`, `<environments_instructions>`, `<apps_instructions>`, `<skills_instructions>`, `<plugins_instructions>`, `<tools>`, `<collaboration_mode>`, `<multi_agent_mode>`, `<realtime_conversation>`, `<context_window>`, `<context_window_guidance>`, and `USER_MESSAGE_BEGIN = "## My request for Codex:"` (`codex-rs/protocol/src/protocol.rs:122-146`); `<permissions instructions>` (space inside the tag name, `codex-rs/prompts/src/permissions_instructions.rs:272`).
- **Recognition**: `matches_marked_text` = case-insensitive start AND end marker after trimming (`codex-rs/context-fragments/src/fragment.rs:116-130`) — lets the harness exclude its own injections from "real user message" logic: rollback boundaries, compaction keep-set (`codex-rs/core/src/context/compaction_summary.rs:17-36`; `codex-rs/core/src/context_manager/history.rs:1364-1379`), UI.
- **`<environment_context>`** is real XML with escaping (`&amp; &lt; &gt; &quot; &apos;`, `codex-rs/core/src/context/environment_context.rs:194-210`): `<cwd>`, `<shell>`, `<current_date>2026-02-26</current_date>`, `<timezone>America/Los_Angeles</timezone>`, `<filesystem><workspace_roots>…<permission_profile type="…">`, `<network enabled="true"><allowed_domains>…` (`codex-rs/core/src/context/world_state/environment_render_tests.rs:87-123`). Role `user`.
- **AGENTS.md** is a hybrid: Markdown H1 line + `<INSTRUCTIONS>` body (`codex-rs/core/src/context/user_instructions.rs`).
- Model output also uses a tag the harness parses: `<proposed_plan>` in Plan mode ([[codex--plan-mode]]); memory citations `<oai-mem-citation>` ([[codex--cross-session-memory]]).

## Evolution
- 2025-08-04 `063083af15` first wrapper `<user_instructions>` (AGENTS.md confused with the user message) → [[markdown-boundaries-ingested-inconsistently]].
- 2025-08-06 `3e8bcf0247` "[prompts] Add <environment_context> (#1869)" (cwd + sandbox as a user message).
- 2025-10-30 `2371d771cc` AGENTS.md → `# AGENTS.md instructions for <dir>` + `<INSTRUCTIONS>`.
- 2025-11-14 `0b28e72b66` compaction summary framed with SUMMARY_PREFIX so later compactions recognize it → [[injected-summary-indistinguishable]].
- 2026-02-03 `968c029471` XML few-shot tags in the approval prompt (`<good_example reason=…>`, `<bad_example>`, `<correction_to_good_example>`) replaced by a plain Markdown list — XML for injected boundaries, Markdown for instructions.
- 2026-02-26 `90cc4e79a2` `<current_date>` + `<timezone>` (date only, UTC fallback).
- 2026-08-13 `3ba52d6075` current-time reminder tagged with XML markers.

## Versus pi
- [[pi--xml-prompt-boundaries]] XML-tags every *system-prompt section* and each context file with a `path` attribute. codex keeps the system prompt Markdown and XML-fences only injected messages, using the markers additionally as a classifier for its own injected user-role content.
