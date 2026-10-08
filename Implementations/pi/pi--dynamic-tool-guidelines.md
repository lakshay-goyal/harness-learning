---
type: implementation
harness: pi
concept: dynamic-tool-guidelines
commit: b30a6dd77
files: [packages/coding-agent/src/core/system-prompt.ts:88-125, packages/coding-agent/src/core/system-prompt.ts:150-161, packages/coding-agent/src/core/extensions/types.ts:581-584, packages/coding-agent/src/core/agent-session.ts:1700-1749, packages/coding-agent/src/core/tools/read.ts:20-23, packages/coding-agent/src/core/tools/bash.ts:45-48, packages/coding-agent/src/core/tools/edit.ts:43-50, packages/coding-agent/src/core/tools/write.ts:16-19, packages/coding-agent/src/core/tools/powershell.ts:18-21]
---
[[dynamic-tool-guidelines]] in [[pi]].

## Mechanism
- `ToolDefinition.promptSnippet` ("Optional one-line snippet… Custom tools are omitted from that section when this is not provided") and `promptGuidelines` ("appended … when this tool is active") (`packages/coding-agent/src/core/extensions/types.ts:581-584`). Built-ins export the same pair as `*ToolSystemPromptContribution` constants (`read.ts:20-23`, `bash.ts:45-48`, `edit.ts:43-50`, `write.ts:16-19`, `powershell.ts:18-21`; exported for reuse `9ab91fb93` #7671; consumed by durable `experimental/durable/prompt.ts:7-17`).
- Session collects snippets only for registered tools with a snippet ("Tools without a snippet are not listed", `agent-session.ts:1700-1706`) and passes `toolGuidelines` keyed by tool name (`:1714-1723`).
- `<tools>` list: `declaredTools = selectedTools − hiddenTools` (`system-prompt.ts:150`); entry only if snippet exists (`:157`); fallback `(none)` (`:159`); trailing line "In addition to the tools above, you may have access to other custom tools depending on the project." (`:160`).
- `buildRules()` (`system-prompt.ts:88-125`):
  1. harness cross-tool rule only when a shell exists and no grep/find/ls: "Use bash for file operations like ls, rg, find" / PowerShell / "bash or PowerShell" variants (`:108-116`);
  2. per-tool guidelines in selected-tool order (`:118-120`);
  3. extension `promptGuidelines` (`:121`);
  4. universal "Be concise in your responses", "Show file paths clearly when working with files" (`:122-123`).
  - Dedup by trimmed text via `Set` (`:93-100`) — required because bash and PowerShell contribute the identical `PI_*` guideline (`bash.ts:47`, `powershell.ts:20`).
- bash guideline emitted only when `exposeSessionEnvironment` (default true) (`bash.ts:250,257`) — guideline tied to the feature it describes → [[env-vars-as-context]].
- Recomputed per request: `_preparePromptAndToolLoadout` applies tool loadout then forces `hiddenTools` = current hidden declarations ("The tool list and rules must match the declarations the request carries", `agent-session.ts:1737-1749`); diff vs transcript sections → section patch ([[transcript-carried-system-prompt]]). Refresh before every next turn (`e547bb9f4`, `fd6659dd5` #6162).
- Skills hint also tool-derived: names `read`, else `bash`, else "indirect" (no tool name) when reader is hidden (`system-prompt.ts:174-182`) → [[skill-progressive-disclosure]].
- Extension-registered meta tools contribute too: codemode snippet "Run JavaScript that calls other tools" + guideline "Use codemode to batch independent tool calls (Promise.allSettled), chain them, or filter large output, instead of many separate calls." (`extensions/codemode/tool.ts:127-132`); tool_search snippet "Search for tools that are not loaded yet and load the matches" (`extensions/tool-search/tool.ts:229`).

### Tool contributions at HEAD (snippet | guidelines)
| tool | snippet | guidelines |
|---|---|---|
| read | "Read file contents" | "Use read to examine files instead of cat or sed." |
| bash | "Execute bash commands (ls, grep, find, etc.)" | "You can inspect PI_* environment variables for current model and session details." |
| edit | "Make precise file edits with exact text replacement, including multiple disjoint edits in one call" | 4: exact oldText; one call with multiple edits[]; matched against original/no overlap/merge nearby; keep oldText small |
| write | "Create or overwrite files" | "Use write only for new files or complete rewrites." |
| grep | "Search file contents for patterns (respects .gitignore)" | — (`grep.ts:35-37`) |
| find | "Find files by glob pattern (respects .gitignore)" | — (`find.ts:34-36`) |
| ls | "List directory contents" | — (`ls.ts:16-18`) |
| powershell | "Execute PowerShell commands" | same PI_* guideline |

## Constants
| name | value | path:line |
|---|---|---|
| default selected tools | read, bash, edit, write | `system-prompt.ts:64` |
| shell rule trigger | (bash∨powershell) ∧ ¬grep ∧ ¬find ∧ ¬ls | `system-prompt.ts:108` |
| skill reader preference | `["read","bash"]` then "indirect" | `system-prompt.ts:175-178` |

## Evolution
- `ffc9be886` 2025-10-17: static "Available tools:" + "Guidelines:" incl. "Always use bash tool for file operations like ls, grep, find".
- `186169a82` 2025-11-29: **tool-aware** prompt born with `--tools` flag + grep/find/ls: conditional "You are in READ-ONLY mode - you cannot modify files or execute arbitrary commands" (no bash/edit/write), "Use bash ONLY for read-only operations (git log, gh issue view, curl, etc.) - do NOT modify any files" (bash w/o edit/write), "Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)" (if any of grep/find/ls), "Use read to examine files before editing" (read+edit). CHANGELOG: "The system prompt now adapts to the selected tools".
- `e3dd4f21d` 2026-01-08: READ-ONLY rule removed (extension tool overrides make tool-set inference invalid; inferred).
- `b846a4bfc` 2026-01-20 (#645): bash read-only rule removed silently; "ls, grep, find" → "ls, rg, find"; "Custom tool" fallback for extension tools.
- `73734a23a` 2026-01-23: only built-ins listed — "Extension tools are already described via API tool definitions."
- `bc2fa8d6d`, `8d4a49487` 2026-03-02 (#1720): `promptSnippet`/`promptGuidelines` on `ToolDefinition`; `addGuideline` Set dedupe.
- `7817e9b22` 2026-03-17 (#2285): snippets opt-in, no fallback to `description` (CHANGELOG `:3037`).
- `235b247f1` 2026-03-22: built-in tools moved to the same mechanism ("built-in tools work like extension tools"); read rule softened → [[guideline-softening]].
- `20a57e759` 2026-03-27 → `e773527b3` 2026-03-28 (#2639): edit snippet + multi-edit guidelines (5 → 4) → [[search-replace-edit]].
- `1ab289980` 2026-05-28 (#5132): removed "Prefer grep/find/ls…" ("avoid preferring unavailable file exploration tools", CHANGELOG `:1806`).
- `bb3d7d399` 2026-07-22 (#6967) → `4e64de695` 2026-08-06 (#7128): PI_* guideline added then softened.
- `9ab91fb93` 2026-08-06 (#7671): contributions exported.
- `80e62761f` 2026-08-24 (#8512): PowerShell rule variants; duplicate PI_* guideline handled by dedupe.
- `1d6dbf9e3` 2026-09-03 (#8552): skills keep working with bash-only toolsets.
- `028c0ec56` 2026-09-30 (#10192): codemode-hidden tools not listed (`codemode.mode:"only"` had listed read/bash/edit/write that requests didn't declare, CHANGELOG `:215`).
- `c30840c2e` 2026-10-05 (#10343): `prepareLoadout` hidden declarations excluded from rules + tool list; skills hint "indirect".

### Tool API description timeline (cross-ref [[tool-description-design]])
- read: `ffc9be886` "Read the contents of a file. Returns the full file content as text." → `84dcab219` images (jpg, png, gif, webp, bmp, svg) → `c7a73d4f8` "defaults to first 2000 lines. Use offset/limit…" → `9e3e319f1` bmp/svg dropped → `de77cd141` limits interpolated from constants (`b813a8b92` #134 actionable notices) → `89636cfe6` 2026-01-24 "When you need the full file, continue with offset until complete." → `4cc339f58` bmp re-added. HEAD `read.ts:96`.
- bash: `ffc9be886` "…Commands run with a 30 second timeout." → `29900ce64` "Optionally provide a timeout in seconds." → `de77cd141` "truncated to last 2000 lines or 50KB… full output is saved to a temp file" → `80e62761f` `${config.shellName}`; `1ff5b6fdd` 2026-09-29 output-schema "Combined stdout and stderr, up to 1 MiB…" added, removed `6f1072cc0`. HEAD `bash.ts:255`.
- edit: `ffc9be886` "Edit a file by replacing exact text…surgical edits" → `20a57e759` dual-mode → `e773527b3` single `edits[]` (CHANGELOG `:2804` "mixed single-edit and multi-edit modes that caused repeated invalid tool calls and retries"), legacy folded by `prepareArguments` (`b5f425ad1`). HEAD `edit.ts:151-152`.
- write: unchanged since `ffc9be886`; "Successfully wrote ${n} bytes" result removed `e583b290a` (chars≠bytes).
- grep/find/ls: added `186169a82` with "(respects .gitignore)"; limits interpolated `de77cd141`.
- codemode/tool_search: `8562bcf66` → `6f1072cc0` one line per global; CHANGELOG `:198` (#10212) descriptions no longer include deferred tools/counts → [[cache-stable-prompt-prefix]].

## Evidence commits
`186169a82`, `e3dd4f21d`, `b846a4bfc`, `73734a23a`, `bc2fa8d6d`, `8d4a49487`, `7817e9b22`, `235b247f1`, `20a57e759`, `e773527b3`, `1ab289980`, `bb3d7d399`, `4e64de695`, `9ab91fb93`, `80e62761f`, `1d6dbf9e3`, `028c0ec56`, `c30840c2e`, `e547bb9f4`, `fd6659dd5`.

## Quirks
- Shell rule is a negative heuristic ("no grep/find/ls ⇒ use bash"), not a positive capability statement; the positive version ("Prefer grep/find/ls") was the one that broke.
- `promptGuidelines` doc comment still says "Guidelines section" though the header is now `<rules>` (`types.ts:583`) — doc drift.
- Ordering means extension guidelines can never precede built-in ones; universal rules always last.

## Durable variant (packages/durable)
- Durable `pi-prompt` maps only read/bash/edit/write contributions (`experimental/durable/prompt.ts:12-17,55-60`); other tools get no snippet/guideline. Tools are passed as `toolsAdded/Removed` deltas by `planTools` (`packages/durable/src/harness/prompt.ts:100-119`).

## Failures
- [[prompt-names-unavailable-tools]]
- [[shell-cat-instead-of-read-tool]]
- [[skills-hidden-when-read-tool-absent]]
- [[imperative-guideline-over-compliance]]
