---
type: implementation
harness: codex
concept: dynamic-tool-guidelines
commit: 622e9e3696
files: [codex-rs/prompts/src/update_plan_instructions.rs, codex-rs/models-manager/src/model_info.rs:117, codex-rs/core/src/context/world_state/collaboration_mode.rs:26, codex-rs/core/src/tools/handlers/multi_agents_spec.rs:692]
---
[[dynamic-tool-guidelines]] in [[codex]].

## Mechanism
- **Inverse of pi**: the prompt is a fixed model-owned document ([[codex--per-model-system-prompt]]); when a tool is disabled the harness *strips* the sections that mention it by matching literal headings (`## Planning`, `## \`update_plan\``, `## Plan tool`, `## Plan Mode vs update_plan tool`) and bullet prefixes (`- Use the plan tool `) (`codex-rs/prompts/src/update_plan_instructions.rs`). Applied only to Codex-owned text: "custom instructions must remain unchanged". Same stripping applied to collaboration-mode text (`codex-rs/core/src/context/world_state/collaboration_mode.rs:26-60`).
- **Catalog flags** `include_skills_usage_instructions`, `include_plugin_usage_instructions`, `include_apps_usage_instructions` gate whole fragments per model (`codex-rs/models-manager/src/model_info.rs:117-119`); all false for unknown models.
- **Placement rule** observed: behavioural guidance for always-present tools lives in the system prompt (update_plan moved out of its description, `30ee24521b`), while optional tools (spawn_agent, goal, plugin install) carry behaviour in their own descriptions (`codex-rs/core/src/tools/handlers/multi_agents_spec.rs:692-770`) → [[tool-description-design]].
- Historical tool-availability prompt variant: models without a native apply_patch tool got `BASE_INSTRUCTIONS_WITH_APPLY_PATCH` = base + apply_patch grammar (`a1abd53b6a^:codex-rs/core/src/models_manager/model_info.rs:20-21`); otherwise the grammar rode on the tool definition.

## Constants
| name | value | path:line |
|---|---|---|
| `tools.update_plan.enabled` default | `false` (since `a9519cbcdd`) | `codex-rs/config/src/config_toml.rs:702` (field) |
| stripped headings | `## Planning`, `## \`update_plan\``, `## Plan tool`, `## Plan Mode vs update_plan tool`; bullet `- Use the plan tool ` | `codex-rs/prompts/src/update_plan_instructions.rs:12-15`, `:43` |
| plan-tool skip threshold (prompt) | "roughly the easiest 25%" | `codex-rs/core/gpt_5_codex_prompt.md:24` |

## Evolution
- 2025-07-31 `6ce0a5875b` "Initial planning tool" heavily prompted.
- 2025-08-13 `30ee24521b` "fix: remove behavioral prompting from update_plan tool def (#2261)" — guidance moved from tool description into the system prompt (opposite direction of pi).
- 2025-11-19 `4985a7a444` "fix: parallel tool call instruction injection (#6893)": `base.replace("## Editing constraints", INSTRUCTIONS)` silently dropped the guidance for prompts lacking the heading → append to the family instructions instead; template `80140c6d9d:codex-rs/core/templates/parallel/instructions.md` ("Think first… Batch everything… Use `multi_tool_use.parallel`… Do not try to parallelize using scripting") deleted 2025-12-09 `6382dc2338` "enable parallel tc", folded into per-model prompts (`codex-rs/core/gpt_5_2_prompt.md:252`) → [[anchor-based-prompt-injection-silently-fails]].
- 2026-02-09 `a1abd53b6a` per-family prompt variants (incl. `BASE_INSTRUCTIONS_WITH_APPLY_PATCH`) removed with the offline fallback.
- 2026-08-31 `a9519cbcdd` "Make the update_plan tool opt-in (#41744)": default off; bundled update_plan guidance removed from model, collaboration-mode, multi-agent, compaction, prewarm and goal-continuation prompts when disabled → [[task-list-tool]].

## Quirks
- Stripping by literal heading text is the same fragility class as anchor splicing: a catalog prompt that renames `## Planning` keeps stale tool guidance (no assertion found — unverified).

## Versus pi
- [[pi--dynamic-tool-guidelines]] *generates* rules from declared tools (per-tool snippets, dedupe). codex *subtracts* from a model-owned document. Both converge on "never mention a tool the request doesn't declare" ([[prompt-names-unavailable-tools]]).
