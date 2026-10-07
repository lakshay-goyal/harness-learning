---
type: concept
stage: tool-design
tier: candidate
aliases: [description, single-shape-tool-schema, docs-offload-from-description, ModelMessages.tools, *_spec.rs]
harnesses: [pi, codex]
---
Tool descriptions state their limits (interpolated from the same constants the code uses), the recovery path when output is cut, and exactly one input shape; long reference material is offloaded to docs the model reads on demand.

## Why
- Models act on what the description says: a description that doesn't mention continuation makes them act on partial files ([[partial-file-read-acted-on]]); one that misstates async-ness yields `{}` results ([[tool-description-lies-about-async]]).
- Two alternative input shapes in one schema cause repeated invalid calls and retries ([[edit-tool-dual-mode-confusion]]).
- Numbers in results/descriptions are reasoned from — wrong ones mislead ([[tool-result-misreports-facts]]).
- Descriptions that change when dynamic tools appear churn the prompt cache ([[mcp-startup-blocks-and-description-churn]]).
- Models trained on another harness call that harness's tool names ([[foreign-harness-tool-hallucination]]).

## Design space
- Limits hard-coded in prose vs **interpolated from constants** (pi, since 2025-12).
- Recovery instructions in description vs in the truncation notice vs both (pi: both — "continue with offset", "Use offset=N to continue", "Full output: <path>").
- One shape per tool + code-side legacy repair (pi) vs alternative shapes (pi tried for one day, reverted).
- Full API reference inline vs one line per capability + doc pointer + error messages that "point the way back" (pi codemode, ~5.3k→3.3k tokens).
- Byte-stable descriptions with volatile facts in patchable prompt sections (pi) vs descriptions listing live tools/servers.
- Separate prompt channels: API description vs system-prompt snippet/guideline (pi: three channels, snippets opt-in) → [[dynamic-tool-guidelines]].
- Out-of-band harness diagnostics instead of in-content notices (pi durable) → [[harness-diagnostics-channel]].
- Telling the model which foreign tools don't exist (pi's short-lived Codex bridge prompt) vs aliasing them.
- One-line descriptions + behavioural policy in the per-model system prompt (✔ codex; `30ee24521b` moved update_plan guidance out) — but long policy text inside descriptions of *optional* tools (✔ codex spawn_agent, goal, `request_plugin_install`).
- Catalog-owned descriptions replaceable per model, bundled text as fallback (✔ codex `ModelMessages.tools`, `codex-rs/protocol/src/openai_models.rs:585-597`).
- Limits as literals in parameter docs (✔ codex `shell_spec.rs:30-61`, "Defaults to 10000 ms; effective range is 250-30000 ms") vs interpolated from constants (✔ pi).
- Description and runtime availability driven by one policy (✔ codex: request_user_input text templated from allowed modes, `998eb8f32b`).
- Platform-keyed hazards appended only where the tool executes (✔ codex Windows rules keyed to the executor, `5ed294d49d`).
- Fix a misfiring tool by renaming it after its side effect and using the model's own vocabulary (✔ codex `tool_suggest` → `request_plugin_install`, "connector" wording).
- Grammar as the format spec instead of prose (✔ codex freeform `apply_patch`, 75-line instructions deleted `8d637ae398`).
- Alias the trained misspellings (✔ codex `applypatch`) vs deny foreign names in the prompt (✔ pi bridge, later dropped).

## Implementations
- [[pi--tool-description-design|pi]] — short descriptions with interpolated 2000-line/50KB limits, continuation protocol in description and notices, edits[]-only schema, docs-offloaded codemode reference, stable discovery-tool descriptions.
- [[codex--tool-description-design|codex]] — terse one-liners, exact parameter defaults/ranges (`1c55bb2702`), catalog-overridable text, mode-scoped and platform-keyed descriptions, renames to fix misfires.

## Failures
- [[edit-tool-dual-mode-confusion]]
- [[partial-file-read-acted-on]]
- [[tool-result-misreports-facts]]
- [[tool-description-lies-about-async]]
- [[foreign-harness-tool-hallucination]]
- [[tool-name-semantics-misfire]]
- [[windows-destructive-cross-shell]]
- (02) [[malformed-tool-json-crashes]]
- (05) [[prompt-states-stale-harness-limits]]
- (09) [[orchestrator-busy-polls-subagents]]

## Related
[[tool-argument-repair]] · [[dynamic-tool-guidelines]] · [[guideline-softening]] · [[tool-output-truncation]] · [[cache-stable-prompt-prefix]] · [[harness-diagnostics-channel]] · [[self-documentation-pointer]] · [[constrained-tool-sampling]] · [[provider-identity-shim]] · [[per-model-system-prompt]] · [[patch-envelope-edit]] · [[plan-checklist-tool]] · [[structured-user-question-tool]] · [[dedicated-vs-shell-tools]]
