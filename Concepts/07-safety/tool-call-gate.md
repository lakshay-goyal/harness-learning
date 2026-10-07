---
type: concept
stage: permissions
tier: candidate
aliases: [tool_call, beforeToolCall, "{block: true}", before_tool, beforeTool, user_bash, tool-call-interception, tool-call-hooks, tool-call-gate-hook, tool.execute.before, "permission.ask hook"]
harnesses: [pi, opencode]
---
Pre-execution hook on every tool call (after arg validation, before execute) that can rewrite arguments or block the call with a reason fed back to the model; the single seam where permission/approval policies live, failing closed when the policy itself errors.

## Why
- Without one choke point, policy plugins must wrap every tool individually and miss paths: pi's wrapper era missed custom tools, RPC inputs and saw stale state ([[side-door-input-bypasses-hooks]], [[pre-tool-hook-sees-stale-state]]).
- A policy hook that crashes must not silently allow the action: pi's remote-exec `user_bash` routing hook fell back to the local shell on error ([[hook-error-fails-open]]).
- Harnesses with no built-in approvals (pi: "No permission popups") still need a place for opt-in gates (confirm-dangerous-bash, protected paths, plan mode, OS sandbox wrapping).

## Design space
- **Built-in approval prompts per tool / per command class** (Claude Code, Codex approval modes). pi rejected: "Permission systems add massive friction while being easily circumvented" (`b172beb92`), "Security theater" (`3424550d2`). See [[no-permission-prompts]].
- **Wrapper interception** (decorate each tool's `execute`) — pi pre-2026-03; replaced by loop-level hook (`63ac2df24`).
- **Loop-level pre-call hook returning `{block, reason}`** — pi's choice (`beforeToolCall` in agent core, surfaced as the extension `tool_call` event).
- **Argument rewriting**: in-place mutation, no re-validation (pi core) vs. rewrite-then-revalidate (pi-durable `beforeTool` chain).
- **Handler error semantics**: fail-open (log + continue) vs fail-closed (block) — pi: fail-closed for `tool_call` and (since `509ee2bd0`) `user_bash`.
- **Combination**: first block wins and short-circuits (pi) vs. all handlers vote.
- **Coverage**: model calls only vs. also nested programmatic calls (code-mode) and user shell commands — pi covers nested calls via `parentToolCallId` and user `!cmd` via a separate `user_bash` hook.
- **Static rule config** (allow/deny lists in settings) — absent in pi core; only example extensions with regex lists.
- **Human wait budget**: hook timeouts (pi had them, removed `88e39471e`) vs. unbounded + abort signal.
- **Policy inputs**: tool name + args only, or declarative tool metadata ([[tool-safety-annotations]]).
- **Declarative rules as the primary gate, hook secondary** (opencode: [[permission-ruleset]] asks inside each tool; plugin `tool.execute.before` runs first).
- **Hook without veto result**: in-place `output.args` mutation, blocking only by throwing (opencode `tool.execute.before`).
- **No registry-level gate**: trusted leaf tools sequence their own permission requests (opencode v2, `specs/v2/tools.md:131`).

## Implementations
- [[pi--tool-call-gate|pi]] — `beforeToolCall` in agent-core → extension `tool_call` event; first block wins; throw = block; no re-validation; nested codemode calls gated too; no built-in policy, only examples.
- [[opencode--tool-call-gate|opencode]] — built-in [[permission-ruleset]] asks inside each tool; plugin `tool.execute.before` mutates args (throw = block, inferred); `permission.ask` decision hook dead since 2026-03; v2 has no tool hook.

## Failures
- [[hook-error-fails-open]]
- [[side-door-input-bypasses-hooks]]
- [[pre-tool-hook-sees-stale-state]]
- [[declined-action-retried]]
- [[dead-hook-in-public-api]]

## Tradeoffs
- [[plan-mode-vs-none]]
- [[permission-prompts-vs-none]]

## Related
[[tool-safety-annotations]] · [[project-trust-gate]] · [[tool-only-isolation]] · [[extension-event-hooks]] · [[tool-error-as-result]] · [[tool-result-rewriting]] · [[nested-tool-calls]] · [[code-mode]] · [[parallel-tool-execution]] · [[no-permission-prompts]] · [[no-sandbox]] · [[no-plan-mode]] · [[permission-ruleset]] · [[shell-command-permission-parsing]]
