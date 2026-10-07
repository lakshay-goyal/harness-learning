---
type: group
group: 03-tools
---
Scope: which tools a harness exposes and how they are designed, described, validated, executed (parallel, sandboxed, nested, durable) and how their results are shaped — from the built-in read/shell/edit/write set to MCP and code-mode.

## Concepts
- [[minimal-default-toolset]] — Ship a tiny general toolset (read/bash/edit/write) on by default; specialized tools opt-in via allowlist or `+x/-x` modifiers.
- [[tool-description-design]] — Descriptions state limits interpolated from constants, the recovery path, and one input shape; long reference offloaded to docs.
- [[tool-argument-repair]] — Pre-validation shim coercing common malformed argument shapes into the canonical schema.
- [[tool-error-as-result]] — Every tool failure (unknown, invalid args, blocked, thrown) becomes an isError result fed back to the model.
- [[parallel-tool-execution]] — Execute a turn's tool calls concurrently (sequential preflight, ordered results) unless a tool opts out.
- [[per-file-mutation-queue]] — Serialize mutating tools per canonical path while other calls run in parallel.
- [[search-replace-edit]] — Edit by exact old→new string with uniqueness, multi-edit against original, CRLF/BOM preservation, diff output.
- [[fuzzy-edit-matching]] — Fallback match after normalization (whitespace, quotes, dashes, NFKC) when exact fails; rewrite only touched lines.
- [[shell-execution]] — Shell tool mechanics: shell selection, bounded streaming, optional timeout, exit idle grace, env injection, no background jobs.
- [[file-read-tool]] — Read tool: offset/limit paging, image detection/resize, bounded scan, encoding handling.
- [[search-tools]] — grep/find/ls on ripgrep/fd: gitignore-aware, capped, auto-downloaded binaries, `--` argv hardening.
- [[path-normalization]] — Normalize model/user path quirks (@, ~, unicode spaces, macOS screenshot names, MSYS drives) against session cwd.
- [[deferred-tool-loading]] — Tools registered but not declared; exposure tiers; model discovers them via a search tool and they load next call.
- [[code-mode]] — Model writes a script calling tools inside a sandbox; only script output enters context.
- [[mcp-integration]] — How MCP servers/tools are surfaced: lazy connect, name sanitization, exposure, result truncation, OAuth.
- [[nested-tool-calls]] — Tools invoking tools through the same validation/hook pipeline with bounded provenance.
- [[tool-result-rewriting]] — Post-execution chained hooks patch tool output (redaction, augmentation, normalization).
- [[structured-tool-output]] — Separate model-facing content from machine-facing structured results; terminating submit tools.
- [[pluggable-tool-backends]] — Tools delegate I/O to swappable operations (local / SSH / VM / remote env).
- [[crash-safe-tool-replay]] — Effect sandwich: commit intent → effect → outcome; per-tool replay policy after a crash.

Failures: [[Tools Failures]]

Adjacent: [[tool-output-truncation]] · [[tool-output-spill]] · [[image-normalization]] (05) · [[tool-call-gate]] · [[process-tree-kill]] · [[tool-safety-annotations]] (07) · [[constrained-tool-sampling]] · [[streaming-json-repair]] (02) · [[dynamic-tool-guidelines]] (04) · [[plugin-tools]] (10)
