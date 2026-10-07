---
type: absence
harnesses: [codex]
---
# no-typescript-cli

The original Node/TypeScript CLI was deleted; the npm package ships the native Rust binary.

**What's missing**
- `codex-cli/` now only `bin/`, `scripts/`, `package.json` (`codex-cli/bin/codex.js` launches per-platform binaries).

**Evidence of decision**
- Era 0 TS CLI `59a180ddec` 2025-04-16 (Ink TUI, `75febbdefa:codex-cli/src/utils/agent/agent-loop.ts`, approval modes suggest/auto-edit/full-auto); Rust import `31d0d7a305` 2025-04-24; npm ships Rust via `--native` `73fe1381aa` 2025-05-12; "chore: remove the TypeScript code from the repository (#2048)" `408c7ca142` 2025-08-08 (v0.20.0).

**Implication**
- TS era lasted ~4 months; agent loop, sandbox and TUI rebuilt in Rust (rationale for the rewrite not stated in commit bodies — unverified; sandbox integration and single-binary distribution are the visible beneficiaries).

Related: [[harness-package-distribution]] · [[codex--harness-package-distribution|codex]] · [[terminal-scrollback-tui]] · [[Absences]]
