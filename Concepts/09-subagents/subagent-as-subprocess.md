---
type: concept
stage: subagents
tier: candidate
aliases: [subagent example, "examples/extensions/subagent", scout, planner, reviewer, worker, "--mode json -p --no-session", subagent-role-prompts, handoff-output-contract, "{previous}", agentScope]
harnesses: [pi]
---
Delegation by spawning the same CLI headless (machine-readable event stream, no session) with a role prompt and tool/model config; the parent model receives only the child's final text under an explicit output contract.

## Why
- Process isolation = context isolation: the child explores without polluting the parent window; parallel children give fan-out.
- The parent only knows what the handoff string says: truncated or preview-only results lose work and failure diagnostics ([[subagent-output-truncated]]).
- Context transfer across agent boundaries is lossy — the stated reason pi refuses built-in subagents ([[no-subagents-core]]); role prompts that write "for a reader who has NOT seen the files" mitigate it.
- Spawn-time config (model, thinking, tools, binary path) must be inherited deliberately ([[subagent-config-not-inherited]], [[subagent-prompt-leaks-host-paths]]).

## Design space
- **Mechanism**: subprocess of same CLI (pi example) · in-process child conversation owned by a task ([[task-owned-subagent]]) · human-orchestrated parallel sessions/tmux (pi's stated default) · tool-level orchestration via nested calls/code mode ([[nested-tool-calls]], [[code-mode]]).
- **Modes**: single · parallel with concurrency cap · chain with `{previous}` substitution.
- **Result contract**: last assistant text (pi) with per-task byte cap; full trace kept in tool details for UI only.
- **Role definition**: markdown + frontmatter (tools allowlist, model) appended to system prompt; workflow prompt templates chaining roles.
- **Trust**: repo-supplied agent definitions gated by confirmation / project trust ([[project-trust-gate]]).
- **Config inheritance**: explicit model wins; otherwise inherit dispatcher model + thinking.
- **Recursion**: unbounded (pi example; child loads same extensions unless restricted) · stripped in child (pi-durable).

## Implementations
- [[pi--subagent-as-subprocess|pi]] — example extension `subagent` tool spawning `pi --mode json -p --no-session`, scout/planner/reviewer/worker roles, 8 parallel / 4 concurrent / 50 KiB per task.

## Failures
- [[subagent-output-truncated]]
- [[subagent-config-not-inherited]]
- [[subagent-prompt-leaks-host-paths]]

## Tradeoffs
- [[builtin-subagents-vs-none]]

## Related
[[task-owned-subagent]] · [[session-handoff]] · [[headless-rpc-mode]] · [[agent-event-stream]] · [[system-prompt-override]] · [[prompt-template-expansion]] · [[plugin-tools]] · [[abort-propagation]] · [[no-subagents-core]]
