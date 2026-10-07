---
type: group
group: 09-subagents
---
Failures whose first concept is in [[Subagents]].

- [[subagent-output-truncated]] — Parallel subagents returned 100-char previews to the parent model, losing results and diagnostics.
- [[subagent-config-not-inherited]] — Subagents ignored parent model/thinking/tools; array-form `tools` rejected; wrong agents dir.
- [[subagent-prompt-leaks-host-paths]] — Child invocation leaked Bun virtual-FS paths / used a different `pi` build.
- [[ownership-cancellation-races]] — Durable ownership-tree abort cascades raced with completion, orphaning and reopen.

## Delegation prompting
- [[over-eager-delegation]] — Spawned sub-agents for "be thorough" requests; fix: explicit-authorization rule with near-miss phrasings (codex).
- [[subagent-model-downgrade]] — "A mini model can solve many tasks…" made the parent pick older models for children (codex).
- [[orchestrator-busy-polls-subagents]] — Parent busy-polled `wait` with tiny timeouts → CPU/token burn; fix: 10 s floor (codex).

## Mailbox & capacity
- [[wait-misses-already-queued-result]] — `wait_agent` missed mail queued before subscribing and slept the full timeout (codex).
- [[blocking-wait-ignores-user-steer]] — A long `wait_agent` held the turn while a user steer waited (codex).
- [[subagent-eviction-loses-mail]] — LRU unloading of idle agents raced/dropped queued messages (codex).
- [[depth-capped-agent-still-has-tools]] — Max-depth agent still saw spawn tools that always failed (codex).

## Roles & review
- [[subagent-role-escalates-authority]] — Role config layers could widen permissions/providers/MCP; fix: narrowing-only allowlist (codex).
- [[review-agent-edits-code]] — `/review` child changed code; read-only sandbox broke review; fix: config-enforced lockdown (codex).
- [[side-task-result-invisible-to-parent]] — Review findings shown in UI but absent from the main agent's history (codex).

Related cross-group: [[forked-child-inherits-parent-tool-noise]] (08-state) · [[non-idempotent-tool-replayed-after-crash]] (background subagent stop replayed after crash, 03-tools) · [[parallel-side-requests-single-slot-provider]] (05-context) · [[proxied-stream-option-loss]] (01-loop).
