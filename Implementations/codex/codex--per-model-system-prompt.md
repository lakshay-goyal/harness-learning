---
type: implementation
harness: codex
concept: per-model-system-prompt
commit: 622e9e3696
files: [codex-rs/models-manager/models.json, codex-rs/protocol/src/openai_models.rs:540, codex-rs/models-manager/src/model_info.rs:16, codex-rs/models-manager/src/model_info.rs:45, codex-rs/models-manager/src/model_info.rs:97, codex-rs/protocol/src/models.rs:1537, codex-rs/protocol/src/prompts/base_instructions/default.md:1, codex-rs/models-manager/prompt.md]
---
[[per-model-system-prompt]] in [[codex]].

## Mechanism
- **Catalog**: every model in bundled `codex-rs/models-manager/models.json` (472 KB) carries `model_messages.instructions_template` (17,297–21,769 chars; gpt-5.5 21,459, gpt-6-astra 21,420) plus per-model overrides `approvals`, `permissions`, `collaboration_modes`, `multi_agent`, `auto_review`, `tools`, `token_budget`, `guardian_v2`, `persistent_instructions`, `confirmation_policies`, `content_filter_guidance` (struct `ModelMessages`, `codex-rs/protocol/src/openai_models.rs:540-575`). The catalog is also fetched from the models endpoint, so OpenAI can change a model's system prompt without a client release.
- Struct doc: "`instructions_template`: Formerly a personality template, now literal text. The name is retained for model catalog compatibility." (`codex-rs/protocol/src/openai_models.rs:540-543`).
- `content_filter_guidance`: "Missing, null, blank, or values over 512 UTF-8 bytes use the bundled guidance" (`codex-rs/protocol/src/openai_models.rs:547-549`) — remote fragment length-capped. `persistent_instructions`: missing → bundled, empty string → disabled (`codex-rs/protocol/src/openai_models.rs:550-553`).
- **Resolution** (`codex-rs/models-manager/src/model_info.rs`): user `config.base_instructions` replaces the template wholesale (`:45-48`); `personality = "none"` strips the `# Personality` H1 section (`:17`, `:61-95`, → [[codex--personality-variants]]); unknown slug → `model_info_from_slug` warns "Unknown model {slug} is used. This will use fallback model metadata." and uses `BASE_INSTRUCTIONS = include_str!("../prompt.md")` via `local_model_messages()` (`:16`, `:97-160`), fallback window 272_000, `effective_context_window_percent` 95, truncation `bytes(10_000)`, all `include_*_usage_instructions` false.
- `ResolvedModelMessages` distinguishes missing (use bundled default) vs explicit empty (suppress) (`a8c36ca6d2`).
- **Fallback text** `codex-rs/models-manager/prompt.md` is byte-identical to `codex-rs/protocol/src/prompts/base_instructions/default.md` (`BASE_INSTRUCTIONS_DEFAULT`, `codex-rs/protocol/src/models.rs:1537`), 275 lines / 20,903 bytes. Anatomy: identity + "Your capabilities" (1-9) → `# How you work` / `## Personality` (11-15) → `# AGENTS.md spec` (17-27) → `## Responsiveness` / `### Preamble messages` (29-50) → `## Planning` + good/bad plan examples (52-121) → `## Task execution` (123-147) → `## Validating your work` (149-163) → `## Ambition vs. precision` (165-171) → `## Sharing progress updates` (173-179) → `## Presenting your work and final message` + `### Final answer structure and style guidelines` (181-256) → `# Tool Guidelines` / `## Shell commands` / `## \`update_plan\`` (258-275). Markdown headings, no XML.
- **Delivery**: since 2026-10-05 `c9253c4977` the resolved text is a leading `developer` input message with stable id, not the Responses `instructions` field → [[codex--message-role-layering]].
- **Orphans**: `codex-rs/core/gpt_5_codex_prompt.md`, `gpt_5_1_prompt.md`, `gpt_5_2_prompt.md`, `gpt-5.1-codex-max_prompt.md`, `gpt-5.2-codex_prompt.md` are unreferenced since `a1abd53b6a` yet still edited by sweeping commits (`2cf2a6a844` 2026-06-23); the two codex-max/5.2-codex files are byte-identical.

## Constants
| name | value | path:line |
|---|---|---|
| fallback base prompt size | 20,903 bytes / 275 lines (from 5,709 bytes at `31d0d7a305`) | `codex-rs/protocol/src/prompts/base_instructions/default.md` |
| catalog prompt sizes | 17,297–21,769 chars per model | `codex-rs/models-manager/models.json` |
| codex-tuned prompt size | 6,647 bytes (gpt-5-codex); 7,589 (5.1-codex-max / 5.2-codex) | `codex-rs/core/gpt_5_codex_prompt.md` |
| catalog content-filter guidance cap | 512 UTF-8 bytes | `codex-rs/protocol/src/openai_models.rs:547-549` |
| unknown-model fallback | window 272_000, 95 % effective, tool output bytes 10_000 | `codex-rs/models-manager/src/model_info.rs:126-132` |
| format retry budget (prompt) | "iterate up to 3 times to get formatting right" | `codex-rs/protocol/src/prompts/base_instructions/default.md:155` |
| preamble / progress length | "8–12 words for quick updates"; "no more than 8-10 words" | `codex-rs/protocol/src/prompts/base_instructions/default.md:36`, `:175` |
| final answer length | "no more than 10 lines" default | `codex-rs/protocol/src/prompts/base_instructions/default.md:191` |
| GPT-5.1 verbosity tiers | tiny ≤3 bullets; medium ≤6 bullets; large 1–2 bullets/file; ≤2 snippets | `codex-rs/core/gpt_5_1_prompt.md` "**Verbosity**" (`8dcbd29edd`) |
| plan-tool skip threshold | "roughly the easiest 25%" of tasks | `codex-rs/core/gpt_5_codex_prompt.md:24` |

## Evolution
- 2025-04-16 `59a180ddec` TS `codex-cli` prompt prefix in `75febbdefa:codex-cli/src/utils/agent/agent-loop.ts`; removed with all TS 2025-08-08 `408c7ca142`.
- 2025-04-24 `31d0d7a305` Rust `31d0d7a305:codex-rs/core/prompt.md` (5,709 bytes), one prompt for every model — a Codex-web *container* prompt: "You are a deployed coding agent. Your session is backed by a container...", "internet access is disabled in the container", "You do not need to `git commit` your changes; this will be done automatically for you." Removed 2025-08-05 `d31e149cb1` / 2025-08-07 `81b148bda2` (GPT-5 rewrite, +265/−75).
- 2025-09-14 `916fdc2a37` "Add per-model-family prompts (#3597)": `swiftfox_prompt.md` → `codex-rs/core/gpt_5_codex_prompt.md` (`f037b2fd56` 2025-09-15); codex-tuned models get the short prompt.
- Before 2026-02-09 selection was a `starts_with` chain in `find_model_info_for_slug` (`a1abd53b6a^:codex-rs/core/src/models_manager/model_info.rs:115-310`): o3/o4-mini/codex-mini/gpt-4.1/gpt-4o/gpt-3.5/plain gpt-5 → `BASE_INSTRUCTIONS_WITH_APPLY_PATCH` (base + apply_patch grammar); gpt-5-codex/gpt-5.1-codex/codex-* → `gpt_5_codex_prompt.md`; gpt-5.1-codex-max → own; gpt-5.2-codex/bengalfox/exp-codex → 5.2-codex prompt + `{{ personality }}`; gpt-5.2/boomslang → `gpt_5_2_prompt.md`; gpt-5.1 → `gpt_5_1_prompt.md`. Codenames as pre-launch slug prefixes: swiftfox, arcticfox (`d5dfba2509`), robin (`238ce7dfad`), caribou (`f084e5264b`), bengalfox, boomslang.
- 2025-11-13 `8dcbd29edd` GPT-5.1 prompt: "## Autonomy and Persistence", verbosity tiers.
- 2026-01-19 `675f165c56` "Preserve base_instructions in SessionMeta (#9427)": copy at `codex-rs/protocol/src/prompts/base_instructions/default.md`.
- 2026-02-09 `a1abd53b6a` "Remove offline fallback for models (#11238)": all per-model `include_str!` removed; catalog is the only source.
- 2026-04-02 `6fff9955f1` models-manager extracted; byte-identical `codex-rs/models-manager/prompt.md`.
- 2026-08-03 `df72fdb415` "Consolidate model instructions in `ModelMessages` (#36787)": `ModelInfo.base_instructions` removed as a source; legacy values promoted into `instructions_template`.
- 2026-09-03 `ed391d4dd2` GPT-6-Astra catalog text (bias to action; no unsolicited disclaimers).
- 2026-09-16 `a8c36ca6d2` "Centralize model-message resolution and rendering in `codex-prompts` (#46026)".
- **Single-line behavioural patches** (each = one observed model behaviour):
  - "Do not waste tokens by re-reading files after calling `apply_patch` on them. The tool call will fail if it didn't work." (`81b148bda2`, `codex-rs/protocol/src/prompts/base_instructions/default.md:143`).
  - "Do not use one-letter variable names unless explicitly requested." / "Do not add inline comments within code unless explicitly requested." (`81b148bda2`, `default.md:145-146`) vs gpt-5-codex "Add succinct code comments… Usage of these comments should be rare." (`916fdc2a37`).
  - "Do not repeat the full contents of the plan after an `update_plan` call — the harness already displays it." (`81b148bda2`, `default.md:58`).
  - "Do not use python scripts to attempt to output larger chunks of a file." (`90d892f4fd`; `default.md:265`) — model evading output truncation.
  - "Default to ASCII when editing or creating files." (`916fdc2a37`, `codex-rs/core/gpt_5_codex_prompt.md:9`).
  - "The user does not command execution outputs. When asked to show the output of a command (e.g. `git show`), relay the important details" (`916fdc2a37`, typo in original) — model assumed the user sees tool output.
  - "If you're building a web app from scratch, give it a beautiful and modern UI" (`9429e8b219`); "## Frontend tasks… avoid collapsing into "AI slop"… avoid purple-on-white defaults. No purple bias or dark mode bias." (`7e0e675db4` 2025-11-18 arcticfox; `codex-rs/core/gpt-5.2-codex_prompt.md:33-44`).
  - "Parallelize tool calls whenever possible… Use `multi_tool_use.parallel` to parallelize tool calls and only this." (`238ce7dfad` 2025-12-11) → newest: "Batch independent searches and reads in one functions.exec using await Promise.allSettled([...])" (gpt-6-astra catalog text).
  - "Do not chain shell commands with separators like `echo "====";` or `printf '---'`; the output becomes noisy" (`c10f95ddac` 2026-04-24, `models.json`).
  - "Never repurpose `$HOME`, `$home`, or `$CODEX_HOME`." (`d26a9bf671` 2026-07-18, `models.json`).
  - "`JSON.stringify()` is not shell escaping" (`ed391d4dd2` / `49e95cc73f` 2026-09, `models.json`) — code-mode era.
  - "Avoid performing blocking sleep or wait calls longer than 60 seconds" (`3380969a29` 2026-07-09, `models.json`).
  - "Do not use tools to send messages to others (e.g. through slack or email) unless given explicit instructions" (`ed391d4dd2`).
  - "Do not let these settings or the sandbox deter you from attempting to accomplish the user's task" (`81b148bda2`) — model too timid under sandbox.
  - `a6139aa003` 2025-08-04 "At the start of the task, call `update_plan`" → "At the start of any nontrivial task" (plan spam on trivial tasks); gpt-5-codex "Skip using the planning tool for straightforward tasks (roughly the easiest 25%)." (`codex-rs/core/gpt_5_codex_prompt.md:24`).
  - "Use the `apply_patch` tool to edit files (NEVER try `applypatch` or `apply-patch`, only `apply_patch`)" and "NEVER output inline citations like "【F:README.md†L5-L14】"…" (`81b148bda2`, `default.md:132,147`) → [[foreign-harness-tool-hallucination]].
  - "Don’t use literal words “bold” or “monospace” in the content." (`81b148bda2`, `default.md:248`); "avoid naming formatting styles in answers" (`gpt_5_codex_prompt.md:57`) → [[formatting-instruction-echoed-literally]].
  - "include an info string as often as possible" (`35c76ad47d`); File References block (`b1c291e2bb`/`d60cbed691`); "Don’t output ANSI escape codes directly" (`default.md:250`) → [[client-unrenderable-output-format]].
  - "You may be in a dirty git worktree. NEVER revert existing changes you did not make unless explicitly requested…" (`916fdc2a37`); "**NEVER** use destructive commands like `git reset --hard` or `git checkout --`…" (`0ad1b0782b`); "Do not amend a commit unless explicitly requested to do so." (`8c75ed39d5`) (`codex-rs/core/gpt_5_codex_prompt.md:12-19`) → [[destructive-git-on-user-changes]].
  - "hold off on running tests or lint commands until the user is ready…" (`3d8bca7814`); "Do not write tests for reversible, low-impact changes or that mirror the implementation." (`ed391d4dd2`) → [[over-validation-in-interactive-mode]].
  - "Persist until the task is fully handled end-to-end…" (`8dcbd29edd`); "Do not stop at acknowledging capability…" / "Do not request permission again…" (`ed391d4dd2`) → [[premature-turn-end]].

## Quirks
- Prompts are Markdown-headed documents; harness-injected fragments are XML → [[codex--xml-prompt-boundaries]].
- Same model family serves several OpenAI surfaces; prompts must forbid the other surfaces' conventions (citations, container assumptions).
- The base-prompt AGENTS.md section says context files arrive "with the developer message", but the fragment is role `user` (stale wording, `default.md:17-27`).
- Absences (W6): [[no-offline-per-model-prompts]], [[no-top-level-instructions-field]].

## Versus pi
- pi: one harness-authored ~680-token prompt, model-agnostic, must not override identity ([[pi--minimal-system-prompt]]). codex: model-owned 17–22 KB data, remotely updatable, names the model; size tracks how in-distribution the model is (~7 KB for codex-tuned models). → [[single-vs-per-model-system-prompt]].
