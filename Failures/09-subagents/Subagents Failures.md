---
type: group
group: 09-subagents
---
Failures whose first concept is in [[Subagents]].

- [[subagent-output-truncated]] — Parallel subagents returned 100-char previews to the parent model, losing results and diagnostics.
- [[subagent-config-not-inherited]] — Subagents ignored parent model/thinking/tools; array-form `tools` rejected; wrong agents dir.
- [[subagent-prompt-leaks-host-paths]] — Child invocation leaked Bun virtual-FS paths / used a different `pi` build.
- [[ownership-cancellation-races]] — Durable ownership-tree abort cascades raced with completion, orphaning and reopen.

Related cross-group: [[non-idempotent-tool-replayed-after-crash]] (background subagent stop replayed after crash, 03-tools) · [[parallel-side-requests-single-slot-provider]] (05-context) · [[proxied-stream-option-loss]] (01-loop).
