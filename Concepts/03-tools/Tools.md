---
type: group
group: 03-tools
---
Scope: which tools a harness exposes and how they are designed, described, validated, executed (parallel, sandboxed, nested, durable) and how their results are shaped — from the built-in read/shell/edit/write set to MCP and code-mode.

## Concepts
- [[minimal-default-toolset]] — Ship a tiny general toolset (read/bash/edit/write) on by default; specialized tools opt-in via allowlist or `+x/-x` modifiers (codex contrast: per-turn feature/model-gated plan, shell + patch core).
- [[tool-wire-kinds]] — Several tool wire types (JSON function, grammar-constrained freeform "custom", namespace, provider-hosted, client-executed search) routed by payload kind.
- [[tool-schema-normalization]] — Sanitize foreign (MCP/dynamic) schemas into the supported subset (widen, never narrow) and compact oversized ones in lossy passes.
- [[tool-description-design]] — Descriptions state limits interpolated from constants, the recovery path, and one input shape; long reference offloaded to docs.
- [[tool-argument-repair]] — Pre-validation shim coercing common malformed argument shapes into the canonical schema.
- [[tool-error-as-result]] — Every tool failure (unknown, invalid args, blocked, thrown) becomes an isError result fed back to the model.
- [[parallel-tool-execution]] — Execute a turn's tool calls concurrently (sequential preflight, ordered results) unless a tool opts out.
- [[per-file-mutation-queue]] — Serialize mutating tools per canonical path while other calls run in parallel.
- [[search-replace-edit]] — Edit by exact old→new string with uniqueness, multi-edit against original, CRLF/BOM preservation, diff output.
- [[fuzzy-edit-matching]] — Fallback match after normalization (whitespace, quotes, dashes, NFKC) when exact fails; rewrite only touched lines.
- [[patch-envelope-edit]] — Edit via one self-delimiting multi-file patch DSL (Add/Delete/Update/Move, `@@` anchors), verified in memory before sequential apply.
- [[shell-execution]] — Shell tool mechanics: shell selection, bounded streaming, optional timeout, exit idle grace, env injection, no background jobs (codex contrast: yield/poll sessions that persist as background terminals).
- [[shell-environment-snapshot]] — Capture the login shell's functions/aliases/exports once, source the snapshot in a cheap non-login shell per command.
- [[shell-command-intent-parsing]] — With shell-only reading, parse each command into read/list/search intents for UI, approvals and telemetry.
- [[user-shell-escape]] — User-typed `!cmd` runs outside the model tool path; output recorded as a tagged context fragment for the next request.
- [[file-read-tool]] — Read tool: offset/limit paging, image detection/resize, bounded scan, encoding handling (codex: image viewer only).
- [[search-tools]] — grep/find/ls on ripgrep/fd: gitignore-aware, capped, auto-downloaded binaries, `--` argv hardening.
- [[web-search-tool]] — Internet search as a provider-hosted tool or a client-executed proxied tool, with cached / indexed / live / disabled modes.
- [[plan-checklist-tool]] — No-op `update_plan` tool that only publishes a step checklist to the UI; plan state lives in the transcript.
- [[structured-user-question-tool]] — Blocking tool that asks the user 1–3 multiple-choice questions and returns the answers as the result.
- [[wall-clock-tools]] — Namespaced `clock` tools: read current time; sleep that wakes on new user input.
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

Tradeoffs: [[edit-format]] · [[dedicated-vs-shell-tools]]

Adjacent: [[tool-output-truncation]] · [[tool-output-spill]] · [[image-normalization]] (05) · [[tool-call-gate]] · [[process-tree-kill]] · [[tool-safety-annotations]] (07) · [[constrained-tool-sampling]] · [[streaming-json-repair]] (02) · [[dynamic-tool-guidelines]] (04) · [[plugin-tools]] (10)
