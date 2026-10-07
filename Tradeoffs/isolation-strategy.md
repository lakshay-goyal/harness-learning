---
type: tradeoff
concepts: [os-level-sandbox, tool-only-isolation, approval-policy-modes, llm-approval-reviewer, tool-call-gate, egress-policy-proxy, command-rule-policy, project-trust-gate]
---
**Axis** — where the trust boundary sits and who decides when to cross it: the OS user + an optional plugin gate (trust the user, ship no policy) vs. a default kernel sandbox per command + built-in approval policy + rules + an LLM reviewer (ship the policy).

| dimension | pi | codex |
|---|---|---|
| Default trust boundary | OS user; "No permission popups" ([[no-sandbox]], [[no-permission-prompts]]; `b172beb92`, `3424550d2` "Security theater") | Kernel sandbox per command, network off, workspace-write after trust (`codex-rs/sandboxing/src/manager.rs:50-64`; `codex-rs/protocol/src/protocol.rs:1093-1147`; `codex-rs/core/src/config/permissions.rs:50-61`) |
| Sandbox backends | none in core; examples: Gondolin VM, SSH, bash-only sandbox-exec/bwrap, containers ([[pi--tool-only-isolation]]) | Seatbelt, bubblewrap + seccomp (Landlock legacy), Windows restricted token / elevated users + WFP / MXC ([[codex--os-level-sandbox]]) |
| Whole-process option | documented as "usually the strongest practical option" (Docker, Docker Sandboxes, OpenShell) | `run_in_container.sh` + firewall, `ExternalSandbox`, `--dangerously-bypass-approvals-and-sandbox` for already-isolated hosts ([[codex--tool-only-isolation]]) |
| Who decides | user's own extensions in `beforeToolCall` / `tool_call`; fail-closed on throw ([[pi--tool-call-gate]]) | approval policy matrix (untrusted / on-request / never / granular) → hooks → Guardian LLM → user ([[codex--approval-policy-modes]], [[codex--llm-approval-reviewer]]) |
| Escalation | n/a (nothing to escalate from) | model-initiated `require_escalated` + `justification` + `prefix_rule`; scoped `request_permissions`; reactive retry only in untrusted/granular ([[codex--sandbox-escalation-retry]]) |
| Static policy | none in core; regex lists in example extensions | Starlark `rules/*.rules`, admin requirements overlay, tiny dangerous-command denylist; curated safe-list deleted `942af8447b` ([[codex--command-rule-policy]]) |
| Network | unrestricted | off by default; loopback proxy with domain allow/deny, limited mode, credential broker ([[codex--egress-policy-proxy]]) |
| Write carve-outs | none (protected-paths example is tool-name based) | `.git`, `.agents`, `.codex`, `.aws` read-only in writable roots ([[codex--protected-workspace-metadata]]) |
| Prompt injection stance | declared out of scope ([[no-prompt-injection-defense]]); AGENTS.md ungated by trust | provenance-labelled reviewer evidence, root-user-only authorization for sub-agents, AGENTS.md gated by trust ([[delegated-authorization-provenance]], [[codex--project-trust-gate]]) |
| Cost | zero friction, zero latency; safety = user's environment choice | approval UX, reviewer latency (90 s cap, 6 s async scorer), per-OS hardening churn (dozens of sandbox bypass fixes, 581 guardian commits), Windows default still unsandboxed |

**When each wins**
- **pi-style (trust the user, ship seams)** — single developer on their own machine or already inside a container/VM; harness authors who don't want to own a security boundary they can't fully enforce ("easily circumvented"); maximal extensibility. Fails when the agent runs on a real workstation with credentials and untrusted repos.
- **codex-style (ship the boundary)** — default-safe for a mass-market CLI/IDE on real workstations, unattended runs (`never` + sandbox), enterprises needing admin floors (requirements, rules, managed network). Costs: per-OS kernel engineering with a long tail of bypasses ([[sandbox-path-binding-races]], [[sandbox-side-channel-syscalls]]), prompts that must teach the model the sandbox ([[permission-state-prompt]]), and an LLM reviewer that needs constant calibration ([[approval-reviewer-overcautious]]).
- Shared ground: both keep the harness process itself outside the isolation and treat project config as trust-gated ([[project-trust-gate]]); both avoid a curated safe-command list today — pi by philosophy, codex after maintaining one and deleting it ([[no-safe-command-allowlist]]).
