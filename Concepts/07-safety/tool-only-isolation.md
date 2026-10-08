---
type: concept
stage: permissions
tier: must-have
aliases: [Gondolin, ssh.ts, containerization.md, sandbox-runtime, Docker Sandboxes, tool-only-vs-whole-process-isolation, credential-proxy-substitution, ExternalSandbox, run_in_container.sh, init_firewall.sh]
harnesses: [pi, codex]
---
Keep the agent process (model client, credentials, plugins) on the host but route tool I/O (file ops, shell) into a VM / container / remote machine through pluggable tool backends — contrasted with whole-process isolation where the entire agent runs inside the sandbox.

## Why
- A harness without a built-in sandbox ([[no-sandbox]]) needs a documented containment story; pi's is "Safety comes from limiting the files, credentials, processes, and network services Pi can access".
- Whole-process isolation exposes credentials inside the box (env-passed API keys, mounted `~/.pi/agent`); tool-only isolation keeps file-based credentials on the host — but leaks env vars if the routing inherits host env.
- Neither protects against plugins/extension tools that don't delegate — tool-only isolation "does not constrain the host Pi process or extension tools".

## Design space
- **None** (YOLO, OS user is the boundary) — pi default.
- **Tool-only, local, kernel sandbox per command, on by default** — ✔ codex ([[os-level-sandbox]]): harness on host, every exec path wrapped by Seatbelt / bwrap+seccomp / Windows token or MXC.
- **Whole-process**: plain Docker (key via `-e`), Docker Sandboxes with proxy substituting a placeholder credential (real key never inside), policy sandbox (NVIDIA OpenShell). pi docs call this "usually the strongest practical option". codex: `run_in_container.sh` (Docker + iptables/ipset domain allowlist) *plus* the inner sandbox; `ExternalSandbox` policy / `--dangerously-bypass-approvals-and-sandbox` for already-isolated hosts.
- **Tool-only, local VM**: Gondolin micro-VM, cwd mounted write-through, host env inherited (not a credential boundary).
- **Tool-only, remote**: SSH-delegated tools (`ssh.ts` example) or pi-env daemon (remote env only, local credentials); codex `exec-server` remote environments keep their own sandbox backend.
- **Credential substitution without a VM**: local egress proxy holds real credentials, children get dummy values (✔ codex credential broker, [[egress-policy-proxy]]).
- **Bash-only OS sandbox**: sandbox-exec / bubblewrap around the shell (pi `sandbox/` example); read/write/edit tools unwrapped. codex has no separate file tools to leave unwrapped — reads go through the sandboxed shell, edits through sandboxed `apply_patch`.
- **Language-level sandbox for model-written code** (QuickJS code-mode): isolates script logic, not tool effects.
- Enabler: tools delegate to swappable `*Operations` backends ([[pluggable-tool-backends]]).
- **None, stated as policy** (opencode: "OpenCode does **not** sandbox the agent … run OpenCode inside a Docker container or VM", `SECURITY.md:15-19`).
- **Whole-agent remote targets via pluggable adapters** with the credential store forwarded (opencode control-plane workspaces, [[remote-execution-env]]).

## Implementations
- [[pi--tool-only-isolation|pi]] — no built-in sandbox; `*Operations` seams; examples Gondolin/ssh/sandbox; `containerization.md` 4-pattern table; pi-env as unlisted 5th pattern.
- [[codex--tool-only-isolation|codex]] — per-command OS sandbox by default (harness unsandboxed); container script + `ExternalSandbox` for whole-process; exec-server remote envs; local credential broker.

## Failures
- [[read-path-traversal]]
- [[repo-config-disables-sandbox-plugin]]
- codex: [[sandbox-failure-misread-as-transient]] · [[symlinked-roots-escape-sandbox-policy]]
- [[read-path-traversal]] (03-tools) — read tool could read files anywhere on disk via ../ or absolute paths. Reported as a "path traversal…
- [[credentials-forwarded-to-execution-target]]

## Related
[[pluggable-tool-backends]] · [[remote-execution-env]] · [[remote-host-trust]] · [[process-tree-kill]] · [[code-mode]] · [[tool-call-gate]] · [[no-sandbox]] · [[no-cwd-confinement]] · [[credential-resolution]] · [[os-level-sandbox]] · [[egress-policy-proxy]] · [[isolation-strategy]] · [[permission-ruleset]]

## Tradeoffs
- [[permission-prompts-vs-none]]
- [[cwd-confinement-vs-none]]
