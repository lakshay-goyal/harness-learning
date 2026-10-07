---
type: concept
stage: tool-design
tier: candidate
aliases: [outputSchema, structuredContent, details, "terminate: true", terminating-submit-tool, durationMs, toModelOutput]
harnesses: [pi, opencode]
---
Separate what a tool result tells the model (`content`) from machine-facing data: a schema'd structured value for programmatic callers, UI/extension `details`, and control hints such as "terminate the run after this batch" for submit-style tools.

## Why
- Programmatic callers (code-mode scripts, evals, SDK) need typed data, not prose; the model needs short, truncated text.
- Failures that still carry data (non-zero exit with output) must reach scripts while reading as errors to the model.
- Structured-output agents need a way to end the run with a typed verdict without an extra model turn.

## Design space
- Text only (baseline) vs **content + structuredContent with outputSchema** (pi, MCP-aligned) vs JSON-in-text.
- Different limits for model vs scripts (pi bash: 50KB to model, 1 MiB to scripts; MCP: middle-truncated 20KB vs full).
- `details` channel for renderers/state (pi; also used for branch-scoped state → [[branch-scoped-extension-state]]).
- Run termination: tool hint honored only if every result in the batch agrees (pi `terminate`) vs single-tool stop; durable also has `handoff` control.
- Persist control hints vs runtime-only (pi: runtime-only).
- Forced terminating tool via `toolChoice: required` on every step (opencode).
- Prefer human-readable MCP `content` over `structuredContent` for the model (opencode).

## Implementations
- [[pi--structured-tool-output|pi]] — `outputSchema`/`structuredContent` on AgentTool, bash/read/MCP structured values consumed by codemode; `terminate:true` unanimity; `submit_documentation_audit`-style terminating tools.
- [[opencode--structured-tool-output|opencode]] — `json_schema` requests add a `StructuredOutput` tool + `toolChoice: required` and break on capture; tool `metadata` is UI-only; v2 `Tool.make` codec with pure `toModelOutput`; MCP `content` preferred over `structuredContent`.

## Failures
- (none recorded specific to this concept)
- [[empty-success-output-read-as-failure]]

## Related
[[code-mode]] · [[tool-error-as-result]] · [[tool-result-rewriting]] · [[turn-loop]] · [[constrained-tool-sampling]] · [[harness-evals]] · [[branch-scoped-extension-state]]
