---
type: implementation
harness: codex
concept: skill-progressive-disclosure
commit: 622e9e3696
files: [codex-rs/ext/skills/src/catalog_prompt.rs:3, codex-rs/ext/skills/src/catalog_prompt.rs:24, codex-rs/ext/skills/src/catalog_prompt.rs:87, codex-rs/ext/skills/src/fragments.rs:53, codex-rs/ext/skills/src/render.rs:19, codex-rs/ext/skills/src/render.rs:1111, codex-rs/ext/skills/src/host_roots.rs:24, codex-rs/ext/skills/src/world_state.rs:12, codex-rs/ext/skills/src/tools/list.rs:31, codex-rs/ext/skills/src/provider.rs:27, codex-rs/ext/skills/src/dynamic_skill_selector.rs]
---
[[skill-progressive-disclosure]] in [[codex]].

## Mechanism
- **Roots**: user, repo `.agents/skills` in every dir between project root and cwd, system (bundled, → [[codex--self-documentation-pointer]]), admin (managed); catalog order by scope rank System < Admin < Repo < User (`codex-rs/ext/skills/src/host_roots.rs:24-26`, `:73-120`; `order_entries` in `codex-rs/ext/skills/src/render.rs`). Plugins contribute skills under a plugin prefix; plugin commands become skills on install ([[codex--prompt-template-expansion]]). Roles can only disable skills ([[agent-profiles]]).
- **Rendered block** (developer role): `## Skills` → optional `### Skill roots` alias table → `### Available skills` (`codex-rs/ext/skills/src/catalog_prompt.rs:87-105`), wrapped in `<skills_instructions>` (`codex-rs/ext/skills/src/fragments.rs:53`). Three intro variants by locator kind: host files with short alias paths; executor/cloud packages read via `skills.read({"package":...})`; resource aliases (`codex-rs/ext/skills/src/catalog_prompt.rs:3-7`, `:98`).
- **Rules text** (`codex-rs/ext/skills/src/catalog_prompt.rs:24-40`): "Trigger rules: If the user names a skill (with `$SkillName` or plain text) OR the task clearly matches a skill's description shown above, you must use that skill for that turn. Multiple mentions mean use them all. Do not carry skills across turns unless re-mentioned." / "open and read its `SKILL.md` completely before taking task actions. If a read is truncated or paginated, continue until EOF." / "Do not delegate reading, summarizing, or interpreting skill instructions to a subagent." / "If `scripts/` exist, prefer running or patching them instead of retyping large code blocks." / "Announce which skill(s) you're using and why (one short line). If you skip an obvious skill, say why." / "Avoid deep reference-chasing".
- **Budgets** (`codex-rs/ext/skills/src/render.rs:19-29`, `:126-152`): configured `skills.max_context_tokens` capped at 10,000 tokens, else 2 % of the context window, else 8,000 chars; descriptions truncated at 1,024 chars with "..."; if still over, all descriptions removed with warning "Exceeded skills context budget. All skill descriptions were removed and…". Paths compressed to root aliases when absolute paths would exceed the budget. Explicitly invoked skill body truncated at `MAX_SKILL_PROMPT_BYTES = 8_000` (`:1111-1113`).
- **World state**: catalog sections `skills`, `cloud_skills`, `host_skills` are diffed; when hidden/over budget the model is told "Explicit skill mentions can still be resolved when available" (`codex-rs/ext/skills/src/world_state.rs:12-24`) → [[codex--world-state-diff-injection]].
- **Tools**: `list` (20 per page) and `read` (`codex-rs/ext/skills/src/tools/list.rs:31-33`); resources up to 1 MiB (`codex-rs/ext/skills/src/provider.rs:27`).
- **Selector** (shadow only): lexical character n-gram / fielded BM25 / LRU (`codex-rs/ext/skills/src/dynamic_skill_selector.rs`), metrics before letting retrieval prune the catalog.
- Review sub-agent includes skills (`be212db0c8`).

## Constants
| name | value | path:line |
|---|---|---|
| `DEFAULT_SKILL_METADATA_CHAR_BUDGET` | 8_000 chars | `codex-rs/ext/skills/src/render.rs:19-29` |
| `MAX_CONFIGURED_SKILL_METADATA_TOKEN_BUDGET` | 10_000 tokens | `codex-rs/ext/skills/src/render.rs:19-29` |
| `SKILL_METADATA_CONTEXT_WINDOW_PERCENT` | 2 % | `codex-rs/ext/skills/src/render.rs:19-29` |
| `MAX_CATALOG_SKILL_DESCRIPTION_CHARS` | 1_024 | `codex-rs/ext/skills/src/render.rs:19-29` |
| `MAX_SKILL_PROMPT_BYTES` | 8_000 | `codex-rs/ext/skills/src/render.rs:1111-1113` |
| skills list page / resource cap | 20 / 1 MiB | `codex-rs/ext/skills/src/tools/list.rs:31-33`, `codex-rs/ext/skills/src/provider.rs:27` |

## Evolution
- 2025-12-01 `a8d5ad37b8` skills (SKILL.md) experimental.
- 2025-12-03 `9a50a04400` "feat: Support listing and selecting skills via $ or /skills (#7506)" (trigger rules, no carry-over across turns).
- 2026-01-05 `57f8158608` "chore: improve skills render section (#8459)": "Remove confusing trigger/discovery wording", added "Avoid deep reference-chasing".
- 2026-04-24 `1e560f33e1` "feat: Compress skill paths with root aliases (#19098)".
- 2026-06-03 `2d385e166c` "feat: add skills extension scaffold (#25953)" (`codex-rs/ext/skills`); 2026-08-07 `45f8cafa4e` "Remove the codex-core-skills crate (#37505)".
- 2026-06-08 `56554904ba` "[codex] Require complete main-agent skill reads (#27044)": REMOVED "open its `SKILL.md`. Read only enough to follow the workflow." and "load only the specific files needed for the request; don't bulk-load everything"; ADDED read-to-EOF, no delegation of skill interpretation, and "Progressive disclosure applies to selecting relevant files, not partially reading a selected instruction file." → [[partial-file-read-acted-on]].
- 2026-07-13 `c100109280` "Add shadow metrics for lexical skill selection (#32761)".
- 2026-07-15 `2cd6ed7509` plugin commands → skills.

## Versus pi
- [[pi--skill-progressive-disclosure]]: `<skills>` system-prompt section, reader-aware hint, trust-gated project skills, spec limits 64/1024. codex: developer-role block, context-window-relative budget with degradation, mandatory full reads, dedicated list/read tools for non-host packages; both converged on "read the chosen file completely" after partial-read failures.
