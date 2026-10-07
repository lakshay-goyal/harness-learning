---
type: concept
stage: tools
tier: candidate
aliases: [grep tool, find tool, ls tool, grep.ts, find.ts, ls.ts, ensureTool, tools-manager, PI_OFFLINE, gitignore-aware-search, managed-binary-bootstrap, argv-separator-hardening]
harnesses: [pi]
---
Dedicated content/file/directory search tools built on fast external binaries (ripgrep, fd): gitignore-aware, result-capped, argv-hardened against flag injection, with the binaries bootstrapped by the harness when missing.

## Why
- Unbounded `grep -r` / `find` output floods context; ignore rules (node_modules, build dirs) must apply.
- Model-supplied patterns starting with `-` become flags of the backend CLI ([[search-tool-argument-injection]]).
- Backends have their own glob/ignore semantics that differ from what models write ([[find-glob-semantics-mismatch]], [[search-ignore-rules-misapplied]]).
- Per-match synchronous file reads stall broad searches ([[search-tool-stalls-on-broad-queries]]).
- Auto-downloading binaries fails behind NAT/CI rate limits, on new platforms, concurrently ([[search-binary-bootstrap-failures]]).

## Design space
- **No dedicated tools; model uses rg/find via shell** (pi default) vs dedicated tools on by default.
- Dedicated tools opt-in (pi: grep/find/ls off by default since added in 2025-11).
- Backend: ripgrep/fd (pi) vs language-native walker vs index/embeddings ([[no-codebase-index]]).
- Binary provisioning: system PATH → harness bin dir → download latest release (pi, with offline switch) vs bundle vs require install.
- Result caps (matches/results/entries) + byte cap + per-line cap with actionable "use limit=N" notice (pi).
- `--` end-of-options before untrusted positionals (pi).
- Hierarchical gitignore incl. nested repos (pi delegates to fd/rg, `--no-require-git` only outside repos).

## Implementations
- [[pi--search-tools|pi]] — `grep` (rg --json, 100 matches, 500-char lines), `find` (fd --glob, 1000 results, full-path for `/` patterns), `ls` (readdir, 500 entries, not gitignore-aware); rg/fd auto-downloaded into pi bin dir.

## Failures
- [[search-tool-argument-injection]]
- [[find-glob-semantics-mismatch]]
- [[search-ignore-rules-misapplied]]
- [[search-tool-stalls-on-broad-queries]]
- [[search-binary-bootstrap-failures]]

## Related
[[minimal-default-toolset]] · [[shell-execution]] · [[tool-output-truncation]] · [[path-normalization]] · [[supply-chain-pinning]]
