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
- [[tool-name-semantics-misfire]] — (codex) `tool_suggest` called when `tool_search` was needed; fixed by renaming to `request_plugin_install` and using "connector" vocabulary.
- [[windows-destructive-cross-shell]] — (codex) PowerShell-enumerated paths deleted via `cmd /c`; visible `Start-Process` windows; rules keyed to the executor platform.
- [[invented-schema-constraint]] — (codex) sanitizer typed unknown MCP schema nodes as `string`, steering arguments wrong; now `{}`.
- [[ambiguous-tool-error-causes-retry-loop]] — one edit error for "not found" and "multiple" caused a doom editing loop (opencode).
- [[tool-description-drifts-from-implementation]] — descriptions promised `run_in_background`, persistent shell, read-before-edit checks the code lacked (opencode).
- [[model-distrusts-preprovisioned-resource]] — model re-created a pre-approved tmp dir until told it already exists (opencode).
- [[empty-success-output-read-as-failure]] — silent edit success read as failure; edits re-applied (opencode).

## Edit
- [[edit-invisible-character-mismatch]] — exact anchors failed on CRLF, BOM, smart quotes, dashes, NFKC variants.
- [[fuzzy-edit-rewrites-untouched-lines]] — fuzzy match wrote the normalized buffer back, rewriting the whole file.
- [[concurrent-file-mutation-interleave]] — parallel same-file edits overwrote each other.
- [[alternate-edit-tool-skips-edit-pipeline]] — `apply_patch` missed permission filter/events wired by tool name (opencode).
- [[edit-oldstring-drops-lines]] — multi-line `newString` silently omitted lines (opencode, Meta prompt rule).

## Patch envelope (codex `apply_patch`)
- [[patch-wrapped-in-heredoc-by-model]] — GPT-4.1 wrapped patches in `<<'EOF'` heredocs; lenient parser strips them for all models.
- [[tool-name-training-artifact]] — models called `applypatch`; harness aliases it.
- [[patch-body-executed-as-shell]] — raw patch run as bash created files named `,` / `{`; now refused with the corrected invocation.
- [[edit-rewrites-line-endings]] — applier normalized files to LF; endings now preserved unconditionally.
- [[partial-multi-file-patch-untracked]] — failed multi-file patch left real changes the turn diff dropped; exact committed prefix now reported.
- [[duplicate-path-ops-in-one-patch]] — `a` and `./a` in one patch accepted; duplicate resolved paths rejected.
- [[apply-patch-path-and-permission-hazards]] — patch permission derivation widened sandbox write roots.

## Shell
- [[bash-spawn-errors-crash-session]] — missing cwd/shell raised uncaught ENOENT and crashed the session.
- [[signal-killed-command-reported-success]] — signal-killed commands reported as success.
- [[bash-descendants-hang-or-lose-output]] — abort left children running; Windows hung forever; late output truncated.
- [[bash-output-integrity]] — binary crash, split UTF-8, missing spill on line truncation, wrong line counts.
- [[bash-timeout-clamped-to-immediate]] — huge/invalid model timeouts fired immediately.
- [[windows-process-tree-and-shells]] — pwsh silent with detached, WSL `$VAR` not expanded, taskkill PATH crash, drive paths.
- [[premature-backgrounding]] — (codex) Windows yield returned before PowerShell-wrapped commands started: +20.7 % tool calls/turn; yield floor 2 s → 10 s.
- [[stale-background-process-status]] — (codex) polls of exited processes emitted events with the old session id; `/ps` and status line disagreed.
- [[tool-output-bypasses-truncation]] — (codex) untruncated exec paths and multi-hundred-MB MCP results overflowed context and rollouts.
- [[shell-waits-on-inherited-stdin]] — interactive commands and stdin-reading `rg` hung forever (opencode).
- [[shell-cd-chaining]] — `cd <dir> &&` before every command until a `workdir` param + ban (opencode).
- [[model-truncates-command-output]] — model piped through head/tail although output was spilled (opencode).
- [[unrequested-git-commits]] — model committed without being asked (opencode).
- [[amend-after-failed-commit]] — `--amend` after a hook-rejected commit rewrote the previous commit (opencode).
- [[background-job-without-observation-tool]] — async jobs without push completion/cancel; removed in v2 (opencode).

## Search, read & paths
- [[find-glob-semantics-mismatch]] — path globs returned nothing; root searches dropped a character.
- [[search-ignore-rules-misapplied]] — sibling/nested-repo files hidden by misapplied .gitignore.
- [[search-tool-stalls-on-broad-queries]] — grep stalled on per-match reads; find not cancellable.
- [[search-tool-argument-injection]] — patterns starting with `-` parsed as rg/fd flags.
- [[search-binary-bootstrap-failures]] — rg/fd download crashed, raced, 404'd, or hit GitHub API 403 every launch.
- [[read-path-unicode-variants]] — macOS screenshot paths with invisible characters not found.
- [[tools-ignore-session-cwd]] — tools used process/creation-time cwd instead of session cwd.
- [[read-fails-on-growing-file]] — durable read rejected actively appended logs.
- [[silent-eof-causes-paging-loop]] — no end-of-file marker; model paged forever (opencode).
- [[binary-content-poisons-transcript]] — binary reads corrupted sessions (opencode).
- [[read-offset-index-mismatch]] — 0-based offset vs 1-based displayed lines (opencode).
- [[tiny-repeated-read-slices]] — 30-line read chunks (opencode).

## Parallelism, hooks & durability
- [[hook-throw-aborts-parallel-batch]] — a throwing afterToolCall hook aborted all sibling tools.
- [[parallel-tool-results-order]] — fast tools stayed pending; risk of out-of-order persisted results.
- [[interactive-tool-in-parallel-batch]] — parallel human-question tools became unanswerable.
- [[tool-result-hook-patches-lost]] — multiple tool_result handlers: last one won.
- [[non-idempotent-tool-replayed-after-crash]] — replayed subagent control tool could stop newer work.
- [[unannotated-mcp-tools-serialized]] — (codex) read-only MCP tools ran serially; `readOnlyHint` now the concurrency signal (distrusted when cached).
- [[parallel-tools-reveal-latent-bugs]] — (codex) enabling parallel calls took months of ordering, approval-scope and shim fixes.

## MCP, deferred tools & code-mode
- [[mcp-connect-timeout-too-short]] — 5 s MCP connect timeout failed slow stdio servers on cold start; raised to 30 s.
- [[mcp-tool-name-collision]] — `-` vs `_` names mapped scripts to the wrong MCP tool.
- [[mcp-startup-blocks-and-description-churn]] — first prompt blocked on servers; descriptions changed as servers connected.
- [[mcp-child-processes-orphaned]] — stdio servers and their children survived exit; reconnect leaks (opencode).
- [[mcp-error-result-treated-as-success]] — `isError` ignored; structured JSON hid the readable text (opencode).
- [[tool-allowlist-hides-mcp-tools]] — `--tools` allowlist removed MCP tools from codemode's reach.
- [[deferred-tools-lost-on-resume]] — tool_search-loaded tools dropped on resume before servers reconnected.
- [[codemode-sandbox-escape-to-host]] — print loops OOM'd the host; patched built-ins crashed the bridge.
- [[read-path-traversal]] — read tool escaped cwd; confinement added in 0.11.7, later dropped.

Elsewhere but tool-related: [[interrupt-kills-background-processes]] · [[approval-scope-too-broad-in-parallel-batch]] · [[harness-credential-leaks-to-tools]] · [[mcp-annotation-defaults-unsafe]] · [[grammar-constrained-tool-instability]] · [[line-separator-breaks-jsonl-framing]] · [[truncation-budget-drift-on-replay]] · [[edits-bypass-patch-tool]] · [[oauth-refresh-token-rotation-lost]] (MCP OAuth fix appended) · [[strict-tool-schema-rejections]] · [[malformed-tool-json-crashes]] · [[placeholder-text-misleads-model]] · [[image-content-poisoning]] · [[length-truncated-tool-calls-executed]]

Back: [[Tools]]

## Web, todo, question, LSP (opencode-only tools)
- [[bot-detection-blocks-fetch]] — spoofed Chrome UA challenged by Cloudflare; honest-UA retry.
- [[model-assumes-training-year]] — searches used the training-era year.
- [[todo-tool-usage-calibration]] — todo cadence toggled per model family (qwen, GPT, Claude).
- [[duplicate-catch-all-option]] — model added "Other" next to the UI's automatic custom answer.
- [[stale-diagnostics-after-edit]] — partial/warning diagnostics sent the model chasing fixed errors.
