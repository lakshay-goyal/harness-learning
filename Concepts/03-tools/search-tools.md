---
type: concept
stage: tools
tier: must-have
aliases: [grep tool, find tool, ls tool, grep.ts, find.ts, ls.ts, ensureTool, tools-manager, PI_OFFLINE, gitignore-aware-search, managed-binary-bootstrap, argv-separator-hardening, "glob tool", Ripgrep.Service]
harnesses: [pi, opencode]
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
- codex: absent — no grep/find/ls tools; the model runs `rg` / `rg --files` through `exec_command` (prompt `codex-rs/core/gpt_5_2_prompt.md:250`) and [[shell-command-intent-parsing]] recovers Search/ListFiles intents. Experimental `grep_files` (`f52320be86` 2025-10-08) and `list_dir` (`226215f36d` 2025-10-07) were removed `178c3b15b4` 2026-03-25 / `70807730f5` 2026-05-05 ("nothing in the current model catalog advertises it via `experimental_supported_tools`"). Axis: [[minimal-vs-rich-toolset]]. `codex-rs/file-search` (nucleo fuzzy filename finder, default limit 20, `codex-rs/file-search/src/lib.rs:130`) serves only the user's @-mention picker, not the model.
- Dedicated tools on by default with shell-search discouraged in the shell description (opencode).
- Directory listing folded into the read tool (opencode, `list` removed).
- Hidden path segments excluded from broad search (opencode v2).

## Implementations
- [[pi--search-tools|pi]] — `grep` (rg --json, 100 matches, 500-char lines), `find` (fd --glob, 1000 results, full-path for `/` patterns), `ls` (readdir, 500 entries, not gitignore-aware); rg/fd auto-downloaded into pi bin dir.
- [[opencode--search-tools|opencode]] — `glob` + `grep` on by default over system or downloaded ripgrep 15.1.0, 100-result caps, 2000-char lines; no ls (read lists directories).

## Failures
- [[search-tool-argument-injection]]
- [[find-glob-semantics-mismatch]]
- [[search-ignore-rules-misapplied]]
- [[search-tool-stalls-on-broad-queries]]
- [[search-binary-bootstrap-failures]]

## Related
[[minimal-default-toolset]] · [[shell-execution]] · [[tool-output-truncation]] · [[path-normalization]] · [[supply-chain-pinning]] · [[shell-command-intent-parsing]] · [[minimal-vs-rich-toolset]] · [[no-file-read-write-tools]]
