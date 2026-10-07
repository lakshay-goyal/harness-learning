---
type: concept
stage: tool-design
tier: candidate
aliases: [description, single-shape-tool-schema, docs-offload-from-description, shell.txt, renderPrompt, tool.definition]
harnesses: [pi, opencode]
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
- Habit-replacing parameter plus a named ban (opencode `workdir` vs `cd &&`).
- Quantize volatile facts to the coarsest useful unit (opencode websearch: year only).
- Plugin hook that rewrites any description/schema per request (opencode `tool.definition`).

## Implementations
- [[pi--tool-description-design|pi]] — short descriptions with interpolated 2000-line/50KB limits, continuation protocol in description and notices, edits[]-only schema, docs-offloaded codemode reference, stable discovery-tool descriptions.
- [[opencode--tool-description-design|opencode]] — per-shell templated description with interpolated limits and a dedicated-tool mapping; byte-stability fixes (no `${directory}`, year-only date, static skill text); stale promises remain at HEAD.

## Failures
- [[ambiguous-tool-parameter-name]]
- [[amend-after-failed-commit]]
- [[edit-tool-dual-mode-confusion]]
- [[partial-file-read-acted-on]]
- [[tool-result-misreports-facts]]
- [[tool-description-lies-about-async]]
- [[foreign-harness-tool-hallucination]]
- [[tool-description-drifts-from-implementation]]
- [[model-distrusts-preprovisioned-resource]]
- [[unrequested-git-commits]]
- [[shell-cd-chaining]]
- [[borrowed-prompt-foreign-references]]

## Related
[[tool-argument-repair]] · [[dynamic-tool-guidelines]] · [[guideline-softening]] · [[tool-output-truncation]] · [[cache-stable-prompt-prefix]] · [[harness-diagnostics-channel]] · [[self-documentation-pointer]] · [[constrained-tool-sampling]] · [[provider-identity-shim]]
