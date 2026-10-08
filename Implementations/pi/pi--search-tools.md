---
type: implementation
harness: pi
concept: search-tools
commit: b30a6dd77
files: [packages/coding-agent/src/core/tools/grep.ts:21, packages/coding-agent/src/core/tools/grep.ts:78, packages/coding-agent/src/core/tools/grep.ts:162, packages/coding-agent/src/core/tools/find.ts:26, packages/coding-agent/src/core/tools/find.ts:78, packages/coding-agent/src/core/tools/find.ts:182, packages/coding-agent/src/core/tools/ls.ts:11, packages/coding-agent/src/core/tools/ls.ts:62, packages/coding-agent/src/utils/tools-manager.ts:11, packages/coding-agent/src/core/tools/truncate.ts:13]
---
[[search-tools]] in [[pi]].

`grep` (ripgrep), `find` (fd), `ls` (node fs): built-in but **off by default** — opt in via `--tools` / `defaultTools: ["+grep", …]` ([[pi--minimal-default-toolset|minimal-default-toolset]]); bundle `createReadOnlyTools` = read,grep,find,ls (`packages/coding-agent/src/core/tools/index.ts:204-211`). No `constrainedSampling` on any of the three. Abbrev `packages/coding-agent/src/` = `packages/coding-agent/src/`. When bash is present and none of these are, the prompt says "Use bash for file operations like ls, rg, find" (`packages/coding-agent/src/core/system-prompt.ts:108-116`).

## Mechanism

### `grep` (`packages/coding-agent/src/core/tools/grep.ts`)
- **Schema** (`:21-33`): `pattern` "Search pattern (regex or literal string)"; `path?` "Directory or file to search (default: current directory)"; `glob?` "Filter files by glob pattern, e.g. '*.ts' or '**/*.spec.ts'"; `ignoreCase?` "Case-insensitive search (default: false)"; `literal?` "Treat pattern as literal string instead of regex (default: false)"; `context?` "Number of lines to show before and after each match (default: 0)"; `limit?` "Maximum number of matches to return (default: 100)".
- **Description** (`:78`): "Search file contents for a pattern. Returns matching lines with file paths and line numbers. Respects .gitignore. Output is truncated to 100 matches or 50KB (whichever is hit first). Long lines are truncated to 500 chars." Snippet "Search file contents for patterns (respects .gitignore)"; no guidelines (`:35-38`).
- Binary: `ensureTool("rg")` else reject "ripgrep (rg) is not available and could not be downloaded" (`:119-123`); path missing → `Path not found: {path}` (`:131`).
- **rg argv** (`:162-166`): `--json --line-number --color=never --hidden [--ignore-case] [--fixed-strings] [--glob G] -- pattern path`. `--` added `3d43d2e17` (#4018 argument injection: a pattern like `--pre=cmd` was parsed as an rg flag).
- `.gitignore` via ripgrep defaults; `--hidden` includes dotfiles (whether `.git/` contents are searched is unverified).
- Streams `--json` events; `effectiveLimit = max(1, limit ?? 100)` (`:136`); stops and kills rg at the limit (`:219-239`); exit 0/1 ok, else stderr as error (`:251-255`); none → "No matches found" (`:258`).
- **Formatting** (`:197-215`): match `relpath:line: text`, context `relpath-line- text`; paths relative to search dir, posix separators; single-file search → basename (`:137-145`). `context=0` uses rg's own line text (no file re-read; `e9ba9e2eb` #3205 broad-search stalls from sync per-match reads); `context>0` reads files (cached) (`:147-160`).
- Each line ≤ `GREP_MAX_LINE_LENGTH = 500` chars + `... [truncated]` (`truncateLine`, `truncate.ts:268-276`); overall `truncateHead(maxLines: MAX_SAFE_INTEGER)` = bytes-only 50KB (`:280-282`).
- **Notice** (one bracket, `:286-303`): `[100 matches limit reached. Use limit=200 for more, or refine pattern. 50.0KB limit reached. Some lines truncated to 500 chars. Use read tool to see full lines]`.

### `find` (`packages/coding-agent/src/core/tools/find.ts`)
- **Schema** (`:26-32`): `pattern` "Glob pattern to match files, e.g. '*.ts', '**/*.json', or 'src/**/*.spec.ts'"; `path?` "Directory to search in (default: current directory)"; `limit?` "Maximum number of results (default: 1000)".
- **Description** (`:78`): "Search for files by glob pattern. Returns matching file paths relative to the search directory. Respects .gitignore. Output is truncated to 1000 results or 50KB (whichever is hit first)." Snippet "Find files by glob pattern (respects .gitignore)".
- **fd argv** (`:182-214`): `--glob --color=never --hidden [--no-require-git] --max-results N [--full-path] -- pattern path`.
  - `--no-require-git` only when **not** inside a git repo (walk up for `.git`) so parent `.gitignore` rules stop at nested repo boundaries (`:184-198`; `756a4e8f6` #5960). Earlier `1d4fdbad2` (#3303) replaced manually collecting every nested `.gitignore` and passing `--ignore-file` (fd applied them globally → `a/.gitignore` hid files under sibling `b/`).
  - pattern containing `/` → `--full-path` + prepend `**/` unless starts with `/`, `**/` or equals `**` (`:201-209`; `c5451af74` #3302 — fd `--glob` matches basename only, so `src/**/*.spec.ts` silently returned nothing). Windows: `/` → `[/\\]` (`:211-212`; `d4eaf052b` #6817).
  - `--` before pattern (`3d43d2e17`).
- Results relativized with `path.relative`, trailing separator preserved (`:14-24`; `523b5a491` commit #7569 / findings #6104 — slicing at `searchPath.length+1` dropped the first char for root paths like `/` or `I:\`).
- fd nonzero exit with some output tolerated (`:251-257`); abort-aware incl. ignore discovery (`e9ba9e2eb` #3148). Unavailable → "fd is not available and could not be downloaded" (`:178`).
- Custom `FindOperations.glob` path (e.g. remote) ignores `**/node_modules/**`, `**/.git/**` (`:125-128`).
- Notice `[1000 results limit reached. Use limit=2000 for more, or refine pattern. 50.0KB limit reached]` (`:281-293`); none → "No files found matching pattern".

### `ls` (`packages/coding-agent/src/core/tools/ls.ts`)
- **Schema** (`:11-14`): `path?` "Directory to list (default: current directory)"; `limit?` "Maximum number of entries to return (default: 500)".
- **Description** (`:62`): "List directory contents. Returns entries sorted alphabetically, with '/' suffix for directories. Includes dotfiles. Output is truncated to 500 entries or 50KB (whichever is hit first)." Snippet "List directory contents".
- `readdir` + per-entry `stat` (follows symlinks → symlinked dirs get `/`), case-insensitive `localeCompare` sort (`:109`); unstat-able entries silently skipped (`:108-130`). **Not** gitignore-aware; non-recursive.
- Errors `Path not found: …`, `Not a directory: …`, `Cannot read directory: …` (`:87-106`); empty → `(empty directory)` (`:135`); notice `[500 entries limit reached. Use limit=1000 for more. 50.0KB limit reached]` (`:147-151`).

### Shared
- Paths: `resolveToCwd(searchDir || ".", ctx?.cwd || cwd)` (`grep.ts:125`, `find.ts:111`, `ls.ts:83`) — [[path-normalization]]; no confinement ([[no-cwd-confinement]]).
- Head truncation ([[tool-output-truncation]]); limits interpolated into descriptions ([[tool-description-design]]); notices tell the model the doubled `limit=` value.

### rg/fd binary bootstrap (`packages/coding-agent/src/utils/tools-manager.ts`)
- Lookup: pi bin dir (`getBinDir()`, also prepended to bash PATH) → system PATH (`fd`, `fdfind` for Debian `3edb8b5cb`; `rg`) verified by running `--version` (`:71-102`, `systemBinaryNames` `:34`).
- Else download latest GitHub release; version from `https://github.com/{repo}/releases/latest` **redirect Location header**, not api.github.com (`:104-138`; `57e53b0d7` #8708 — anonymous API quota 60/h exhausted behind corporate NAT/CI → "GitHub API error: 403" every launch).
- Assets darwin/linux/win32 × x86_64/aarch64; Linux **musl static** (`:42,61`; `6aedd1066` #9070); fd pinned `10.3.0` on darwin/x64 (last Intel build; `:265-267`; `7afd80d78` #4559).
- Timeouts `NETWORK_TIMEOUT_MS = 10_000`, `DOWNLOAD_TIMEOUT_MS = 120_000` (`:11-12`; `5c9ce47c5` #2066 — 10 s too short for multi-MB archives; also `pipeline()` to catch abort errors → was a startup crash).
- Extract into unique temp dir `extract_tmp_<bin>_<pid>_<time>_<rand>` (`:286-292`; `3db5715de` #1348 — concurrent fd+rg downloads raced on a fixed dir). Windows zip: System32 `tar.exe` (bsdtar) then PowerShell `Expand-Archive` (`:211-255`).
- `PI_OFFLINE=1|true|yes` skips download (`:14-18`; `757d36a41` #1631). Android/Termux: never download (Bionic libc), warn "Install with: pkg install {pkg}" (`:333,366-372`; Termux fd pkg name `7ddb7c67a` #1433).
- Download error cause chain surfaced to depth ≤ 5 (`:382-397`).

## Constants
| name | value | path:line |
|---|---|---|
| grep `DEFAULT_LIMIT` | 100 matches | `packages/coding-agent/src/core/tools/grep.ts:41` |
| `GREP_MAX_LINE_LENGTH` | 500 chars | `packages/coding-agent/src/core/tools/truncate.ts:13` |
| find `DEFAULT_LIMIT` | 1000 results | `packages/coding-agent/src/core/tools/find.ts:41` |
| ls `DEFAULT_LIMIT` | 500 entries | `packages/coding-agent/src/core/tools/ls.ts:23` |
| byte cap | 50 KiB | `truncate.ts:12` |
| `NETWORK_TIMEOUT_MS` / `DOWNLOAD_TIMEOUT_MS` | 10 s / 120 s | `packages/coding-agent/src/utils/tools-manager.ts:11-12` |
| fd pin darwin/x64 | 10.3.0 | `tools-manager.ts:267` |
| error cause depth | 5 | `tools-manager.ts:388-390` |

## Evolution
- 2025-11-29 `186169a82`: grep/find/ls added as opt-in "read-only exploration tools" with `--tools` ("safe code exploration without modification risk"); prompt rule "Prefer grep/find/ls tools over bash for file exploration (faster, respects .gitignore)" — never became default.
- 2025-12-07 `de77cd141` / `b813a8b92` (#134): limits interpolated; grep long-line notice "Some lines truncated to 500 chars. Use read tool to see full lines".
- 2026-02-12 `7ddb7c67a` (#1433) Termux name; 2026-02-25 `757d36a41` (#1631) `PI_OFFLINE` + network timeouts; 2026-02-26 `3db5715de` (#1348) Windows bootstrap hardening/unique extract dir; 2026-03-14 `5c9ce47c5` (#2066) first-run crash.
- 2026-04-16 `c5451af74` (#3302) path globs; `1d4fdbad2` (#3303) scoped nested `.gitignore`; `e9ba9e2eb` (#3148/#3205) find cancellation + grep from rg JSON.
- 2026-04-28 `3edb8b5cb`: `fdfind` fallback. 2026-04-30 `3d43d2e17` (#4018): `--` end-of-options.
- 2026-05-16 `7afd80d78` (#4559): pin fd for Intel macs.
- 2026-05-28 `1ab289980` (#5132): "Prefer grep/find/ls…" rule removed (named unavailable tools).
- 2026-06-22 `756a4e8f6` (#5960): nested repo ignore boundaries.
- 2026-08-04 `523b5a491` (#7569): find root results; 2026-08-05 `d4eaf052b` (#6817): Windows path globs.
- 2026-09-01 `62835ea81` (#8627): `ctx.cwd`.
- 2026-09-03 `57e53b0d7` (#8708) no GitHub API; `6aedd1066` (#9070) musl builds.
- Durable `CodingTools` (2026-09-29 `445770e03`) ships **no** grep/find/ls.

## Evidence commits
`186169a82` `de77cd141` `b813a8b92` `7ddb7c67a` `757d36a41` `3db5715de` `5c9ce47c5` `c5451af74` `1d4fdbad2` `e9ba9e2eb` `3edb8b5cb` `3d43d2e17` `7afd80d78` `1ab289980` `756a4e8f6` `523b5a491` `d4eaf052b` `62835ea81` `57e53b0d7` `6aedd1066` `445770e03`

## Quirks
- grep clamps limit to ≥1 (`max(1, …)`), find/ls don't (`limit ?? DEFAULT`; `find.ts:112`, `ls.ts:84`).
- `ls` ignores `.gitignore` while grep/find respect it; `ls` description says so implicitly ("Includes dotfiles").
- `--hidden` on both rg and fd: dotfiles searched; `.git/` internals inclusion unverified.
- Off-by-default tools still ship descriptions advertising `.gitignore` respect — the prompt rule preferring them was removed because it fired when they weren't declared.

## Durable variant (packages/durable)
- None: `CodingTools` = read, write, edit, bash (+ optional powershell) only (`packages/durable/src/tools/index.ts:21-25`); exploration via bash.

## Failures
- [[find-glob-semantics-mismatch]]
- [[search-ignore-rules-misapplied]]
- [[search-tool-stalls-on-broad-queries]]
- [[search-tool-argument-injection]]
- [[search-binary-bootstrap-failures]]
