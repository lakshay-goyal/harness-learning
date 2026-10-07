---
type: implementation
harness: pi
concept: context-file-hierarchy
commit: b30a6dd77
files: [packages/coding-agent/src/core/resource-loader.ts:184-203, packages/coding-agent/src/core/resource-loader.ts:205-230, packages/coding-agent/src/core/resource-loader.ts:232-270, packages/coding-agent/src/core/resource-loader.ts:640-650, packages/coding-agent/src/core/system-prompt.ts:79-86, packages/coding-agent/src/core/system-prompt.ts:173, packages/coding-agent/docs/security.md:57]
---
[[context-file-hierarchy]] in [[pi]].

## Mechanism
- Per directory, **first match wins** among `AGENTS.override.md, AGENTS.md, AGENTS.MD, CLAUDE.md, CLAUDE.MD` (`packages/coding-agent/src/core/resource-loader.ts:184-203`); directories with those names skipped via `statSync().isFile()` (`:190-192`); content BOM-stripped (`:195`); read errors → yellow warning, continue (`:197-199`).
- `AGENTS.override.md` replaces AGENTS/CLAUDE only within the same dir; doesn't suppress other dirs (`docs/configuration.md:45`; `8ecf8a988` #7681).
- `loadProjectContextFiles` (`resource-loader.ts:232-270`): global `~/.pi/agent/<file>` first (`:242-246`), then walk cwd → root `unshift`-ing, so order = filesystem root … cwd (most specific **last**, nearest the user turn) (`:248-267`); dedupe by path (`seenPaths`, `:240,257`); termination `parentDir === currentDir` (`:262-264`).
- Nested linked git-worktree shadowing: if cwd is inside a worktree nested under its main repo and the worktree root has its own context file, skip the main repo's same-named copy ("both occupy the same logical repository scope, so loading both applies that context twice") — canonicalized via realpath; bare layouts/submodules/sibling worktrees excluded (`findShadowedContextFile`, `:205-230`; `cced6a21d` #7221).
- Render: section `project_context` = "Project-specific instructions and guidelines:" + one `<project_instructions path="…">\n…\n</project_instructions>` per file, blank-line joined (`system-prompt.ts:79-86, 173`); wrapped again `<project_context>` (`:188-191`) → [[xml-prompt-boundaries]]. Placed after `addendum`, before `skills`/`cwd`.
- Applies even with a custom SYSTEM.md (`customPrompt` replaces only preamble/tools/rules/docs, `system-prompt.ts:152-173`) → [[system-prompt-override]].
- **Not gated by project trust**: "Context files such as `AGENTS.override.md`, `AGENTS.md`, and `CLAUDE.md` load regardless of project trust unless you disable context loading. Treat instructions in a folder as untrusted input…" (`docs/security.md:57`); vs `.pi/SYSTEM.md`/`APPEND_SYSTEM.md` which require `isProjectTrusted()` (`resource-loader.ts:1209-1235`) → [[project-trust-gate]], [[no-prompt-injection-defense]].
- Disable: `--no-context-files` / `noContextFiles` → `[]` (`resource-loader.ts:642-649`; `docs/cli.md:215`); SDK `agentsFilesOverride` hook (`:650`).
- Reload via `/reload` (`slash-commands.ts:42`); durable `pi-prompt` loads once per cwd and caches (`experimental/durable/prompt.ts:27-39`).
- Example `claude-rules.ts` lists `.claude/rules/` files in the system prompt for on-demand reading (pattern: progressive disclosure for rules) (`examples/extensions/claude-rules.ts`).

## Constants
| name | value | path:line |
|---|---|---|
| candidate names | AGENTS.override.md, AGENTS.md, AGENTS.MD, CLAUDE.md, CLAUDE.MD | `resource-loader.ts:185` |
| global dir | `~/.pi/agent/` | `resource-loader.ts:242` |
| section header | "Project-specific instructions and guidelines:" | `system-prompt.ts:81` |

## Evolution
- `9e3e319f1` 2025-11-12: `AGENT.md`/`CLAUDE.md` in cwd injected as **user message** "[Project Context from AGENT.md]".
- `dca3e1cc6` 2025-11-12: hierarchical global + every ancestor, top-most first, each as separate message ("monorepos").
- `b1c2c32e2` 2025-11-12: "move context files to system prompt instead of user messages" — `# Project Context` / "The following project context files have been loaded:" / `## <path>` headings.
- `c82f9f4f8` 2025-11-13: `AGENT.md` → `AGENTS.md`.
- `4068bc556` 2026-01-17: header → "Project-specific instructions and guidelines:".
- `e2fd651eb` 2026-05-16 (merge `8e6913711`, #4541) custom path, `7577d3b8d` 2026-05-18 (merge `aad8cf660`, #4709) default path: `##` headings → `<project_context>`/`<project_instructions path>` XML.
- `89a92207f` 2026-06-05 (#5332): project trust gating initially included "instructions" (CHANGELOG `:1687`); `5cb4f597f` 2026-06-09 ungated — security doc sentence reworded: "prevents a repository from silently changing pi's ~~instructions,~~ settings or extensions"; prompt-injection list gains "context files" ("…is expected local-agent risk and cannot be reliably prevented by pi.")
- `2170363af` 2026-07-09 (#6369): Windows walk hang (compared with `resolve("/")`) → `dirname(d) === d`.
- `58c0bc2fb` 2026-07-25 (#7106): dirs named like context files caused EISDIR → skipped.
- `cced6a21d` 2026-07-29 (#7221): nested worktree double load.
- `8ecf8a988` 2026-08-05 (#7681): `AGENTS.override.md`.
- `1355cd36e` 2026-08-19 (#8337): BOM normalization of text inputs — adds `stripBom(readFileSync(filePath…))` to `loadContextFileFromDir` (verified in `git show 1355cd36e -- …/resource-loader.ts`).
- `9e05370b2` 2026-09-16: context now one patchable section `project_context` → edits to AGENTS.md mid-session become a section update, not a prefix rewrite ([[transcript-carried-system-prompt]]).

## Evidence commits
`9e3e319f1`, `dca3e1cc6`, `b1c2c32e2`, `c82f9f4f8`, `4068bc556`, `e2fd651eb`, `8e6913711`, `7577d3b8d`, `aad8cf660`, `89a92207f`, `5cb4f597f`, `2170363af`, `58c0bc2fb`, `cced6a21d`, `8ecf8a988`, `9e05370b2`.

## Quirks
- Only one file per directory: a dir with both AGENTS.md and CLAUDE.md loads AGENTS.md only.
- Walk goes to filesystem root (not git root) — `~/AGENTS.md` above a repo is picked up.
- Trust asymmetry deliberate: context files (prompt text) ungated, executable/prompt-replacing config gated; no commit rationale beyond the `5cb4f597f` doc diff.
- No size cap on context files in prompt (none found; unverified).

## Durable variant (packages/durable)
- Experimental durable TUI reuses `loadProjectContextFiles` per cwd inside the `pi-prompt` extension (`experimental/durable/prompt.ts:33`); sections re-rendered each request, only diffs stored as `pi.system` ([[transcript-carried-system-prompt]]).

## Failures
- [[markdown-boundaries-ingested-inconsistently]]
- [[context-file-loaded-twice-in-worktrees]]
- [[context-file-discovery-filesystem-edge-cases]]
