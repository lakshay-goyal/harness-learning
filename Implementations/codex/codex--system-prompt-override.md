---
type: implementation
harness: codex
concept: system-prompt-override
commit: 622e9e3696
files: [codex-rs/config/src/config_toml.rs:248, codex-rs/config/src/config_toml.rs:251, codex-rs/config/src/config_toml.rs:255, codex-rs/config/src/config_toml.rs:268, codex-rs/config/src/config_toml.rs:275, codex-rs/models-manager/src/model_info.rs:45, codex-rs/core/src/compact.rs:121, codex-rs/core/src/tasks/review.rs:119]
---
[[system-prompt-override]] in [[codex]].

## Mechanism
- **Replace**: `config.base_instructions` overwrites the model's catalog `instructions_template` (`codex-rs/models-manager/src/model_info.rs:45-48`); `model_instructions_file`: "Optional path to a file containing model instructions that will override the built-in instructions for the selected model. Users are STRONGLY DISCOURAGED from using this field, as deviating from the instructions sanctioned by Codex will likely degrade model performance." (`codex-rs/config/src/config_toml.rs:268-272`); `instructions` "System instructions" field (`:248-249`). Replacement covers only the base prompt — AGENTS.md, permissions, environment and mode fragments still arrive as separate messages ([[codex--message-role-layering]]).
- **Append**: `developer_instructions` — "Developer instructions inserted as a `developer` role message" (`codex-rs/config/src/config_toml.rs:251-253`); mode presets may carry their own `developer_instructions` ([[codex--plan-mode]]).
- **Remove harness blocks**: `include_permissions_instructions`, `include_apps_instructions`, `include_collaboration_mode_instructions`, `include_environment_context` (`codex-rs/config/src/config_toml.rs:255-265`); `personality = "none"` strips the `# Personality` section ([[codex--personality-variants]]).
- **Side prompts**: `compact_prompt` / `experimental_compact_prompt_file` replace `SUMMARIZATION_PROMPT` wholesale (`codex-rs/config/src/config_toml.rs:275`, `:557`; `.unwrap_or(SUMMARIZATION_PROMPT)` at `codex-rs/core/src/compact.rs:121-125`) → [[codex--auto-compaction]].
- **Internal replacement**: review child sessions set `sub_agent_config.base_instructions = Some(crate::REVIEW_PROMPT.to_string())` (`codex-rs/core/src/tasks/review.rs:119`) → [[review-subagent]].
- **Catalog-side override** of fragments: per-model `model_messages.{approvals, permissions, collaboration_modes, …}` replace bundled templates; missing → bundled, explicit empty → suppress ([[codex--per-model-system-prompt]]).
- No runtime hook mutates the prompt: plugins carry skills/MCP/hooks/metadata only, no code (`codex-rs/plugin/src/manifest.rs:8-58`); command hooks can add `<external_*>` additional context, `prompt`/`agent` hook types are rejected at load (`codex-rs/hooks/src/engine/discovery.rs:637-656`).

## Evolution
- 2025-10-30 `f4f9695978` "feat: compaction prompt configurable (#5959)".
- 2026-01-19 `675f165c56` base_instructions preserved in SessionMeta (resume keeps the prompt the session started with).
- 2026-08-03 `df72fdb415` `ModelInfo.base_instructions` removed as a source; override applies to `instructions_template`.

## Versus pi
- [[pi--system-prompt-override]]: three tiers (append / replace base keeping context / force everything) plus a chained structured hook. codex: replace-base and append-as-developer via config only, per-fragment toggles, and an explicit warning that replacing the vendor prompt degrades the model.
