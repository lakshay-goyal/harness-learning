---
type: tradeoff
concepts: [permission-ruleset, tool-call-gate, shell-command-permission-parsing, tool-only-isolation, project-trust-gate]
harnesses: [pi, opencode]
---
# permission-prompts-vs-none

**Axis**: does the harness ask the user before risky tool calls, or run everything with the user's privileges and leave containment to the OS?

| option | pi | opencode | evidence |
|---|---|---|---|
| No approvals, YOLO by default | ✅ default | ❌ | pi: "Pi does not include a built-in permission system" (`README.md:90`); "Permission systems add massive friction while being easily circumvented" (`b172beb92`) → [[no-permission-prompts]] |
| Hook to build your own gate | ✅ `tool_call` event can block/mutate; fail-closed on throw | ✅ plugin hooks exist, but the ruleset is the primary path | pi: `packages/coding-agent/docs/extensions.md:105`; `packages/coding-agent/src/core/agent-session.ts:667-679` → [[tool-call-gate]] |
| Declarative allow/ask/deny ruleset, last match wins | example only (`permission-gate.ts` regex) | ✅ default `*: allow`, `doom_loop: ask`, `external_directory: ask`, `*.env: ask` | opencode: `packages/opencode/src/permission/index.ts:28-163`; `packages/opencode/src/agent/agent.ts:119-134`; rework `351ddeed91` 2026-01-01 → [[permission-ruleset]] |
| Shell command parsed into sub-commands for rules | ❌ | ✅ tree-sitter AST, arity-based "always" prefixes | features A8 → [[shell-command-permission-parsing]] |
| Headless behavior | no gate unless installed; example gate blocks when `!ctx.hasUI` | fail closed: every ask auto-rejected unless `--auto` (aliases `--yolo`) | pi: `examples/extensions/permission-gate.ts:20-22`; opencode: `packages/opencode/src/cli/cmd/run.ts:242-256,801-820` |
| Claimed as security? | no: "Security theater" (`3424550d2`) | no: "a UX feature … not designed to provide security isolation" (`SECURITY.md:17`) | both point at containers → [[no-sandbox]] |
| Gate on loading repo config | ✅ project trust (`packages/coding-agent/src/core/trust-manager.ts:30-39`) | ❌ ([[no-project-trust-gate]]) | — |

**Where they agree**: neither treats prompts as a security boundary. Both say "run in a container" for isolation.

**When each wins**
- **No prompts (pi)**: an attentive human at the terminal, a disposable environment (container/VM), or an embedder who wants to own policy. Zero friction, zero false positives, no "always allow" fatigue.
- **Ruleset (opencode)**: shared or long-running sessions, headless CI where an unknown action must fail closed, and teams that want policy as config (e.g. `bash: {"git *": allow, "*": ask}`). The extra machinery pays for itself only if users read the prompts; "always" scopes (`read: ["*"]` on one approval) erode it.
- Hybrid lesson: approvals guard **actions**, trust gates guard **inputs**. pi has only the second, opencode only the first.

Related: [[no-permission-prompts]] · [[no-sandbox]] · [[secret-handling]] · [[cwd-confinement-vs-none]] · [[pi]] · [[opencode]] · [[Tradeoffs]]
