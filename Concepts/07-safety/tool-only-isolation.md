---
type: concept
stage: permissions
tier: candidate
aliases: [Gondolin, ssh.ts, containerization.md, sandbox-runtime, Docker Sandboxes, tool-only-vs-whole-process-isolation, credential-proxy-substitution]
harnesses: [pi]
---
Keep the agent process (model client, credentials, plugins) on the host but route tool I/O (file ops, shell) into a VM / container / remote machine through pluggable tool backends — contrasted with whole-process isolation where the entire agent runs inside the sandbox.

## Why
- A harness without a built-in sandbox ([[no-sandbox]]) needs a documented containment story; pi's is "Safety comes from limiting the files, credentials, processes, and network services Pi can access".
- Whole-process isolation exposes credentials inside the box (env-passed API keys, mounted `~/.pi/agent`); tool-only isolation keeps file-based credentials on the host — but leaks env vars if the routing inherits host env.
- Neither protects against plugins/extension tools that don't delegate — tool-only isolation "does not constrain the host Pi process or extension tools".

## Design space
- **None** (YOLO, OS user is the boundary) — pi default.
- **Whole-process**: plain Docker (key via `-e`), Docker Sandboxes with proxy substituting a placeholder credential (real key never inside), policy sandbox (NVIDIA OpenShell). pi docs call this "usually the strongest practical option".
- **Tool-only, local VM**: Gondolin micro-VM, cwd mounted write-through, host env inherited (not a credential boundary).
- **Tool-only, remote**: SSH-delegated tools (`ssh.ts` example) or pi-env daemon (remote env only, local credentials).
- **Bash-only OS sandbox**: sandbox-exec / bubblewrap around the shell (pi `sandbox/` example); read/write/edit tools unwrapped.
- **Language-level sandbox for model-written code** (QuickJS code-mode): isolates script logic, not tool effects.
- Enabler: tools delegate to swappable `*Operations` backends ([[pluggable-tool-backends]]).

## Implementations
- [[pi--tool-only-isolation|pi]] — no built-in sandbox; `*Operations` seams; examples Gondolin/ssh/sandbox; `containerization.md` 4-pattern table; pi-env as unlisted 5th pattern.

## Failures
- [[repo-config-disables-sandbox-plugin]]

## Related
[[pluggable-tool-backends]] · [[remote-execution-env]] · [[remote-host-trust]] · [[process-tree-kill]] · [[code-mode]] · [[tool-call-gate]] · [[no-sandbox]] · [[no-cwd-confinement]] · [[credential-resolution]]
