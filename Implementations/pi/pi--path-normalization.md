---
type: implementation
harness: pi
concept: path-normalization
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/path-utils.ts:48, packages/coding-agent/src/core/tools/path-utils.ts:86, packages/coding-agent/src/utils/paths.ts:7, packages/coding-agent/src/utils/paths.ts:67, packages/coding-agent/src/utils/paths.ts:76, packages/coding-agent/src/utils/paths.ts:103, packages/durable/src/tools/path-utils.ts:4]
---
[[path-normalization]] in [[pi]].

All file tools resolve model-supplied paths through one normalizer; `read` additionally probes macOS filename variants. Normalization only — **no confinement** to cwd ([[no-cwd-confinement]]).

## Mechanism
- **Entry points** (`packages/coding-agent/src/core/tools/path-utils.ts`):
  - `resolveToCwd(path, cwd)` = `resolvePath(path, cwd, {normalizeUnicodeSpaces:true, stripAtPrefix:true})` (`:48-50`) — used by edit (`edit.ts:161`), write (`write.ts:65`), grep (`grep.ts:125`), find (`find.ts:111`), ls (`ls.ts:83`), edit preview diff (`edit-diff.ts:519`).
  - `expandPath` = same normalization without cwd (`:40-42`).
  - `resolveReadPath` / `resolveReadPathAsync` (`:52-84`, `:86-118`) — used only by `read` (`read.ts:124`).
- **`normalizePath(input, opts)`** (`packages/coding-agent/src/utils/paths.ts:76-100`), in order:
  1. optional `trim`.
  2. `UNICODE_SPACES = /[\u00A0\u2000-\u200A\u202F\u205F\u3000]/g` → ASCII space (`:7,78-80`).
  3. strip one leading `@` — CLI `@file` habit leaking into tool args (`:81-83`; `c9a20a3aa` #1206).
  4. Windows only: `normalizeWindowsShellPath` — Git Bash/MSYS `/c/...`, WSL `/mnt/c/...`, Cygwin `/cygdrive/c/...` → `C:\...` (skips `//`, paths with `\`, non-matching) (`:67-74,84-86`).
  5. `~` → home; `~/…` (and `~\…` on Windows) → `join(home, …)` (`:88-94`).
  6. `file://` URLs → `fileURLToPath` (`:96-98`).
- **`resolvePath(input, baseDir)`** (`:103-107`): normalize input and baseDir; absolute → `path.resolve(normalized)`, else `path.resolve(baseDir, normalized)`. `../` and absolute paths anywhere accepted.
- **macOS screenshot variants for `read`** (`path-utils.ts:5-20,52-84`): if resolved path doesn't exist, try in order: (a) ` AM.`/` PM.` → U+202F narrow NBSP before AM/PM, case-insensitive (`/ (AM|PM)\./gi`); (b) NFD (macOS stores decomposed names); (c) `'` → U+2019 (French "Capture d'écran"); (d) NFD + curly. First existing wins; else the original resolved path (so the error names the model's path).
- **Effective cwd** = `ctx?.cwd || construction cwd` in all 8 built-ins (`62835ea81` #8627 — extension-registered tools/SDK factories used creation-time `process.cwd()` instead of session cwd).
- `getCwdRelativePath` (`paths.ts:109-118`) exists for **display only** (TUI labels), not enforcement.
- System prompt `<cwd>` uses forward slashes on Windows (`671798d67` #2080: `C:\…` backslashes copied into bash broke commands) — prompt-side companion, see [[pi--shell-execution|shell-execution]].

## Constants
| name | value | path:line |
|---|---|---|
| `UNICODE_SPACES` | `[\u00A0\u2000-\u200A\u202F\u205F\u3000]` | `packages/coding-agent/src/utils/paths.ts:7` |
| `NARROW_NO_BREAK_SPACE` | U+202F | `packages/coding-agent/src/core/tools/path-utils.ts:5` |
| screenshot AM/PM regex | `/ (AM\|PM)\./gi` | `path-utils.ts:8` |
| curly apostrophe | U+2019 | `path-utils.ts:19` |

## Evolution
- 2025-12-13 `9a7bbb283` (#181): macOS screenshot filenames with unicode spaces (U+202F before AM/PM).
- 2026-01-29 `4edb506df` (#1078): NFD and curly-quote variants (+ combined).
- 2026-02-02 `c9a20a3aa` (#1206): strip `@` prefix.
- 2026-03-14 `671798d67` (#2080): forward-slash cwd in prompt.
- 2026-04-15 `d22c120b8` (#3194): lowercase am/pm (`gi`).
- 2026-09-01 `62835ea81` (#8627): `ctx.cwd` for cwd-sensitive tools.
- 0.21.0 (CHANGELOG, hash unverified): first macOS variant fallback; 0.26.1 tool factories take `cwd`; 0.68.0 shell path from session cwd (per `06-fixes` `tools-ignore-session-cwd`).
- Security history: 0.10.6 / 0.11.7 read path validation allowing cwd + ancestors + descendants (CHANGELOG `:6023`, hash unverified) — no such check exists at HEAD (`resolveToCwd` has none) → see [[read-path-traversal]] and [[no-cwd-confinement]].

## Evidence commits
`9a7bbb283` `4edb506df` `c9a20a3aa` `671798d67` `d22c120b8` `62835ea81`

## Quirks
- Step 2 turns a real U+202F into a space, then the read variant (a) puts U+202F back only before `AM.`/`PM.` — a file whose name genuinely contains U+202F elsewhere can't be addressed (inferred from code).
- Variants are only tried by `read`; `edit`/`write`/`ls` on a screenshot path with typed spaces fail or (write) create a new sibling file (inferred).
- Windows MSYS conversion only runs when `process.platform === "win32"`; drive letter uppercased.
- Security model is OS user / container: `packages/coding-agent/docs/security.md:3,15-16,33`; no symlink-escape check.

## Durable variant (packages/durable)
- `packages/durable/src/tools/path-utils.ts`: same `UNICODE_SPACES` regex and `@` strip (`:4-11`), then `env.absolutePath(…)` via the `ExecutionEnv` ([[pluggable-tool-backends]]) — `~`/`file://`/MSYS handling delegated to the env (unverified per env); `resolveReadToolPath` probes the same four variants deduped with a `Set`, checked with `env.exists` (`:17-30`).

## Failures
- [[read-path-unicode-variants]]
- [[tools-ignore-session-cwd]]
- [[find-glob-semantics-mismatch]]
- [[windows-process-tree-and-shells]]
