---
type: group
group: 07-safety
---
What the harness does to limit damage from model actions, repo-supplied inputs, its own supply chain, remote hosts and secrets — permissions, trust, isolation, process containment.

## Concepts

### Sandbox (kernel isolation)
- [[os-level-sandbox]] — Every model-launched process wrapped in a per-OS kernel sandbox (Seatbelt / bwrap+seccomp / Windows token or MXC) derived from one portable policy; the sandbox is the trust boundary.
- [[protected-workspace-metadata]] — Inside writable roots, `.git`/agent-config/skills/credential-helper paths stay read-only, incl. first creation, ancestor renames and pointer-resolved gitdirs.
- [[tool-only-isolation]] — Agent stays on host, tool I/O routed into VM/container/remote or a per-command sandbox; vs whole-process isolation.
- [[process-tree-kill]] — Commands in their own process group/job; kill the whole tree on abort/timeout/shutdown; remote dead-man switch.

### Approvals (who decides)
- [[approval-policy-modes]] — User/admin policy (untrusted / on-request / never / granular) deciding per action whether to ask, auto-approve or auto-reject; orthogonal to the sandbox.
- [[sandbox-escalation-retry]] — Run sandboxed first; on denial (or model request with justification) get approval and re-run widened/unsandboxed.
- [[model-requested-permissions]] — Tool/parameter for scoped sandbox widening (paths, network) for a turn or session instead of full escape.
- [[permission-state-prompt]] — Active sandbox/approval/network policy rendered into a dynamic developer message that names the escalation channel and failure signatures.
- [[approval-key-canonicalization]] — Stable approval-cache keys: single commands canonicalized, complex scripts keyed by exact text, per call id.
- [[tool-call-gate]] — Pre-execution hook that can rewrite args or block a tool call; fail-closed (pi) vs fail-open external hooks (codex).

### Policy (what is allowed)
- [[command-rule-policy]] — Declarative argv-prefix / network rules (allow/prompt/forbidden, strictest wins), layered, learned from approvals, model-proposed rules validated.
- [[dangerous-command-heuristics]] — Tiny parsed denylist (forced `rm`, Windows deletes/launchers) forcing a prompt, or a deny when nobody can be asked.
- [[tool-safety-annotations]] — Declarative unverified hints (readOnly/destructive/idempotent/openWorld) read by policy code or core approval logic.
- [[per-exec-interception]] — Patched shell routes every `execve` to the harness for Run / Escalate / Deny per sub-command.

### Reviewer (LLM judge)
- [[llm-approval-reviewer]] — Sandboxed LLM subagent answers approval requests with a categorical verdict over provenance-labelled evidence; fails closed; denial circuit breaker.
- [[delegated-authorization-provenance]] — Reviews of sub-agent actions count only the root user's genuine messages as authorization.

### Network
- [[egress-policy-proxy]] — Sandbox only reaches a harness loopback proxy enforcing domain allow/deny, limited methods, approvals, credential substitution.

### Trust, secrets & hardening
- [[project-trust-gate]] — Remembered per-directory trust decision gating repo-supplied executable/behavior-changing config (codex: also AGENTS.md and runtime defaults).
- [[secret-handling]] — Credentials at rest, child-env scrubbing, redaction in displays/reports, key-hiding proxies.
- [[harness-process-hardening]] — Credential-holding processes block ptrace/core dumps and loader-injection env; fail closed.
- [[supply-chain-pinning]] — Exact pins, lockfiles, install-script allowlists, release-age gates, cargo-deny source policy, pinned security binaries.
- [[remote-host-trust]] — Transport-layer auth (pinned host keys, capability tokens/JWT on upgrade, 0600 sockets), no forwarding, sanitized errors.
- [[permission-ruleset]] — Ordered allow/ask/deny rules over (permission, wildcard pattern), last match wins, layered defaults → agent → config → session; "ask" suspends until once/always/reject.
- [[shell-command-permission-parsing]] — Parse the shell command into an AST so each sub-command and path argument is permission-checked separately.
- [[workspace-boundary-check]] — Paths resolving outside the project/worktree trigger a separate `external_directory` permission.

## Absences
[[no-permission-prompts]] · [[no-sandbox]] · [[no-cwd-confinement]] · [[no-prompt-injection-defense]] · [[no-safe-command-allowlist]] · [[no-on-failure-approval-mode]] · [[no-cli-process-hardening]]

Tradeoffs: [[isolation-strategy]]

Failures: [[Safety Failures]]
