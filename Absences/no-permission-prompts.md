---
type: absence
harnesses: [pi]
---
# no-permission-prompts

YOLO by default: no per-tool approval, no command pre-screening.

**What's missing**
- "Pi does not include a built-in permission system for restricting filesystem, process, network, or credential access. By default, it runs with the permissions of the user and process that launched it." (`README.md:90`)
- "…it does not ask for approval before every tool call." (`packages/coding-agent/docs/security.md:3`)
- No allow/deny rule language and no "dangerous command" classifier in core.

**Evidence of decision**
- `b172beb92` (2025-11-12), README "## Security (YOLO by default)":
  - "This agent runs in full YOLO mode and assumes you know what you're doing."
  - Why: "Permission systems add massive friction while being easily circumvented"; "Pre-checking tools for 'dangerous' patterns introduces latency and false positives"; "Fast iteration requires trust, not sandboxing".
  - "This is how I want it to work. Use at your own risk."
- `3424550d2` (2025-12-17) Philosophy: "**No permission popups.** Security theater. Run in a container or build your own with Hooks."
- `25cc5c7bf^:packages/coding-agent/README.md:541` (softened): "**No permission popups.** Run in a container, or build your own confirmation flow with extensions inline with your environment and security requirements." The section was deleted in `25cc5c7bf`.
- HEAD reframing: "Watching the transcript, using project trust, and reviewing changes do not create a security boundary." (`packages/coding-agent/docs/security.md:7`).
- Agent-runtime design doc: "permissions and approval policy are plugin territory (`before_tool` can block or rewrite args and may wait for a person …)" (`c7eee0195:packages/agent/docs/pico/pico-work.md:284`, 2026-09-10; sentence removed next day in `4819cc877`, file deleted with the harness in `7fd478a2e`; historical only).

**Opt-in replacement**
- Gate: the `tool_call` event can mutate input or block (`packages/coding-agent/docs/extensions.md:105`) → [[tool-call-gate]].
  - Fail-closed: a throwing handler blocks the call with `Extension failed, blocking execution: …` (`packages/coding-agent/src/core/agent-session.ts:667-679`; doc `extensions.md:260`).
  - The first `block` wins and short-circuits the remaining handlers (`packages/coding-agent/src/core/extensions/runner.ts:1242-1259`).
  - Nested codemode calls pass through the same hooks with `parentToolCallId` (`extensions.md:148`; `agent-session.ts:657-674`).
- Metadata: since 0.99.0 tools carry MCP-style `annotations` (`readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`) so "a permission extension can use them". Docs include a snippet that "confirms the calls Codex asks approval for" (`extensions.md:166-175`; `docs/mcp.md:254`) → [[tool-safety-annotations]].
- Examples (`packages/coding-agent/examples/extensions/`):
  - `permission-gate.ts`: regex for `rm -rf`, `sudo`, `chmod/chown 777`; **blocks by default when `!ctx.hasUI`** (`:11,20-22`).
  - `protected-paths.ts`: substring match on `.env`, `.git/`, `node_modules/` (`:11,19`).
  - `confirm-destructive.ts`: confirms clear/switch/fork via `before_*` events.
  - `dirty-repo-guard.ts`: blocks session changes when there are uncommitted changes.
  - `tool-override.ts`: replaces a built-in tool, e.g. for access control on `read`.
- Durable variant: tool-override pattern in `packages/durable/test/examples/30-tool-override.ts` (whether it is used for policy is unverified).

**History**
- Stable since day one: "Security theater" (Dec 2025), softened to "Run in a container…" (Sep 2026), then folded into the containment framing in `docs/security.md`.
- The one built-in gate added later is [[project-trust-gate]] (`89a92207f`, 2026-06-05). It guards **input loading**, not actions.
- Prompt-level "read-only" rules were removed (`e3dd4f21d`, `b846a4bfc`); see [[no-plan-mode]].

**Implication**
- pi treats approval prompts as UX theatre and real isolation as the only boundary ([[no-sandbox]], [[tool-only-isolation]]). Core offers a hook plus metadata, not a policy.
- Headless/RPC users get no protection unless they install a gate. The example gate fails closed without a UI.

**codex** — *not absent; a core pillar*. Approval policies (`AskForApproval`) since the TS era (suggest / auto-edit / full-auto) → [[approval-policy-modes]]; execpolicy rules (crate `58f0e5ab74` 2025-04-24, v2 Starlark-ish `a941ae7632` 2025-11-17, legacy removed `656a2d0905` 2026-07-10) → [[command-rule-policy]]; Guardian LLM approval reviewer (`e84ee33cc0` 2026-03-07; 581 commits mention guardian) → [[llm-approval-reviewer]]; approvals are server→client requests in the app-server protocol ([[codex--client-server-session-split|codex client-server-session-split]]). Removed variants: `OnFailure` mode (deprecated `4668feb43a` 2026-02-12 "performs worse than `on-request`", deleted `2cf2a6a844` 2026-06-23) → [[no-on-failure-approval-mode]]; known-safe command allowlist (`942af8447b` 2026-08-19) → [[no-safe-command-allowlist]]; LLM command-risk explanation removed `c4af707e09` 2025-12-10 ("lukewarm reception during internal testing"). Headless `codex exec` forces approval `never` with the sandbox still on (`codex-rs/exec/src/lib.rs:587-592`).
**opencode contrast**: implements it: allow/ask/deny ruleset, last match wins, tree-sitter bash parsing, once/always/reject replies (`packages/opencode/src/permission/index.ts:28-163`), but `SECURITY.md:17` calls it "a UX feature", not isolation — see [[permission-ruleset]] / [[shell-command-permission-parsing]] / [[permission-prompts-vs-none]].

Related: [[tool-call-gate]] · [[tool-safety-annotations]] · [[project-trust-gate]] · [[tool-only-isolation]] · [[no-sandbox]] · [[no-prompt-injection-defense]] · [[no-cwd-confinement]] · [[Absences]] · [[approval-policy-modes]] · [[command-rule-policy]] · [[llm-approval-reviewer]] · [[sandbox-escalation-retry]] · [[no-on-failure-approval-mode]] · [[no-safe-command-allowlist]] · [[isolation-strategy]]
