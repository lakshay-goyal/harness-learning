---
type: absence
harnesses: [pi]
---
# no-sandbox

**What's missing**
- No built-in OS, VM or container sandbox for tools or for the agent process. The trust boundary is the OS user.

**Evidence of decision**
- `SECURITY.md:50` (Out Of Scope): "Local code execution or sandboxing behavior (the Pi coding agent intentionally does not have a sandbox)".
- `SECURITY.md:6-9`: "It's the responsibility of the user to monitor its operations or to contain it within a container, virtual machine or other Sandbox solution."
- `SECURITY.md:11-17`: "Pi treats the local user account and files writable by that account as inside the same trust boundary as the Pi process itself."
- `packages/coding-agent/docs/security.md:99`: "…lack of a built-in sandbox … generally outside the security boundary".
- `packages/coding-agent/docs/security.md:7`: "Safety comes from limiting the files, credentials, processes, and network services Pi can access…"
- `b172beb92` (2025-11-12): "Fast iteration requires trust, not sandboxing"; mitigation "Run pi inside a container if you're uncomfortable with full access".

**Opt-in replacement**
- `docs/security.md:13-17` gives a three-tier table: direct / whole-process isolation ("usually the strongest practical option") / tool-only isolation.
- **Whole-process** (`packages/coding-agent/docs/containerization.md`):
  - Comparison table `:9-14`.
  - Plain Docker with `npm install -g --ignore-scripts` (`:38-48`).
  - Docker Sandboxes with proxy-substituted placeholder credentials (`:82-119`, added `47236c844` 2026-09-04).
  - OpenShell policy sandbox (`:121`).
- **Tool-only** ([[tool-only-isolation]], [[pluggable-tool-backends]]):
  - `examples/extensions/gondolin/` routes built-in tools and `!` commands into a local micro-VM via pluggable `*Operations` (`gondolin/index.ts:1-7,84-122`).
    - Caveat: "Commands inside the VM inherit the host process environment … Do not use this pattern as a credential boundary" (`containerization.md:157`).
  - `examples/extensions/ssh.ts` delegates all tools to a remote host.
  - The experimental `packages/env` daemon does the same over SSH ([[remote-execution-env]], [[remote-host-trust]]).
- **Bash-only OS sandbox**: `examples/extensions/sandbox/` uses `@anthropic-ai/sandbox-runtime` (sandbox-exec/bubblewrap; `sandbox/index.ts:47`).
  - Defaults: `denyRead: ["~/.ssh","~/.aws","~/.gnupg"]`, `allowWrite: [".","/tmp"]` (`:73-74`).
  - Added `4751ebddb` (2026-01-13, #673).
  - read/write/edit are **not** wrapped ("OS-level sandboxing for bash commands", `:2`).
- **Script sandbox only**: codemode's QuickJS has "no Node APIs, file system, network, or timers; scripts reach the outside world only through tools and `models`" (`packages/coding-agent/docs/codemode.md:7`). It isolates script logic, not tool effects → [[code-mode]].
- Durable: `packages/durable/test/examples/29-sandbox-per-conversation.ts` (per-conversation env; details unverified).

**History**
- Stance unchanged since `b172beb92`. Containment documentation grew over time: containerization doc, Docker Sandboxes `47236c844`, the gondolin example, and the remote `packages/env` daemon (`ba03e03f2`, 2026-10-05).

**Implication**
- Isolation is pushed to the OS/VM layer. The architectural enabler is that every built-in tool does its I/O through swappable `ReadOperations` / `BashOperations` / `ExecutionEnv`.
- Open hole (unverified): `examples/extensions/sandbox/` reads `<cwd>/.pi/sandbox.json` unconditionally (`sandbox/index.ts:80,94-102`), and `enabled` is overridable (`:108`). `sandbox.json` is not in the project-trust list (`packages/coding-agent/src/core/trust-manager.ts:30-39`), so a hostile repo could disable or widen the sandbox for a user who installed the example globally.

**codex** — *not absent; design pillar since the TS era*: `codex debug seatbelt/landlock` `e79549f039` 2025-04-28; crates `codex-rs/linux-sandbox`, `codex-rs/windows-sandbox-rs`, `codex-rs/sandboxing`, `codex-rs/bwrap`, `codex-rs/mxc-sandbox`; Windows sandbox alpha `87cce88f48` 2025-10-30 → [[os-level-sandbox]]. On by default on macOS/Linux; on Windows only when `windows.sandbox` is enabled — otherwise the default profile drops to a read-only *policy shape* and dangerous-command heuristics treat everything as needing approval (`codex-rs/core/src/config/permissions.rs:55-61`, `codex-rs/core/src/exec_policy.rs:787-806`). Escape hatch `--dangerously-bypass-approvals-and-sandbox` → `SandboxMode::DangerFullAccess` (`codex-rs/exec/src/lib.rs:343-347`).

Related: [[tool-only-isolation]] · [[pluggable-tool-backends]] · [[remote-execution-env]] · [[supply-chain-pinning]] · [[process-tree-kill]] · [[no-permission-prompts]] · [[no-cwd-confinement]] · [[no-prompt-injection-defense]] · [[Absences]] · [[os-level-sandbox]] · [[sandbox-escalation-retry]] · [[harness-process-hardening]] · [[isolation-strategy]]
