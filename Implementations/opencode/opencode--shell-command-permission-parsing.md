---
type: implementation
harness: opencode
concept: shell-command-permission-parsing
commit: ecc4916b5a
files: [packages/opencode/src/tool/shell.ts:28-56, packages/opencode/src/tool/shell.ts:124, packages/opencode/src/tool/shell.ts:256-288, packages/opencode/src/tool/shell.ts:311-330, packages/opencode/src/tool/shell.ts:380-414, packages/opencode/src/permission/arity.ts:1-9]
---
[[shell-command-permission-parsing]] in [[opencode]].

## Mechanism
### Legacy runtime (`packages/opencode/src/tool/shell.ts`)
- **Parser**: web-tree-sitter with bash, powershell grammars loaded as wasm (`packages/opencode/src/tool/shell.ts:311-330`); PowerShell chosen by shell kind. `commands(root)` = `root.descendantsOfType("command")`, so commands inside `$(…)`, pipelines, `&&`/`||` chains are all visited (`packages/opencode/src/tool/shell.ts:124`).
- **Scan** (`packages/opencode/src/tool/shell.ts:380-414`) builds `{dirs, patterns, always}` per command node:
  - `patterns += source(node)`: the node's own text becomes one `bash` permission pattern; `a && b` → two patterns, each evaluated (`packages/opencode/src/tool/shell.ts:407-410`).
  - `always += BashArity.prefix(tokens).join(" ") + " *"` (`packages/opencode/src/tool/shell.ts:409`).
  - Directory-change commands (`CWD` set: `cd`, `chdir`, `popd`, `pushd`, `push-location`, `set-location`) add no pattern (`packages/opencode/src/tool/shell.ts:28`, `packages/opencode/src/tool/shell.ts:407`).
  - File commands (`FILES`: `rm cp mv mkdir touch chmod chown cat` + PowerShell `*-content`/`*-item`; `CMD_FILES` for cmd.exe) have path args resolved; any outside the project add their directory to `dirs` (`packages/opencode/src/tool/shell.ts:29-56`, `packages/opencode/src/tool/shell.ts:397-405`). Args with dynamic expansion are skipped: `argPath` returns nothing when `dynamic()` sees a leading `(`/`@(`, `$(`, `${`, a backtick, or any `$` (PowerShell: any `$` except `$env:`) (`packages/opencode/src/tool/shell.ts:174-179`, `packages/opencode/src/tool/shell.ts:369-376`) — so `rm $HOME/x` adds no `external_directory` ask.
- **Ask order** (`packages/opencode/src/tool/shell.ts:262-288`): `external_directory` for all collected `<dir>/*` globs first, then one `bash` ask with all patterns and all "always" prefixes. Any pattern hitting a deny rule fails the whole call ([[permission-ruleset]]).
- **Arity dictionary** (`packages/opencode/src/permission/arity.ts`): `prefix(tokens)` takes the longest token prefix found in `ARITY` and returns `tokens.slice(0, arity)`, else the first token (`packages/opencode/src/permission/arity.ts:1-9`). 136 entries; the LLM prompt that generated it is embedded in a comment ("Flags NEVER count as tokens", "Longest matching prefix wins", examples `git checkout main` → `git checkout`, `npm install` → `npm install`).
- So "always allow" on `git checkout main` stores `git checkout *`, which (with the optional-trailing-` *` wildcard) also matches bare `git checkout`.

### v2 runtime (`packages/core/src/tool/bash.ts`)
- The v2 bash tool asserts one `bash` permission on the **whole command string** (resource and save = `input.command`) plus `external_directory` for `workdir` (`packages/core/src/tool/bash.ts:123-149`); no tree-sitter parsing and no arity prefixes — listed as parity debt ("TODO: Port tree-sitter bash / PowerShell parser-based approval reduction", "TODO: Port BashArity reusable command-prefix approvals", `packages/core/src/tool/bash.ts:62-69`). External directories named in arguments only produce an advisory warning ("this scan is advisory only", `packages/core/src/tool/bash.ts:138-141`).

## Constants
| name | value | path:line |
|---|---|---|
| arity entries | 136 | `packages/opencode/src/permission/arity.ts` (count of `"…": N,` lines) |
| "always" suffix | `" *"` | `packages/opencode/src/tool/shell.ts:409` |

## Evolution
- 2025-10-06 `2bf0e42367` "restore bash command security validation to prevent accidental directory traversal" (had been disabled by accident).
- 2026-01-01 `351ddeed91` Permission rework (#6319): per-command patterns + arity prefixes.
- 2026-01-12 `62702fbd11`: `ls *` did not match bare `ls` → trailing ` *` optional in the wildcard.
- 2026-01-30 `e7ff7143b6`: `redirected_statement` nodes (`cmd > file`) mis-parsed; "always" prefix changed from `prefix*` (matched `lsof` for `ls`) to `prefix *`.
- 2026-04-29 `d4bf70be06`: parsed syntax trees leaked memory → released after scan.
- 2026-05-03 `3f459819ba` shell-aware prompts and parsing for bash, pwsh/powershell, cmd (#20039).

## Quirks / drift
- `cat .env` through the shell is checked only as a `bash` pattern; the default `read: {*.env: ask}` rule does not apply ([[secret-guard-bypassed-by-other-tools]]). `cat`'s path arg is boundary-checked only.
- Command templates' `` !`cmd` `` expansion runs without any of this ([[side-door-input-bypasses-hooks]]).
- PowerShell aliases deliberately not normalized ("alias normalization should happen in one place later", `packages/opencode/src/tool/shell.ts:38-40`).

pi contrast: no command parsing; bash runs unchecked unless a `tool_call` extension inspects it ([[pi--tool-call-gate|pi]]).
