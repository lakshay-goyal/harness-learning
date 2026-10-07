---
type: implementation
harness: opencode
concept: search-tools
commit: ecc4916b5a
files: [packages/core/src/ripgrep.ts:18-20, packages/core/src/ripgrep.ts:161-195, packages/core/src/ripgrep.ts:268-269, packages/core/src/ripgrep/binary.ts:14, packages/opencode/src/tool/glob.ts:49-50, packages/opencode/src/tool/grep.ts:80, packages/opencode/src/tool/glob.txt:1-6, packages/opencode/src/tool/grep.txt:1-8]
---
[[search-tools]] in [[opencode]].

## Mechanism
- Dedicated `glob` and `grep` tools on by default (no `ls`; directories via `read`).
- Shared `Ripgrep.Service` in core: system `rg` if present, else ripgrep 15.1.0 downloaded per platform into the harness bin dir (`packages/core/src/ripgrep/binary.ts:14`).
- Argv always `--no-config`, `--glob=!**/.git/**`; grep adds `--json`, `--hidden` (when requested), `--no-messages`, `--` (`packages/core/src/ripgrep.ts:161-195`).
- Caps: 100 results for glob and grep (`packages/opencode/src/tool/glob.ts:49`; `packages/opencode/src/tool/grep.ts:80`); per-line 2000 chars (`packages/core/src/ripgrep.ts:268-269`); JSON record 64 KiB, 100 submatches, 8 KiB error bytes (`ripgrep.ts:18-20`).
- Truncation stated: grep appends "(Results truncated. Consider using a more specific path or pattern.)" (`packages/opencode/src/tool/grep.ts:101`).
- Each tool asks its own permission keyed on the pattern and checks external-directory access on the search path → [[workspace-boundary-check]].
- glob.txt points open-ended multi-round searches at the Task tool → [[task-owned-subagent]].
- v2 runtime: no default limit (`MAX_SAFE_INTEGER`, `packages/core/src/tool/glob.ts:80`); broad globs exclude hidden path segments (`specs/v2/schema-changelog.md:220,225`).

## Constants
| name | value | path:line |
|---|---|---|
| ripgrep version | 15.1.0 | `packages/core/src/ripgrep/binary.ts:14` |
| result limit | 100 | `packages/opencode/src/tool/glob.ts:49`, `packages/opencode/src/tool/grep.ts:80` |
| line cap | 2000 chars | `packages/core/src/ripgrep.ts:268-269` |
| `MAX_RECORD_BYTES` / `MAX_SUBMATCHES` | 64 KiB / 100 | `packages/core/src/ripgrep.ts:19-20` |

## Evolution
- 2025-12-01 `95c3a8b805` grep line length capped at 2000.
- 2025-12-23 `5af35117db` Windows CRLF in grep; 2026-01-15 `4edb4fa4fa` broken symlinks handled.
- 2025-12-25 `f397c92ddf` `list` tool removed.
- 2026-02-12 `624dd94b5d` "tool outputs to be more llm friendly" (truncation and error wording).
- origin/v2: plugins deleting grep/glob for GPT and Claude exist but are disabled "until we can figure out a good ux for displaying heavy grep/glob usage done via shell" (`origin/v2:packages/core/src/plugin/optimize.ts:32-49`) → [[model-specific-toolset]].

## Quirks / drift
- grep.txt: "If you need to identify/count the number of matches within files, use the Bash tool with `rg` (ripgrep) directly" contradicts the shell description's "Content search: Use Grep (NOT grep or rg)" (`packages/opencode/src/tool/shell/prompt.ts:100-103`) → [[tool-description-drifts-from-implementation]].

Contrast: pi keeps grep/find/ls off by default and lets the model use rg via bash → [[pi--search-tools|pi]].
