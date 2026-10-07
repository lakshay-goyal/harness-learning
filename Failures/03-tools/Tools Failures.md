---
type: group
group: 03-tools
---
Failures whose primary concept is in [[Tools]].

## Tool design & arguments
- [[edit-tool-dual-mode-confusion]] — dual-shape edit schema caused repeated invalid calls and retries.
- [[tool-arg-shape-drift]] — valid intents rejected (stringified/single-object edits, extra fields, nulls); models fell back to sed/python.
- [[tool-arg-coercion-breaks-unions]] — numeric strings rejected; coercion then rewrote valid nullable-union values.
- [[tool-description-lies-about-async]] — description showed async helpers as sync; scripts printed `{}`.
- [[tool-result-misreports-facts]] — write reported UTF-16 units as bytes.
- [[partial-file-read-acted-on]] — model stopped at the first truncated chunk and acted on a partial file.
- [[foreign-harness-tool-hallucination]] — Codex-trained models called `apply_patch`/`update_plan` that pi lacks.

## Edit
- [[edit-invisible-character-mismatch]] — exact anchors failed on CRLF, BOM, smart quotes, dashes, NFKC variants.
- [[fuzzy-edit-rewrites-untouched-lines]] — fuzzy match wrote the normalized buffer back, rewriting the whole file.
- [[concurrent-file-mutation-interleave]] — parallel same-file edits overwrote each other.

## Shell
- [[bash-spawn-errors-crash-session]] — missing cwd/shell raised uncaught ENOENT and crashed the session.
- [[signal-killed-command-reported-success]] — signal-killed commands reported as success.
- [[bash-descendants-hang-or-lose-output]] — abort left children running; Windows hung forever; late output truncated.
- [[bash-output-integrity]] — binary crash, split UTF-8, missing spill on line truncation, wrong line counts.
- [[bash-timeout-clamped-to-immediate]] — huge/invalid model timeouts fired immediately.
- [[windows-process-tree-and-shells]] — pwsh silent with detached, WSL `$VAR` not expanded, taskkill PATH crash, drive paths.

## Search, read & paths
- [[find-glob-semantics-mismatch]] — path globs returned nothing; root searches dropped a character.
- [[search-ignore-rules-misapplied]] — sibling/nested-repo files hidden by misapplied .gitignore.
- [[search-tool-stalls-on-broad-queries]] — grep stalled on per-match reads; find not cancellable.
- [[search-tool-argument-injection]] — patterns starting with `-` parsed as rg/fd flags.
- [[search-binary-bootstrap-failures]] — rg/fd download crashed, raced, 404'd, or hit GitHub API 403 every launch.
- [[read-path-unicode-variants]] — macOS screenshot paths with invisible characters not found.
- [[tools-ignore-session-cwd]] — tools used process/creation-time cwd instead of session cwd.
- [[read-fails-on-growing-file]] — durable read rejected actively appended logs.

## Parallelism, hooks & durability
- [[hook-throw-aborts-parallel-batch]] — a throwing afterToolCall hook aborted all sibling tools.
- [[parallel-tool-results-order]] — fast tools stayed pending; risk of out-of-order persisted results.
- [[interactive-tool-in-parallel-batch]] — parallel human-question tools became unanswerable.
- [[tool-result-hook-patches-lost]] — multiple tool_result handlers: last one won.
- [[non-idempotent-tool-replayed-after-crash]] — replayed subagent control tool could stop newer work.

## MCP, deferred tools & code-mode
- [[mcp-tool-name-collision]] — `-` vs `_` names mapped scripts to the wrong MCP tool.
- [[mcp-startup-blocks-and-description-churn]] — first prompt blocked on servers; descriptions changed as servers connected.
- [[tool-allowlist-hides-mcp-tools]] — `--tools` allowlist removed MCP tools from codemode's reach.
- [[deferred-tools-lost-on-resume]] — tool_search-loaded tools dropped on resume before servers reconnected.
- [[codemode-sandbox-escape-to-host]] — print loops OOM'd the host; patched built-ins crashed the bridge.
- [[read-path-traversal]] — read tool escaped cwd; confinement added in 0.11.7, later dropped.

Elsewhere but tool-related: [[oauth-refresh-token-rotation-lost]] (MCP OAuth fix appended) · [[strict-tool-schema-rejections]] · [[malformed-tool-json-crashes]] · [[placeholder-text-misleads-model]] · [[image-content-poisoning]] · [[length-truncated-tool-calls-executed]]

Back: [[Tools]]
