---
type: implementation
harness: pi
concept: tool-only-isolation
commit: b30a6dd77
files: [packages/coding-agent/docs/containerization.md:7, packages/coding-agent/docs/security.md:11, packages/coding-agent/src/core/tools/bash.ts:75, packages/coding-agent/src/core/tools/read.ts:48, packages/coding-agent/examples/extensions/gondolin/index.ts:84, packages/coding-agent/examples/extensions/sandbox/index.ts:73, packages/coding-agent/examples/extensions/ssh.ts:1, packages/env/README.md:3, SECURITY.md:6]
---
[[tool-only-isolation]] in [[pi]].

## Mechanism
- **Stance**: no built-in sandbox — "Local code execution or sandboxing behavior (the Pi coding agent intentionally does not have a sandbox)" is out of scope (`SECURITY.md:50`); "responsibility of the user … to contain it within a container, virtual machine or other Sandbox solution" (`SECURITY.md:6-9`); boundary = OS user (`SECURITY.md:11-17`). "A partial in-process sandbox would be easy to misunderstand as a security boundary" (`packages/coding-agent/docs/security.md`, `5cb4f597f` diff). See [[no-sandbox]].
- **Three-tier table** (`packages/coding-agent/docs/security.md:11-17`): direct (OS user) / whole process in container-VM-sandbox ("usually the strongest practical option") / tool-only ("narrower form of isolation" — "Pi itself and other extensions remain outside the boundary"). "The working folder … does not prevent commands from accessing other paths" (`:19`; [[no-cwd-confinement]]).
- **Four documented patterns** (`packages/coding-agent/docs/containerization.md:7-14`):
  - Plain Docker: whole process; key passed as env; Dockerfile `node:24-bookworm-slim` + bash/git/ripgrep, `npm install -g --ignore-scripts`, named volume for `~/.pi/agent` (`:38-68`).
  - Docker Sandboxes (`sbx`): whole process; proxy substitutes a placeholder (`sk-ant-oat01-{rand}`) for the real credential per host; "Do not run `/login` inside the sandbox" (`:82-103`; `47236c844`).
  - NVIDIA OpenShell: whole process; policy-controlled credentials + inference routing (`:121-151`).
  - Gondolin extension: **tools only**, local QEMU micro-VM; host cwd mounted write-through at `/workspace`; "Commands inside the VM inherit the host process environment … Do not use this pattern as a credential boundary" (`:153-157`).
  - Cautions: rw mounts, mounting `~/.pi/agent` exposes credentials/sessions, env vars visible, network can exfiltrate, "Tool-only isolation does not constrain the host Pi process or extension tools" (`:18-28`).
- **Enabling seam** ([[pluggable-tool-backends]]): every built-in tool takes an `*Operations` backend — `BashOperations` (`packages/coding-agent/src/core/tools/bash.ts:75`), `ReadOperations` (`packages/coding-agent/src/core/tools/read.ts:48`), `WriteOperations` (`packages/coding-agent/src/core/tools/write.ts:27`), `EditOperations` (`packages/coding-agent/src/core/tools/edit.ts:83`), `LsOperations`, `FindOperations`, `GrepOperations`; bash also `spawnHook(ctx)` rewriting command/cwd/env (`bash.ts:189,211`) and `commandPrefix`; `user_bash` hook can return `{operations}` to route `!cmd`.
- **Gondolin example** (`packages/coding-agent/examples/extensions/gondolin/index.ts`): creates per-tool ops against the VM (`:84-122,183,324`), re-registers 7 built-ins (`:443-509`), routes `!` via `user_bash` (`:517-520`), rewrites prompt cwd line to guest path (`:522-530`). `sanitizeEnv` only drops non-string values (`:315-322`) → host env forwarded.
- **ssh.ts example**: `--ssh user@host[:path]` delegates read/write/edit/bash to the remote (`examples/extensions/ssh.ts:1-13`).
- **sandbox/ example** (bash-only OS sandbox, `@anthropic-ai/sandbox-runtime`: sandbox-exec / bubblewrap) (`sandbox/index.ts:1-10`; `4751ebddb`, #673): defaults `denyRead ~/.ssh ~/.aws ~/.gnupg`, `allowWrite . /tmp`, `denyWrite .env .env.* *.pem *.key`, network allowlist (`:68-76`); read/write/edit tools unwrapped. Config merges global `~/.pi/agent/extensions/sandbox.json` then project `<cwd>/.pi/sandbox.json` (project wins, `:79-101`), `enabled` overridable (`:108`), **no trust check** ([[repo-config-disables-sandbox-plugin]]).
- **pi-env (5th, unlisted pattern)**: Durable `ExecutionEnv` whose tools run on a remote machine over SSH while "the Durable worker, its storage and its credentials stay local" (`packages/env/README.md:3-4`); commands get remote daemon env + `shellEnv` only, never local env (`packages/env/docs/semantics.md:48-49`); daemon has no path allowlist/chroot — isolation = where it runs ([[remote-execution-env]], [[remote-host-trust]]).
- **Code-mode QuickJS**: "no Node APIs, file system, network, or timers" (`packages/coding-agent/docs/codemode.md:7`) — isolates script logic only; nested tool calls run with full effects ([[code-mode]]).
- **Durable env namespace**: `FileSystem.id` distinguishes `node:local` from each container/remote so per-file mutation queues don't collide (`packages/durable/src/env/index.ts:172-176`).

## Constants
| name | value | path:line |
|---|---|---|
| sandbox example `denyRead` | `~/.ssh, ~/.aws, ~/.gnupg` | packages/coding-agent/examples/extensions/sandbox/index.ts:73 |
| sandbox example `allowWrite` | `., /tmp` | packages/coding-agent/examples/extensions/sandbox/index.ts:74 |
| sandbox example `denyWrite` | `.env, .env.*, *.pem, *.key` | packages/coding-agent/examples/extensions/sandbox/index.ts:75 |
| Docker base image | `node:24-bookworm-slim` | packages/coding-agent/docs/containerization.md:38 |

## Evolution
- 2025-11-12 `b172beb92` "Fast iteration requires trust, not sandboxing … Use at your own risk".
- 2026-01-13 `4751ebddb` sandbox extension (#673).
- 2026-06-03 `86314bf38` containerization guide + Gondolin example (#5356); `a85196817` (2026-06-15) patterns reordered.
- 2026-06-08 `ce3a72444` security model doc; 2026-06-09 `5cb4f597f` "No built-in sandbox" rationale.
- 2026-09-04 `47236c844` Docker Sandboxes with proxy credential substitution (#9077).
- 2026-10-05 `ba03e03f2`/`b78e6a908`/`ed94330a2` pi-env remote env over hardened SSH.

## Evidence commits
`b172beb92`, `4751ebddb`, `86314bf38`, `a85196817`, `5cb4f597f`, `47236c844`, `ba03e03f2`, `ed94330a2`

## Quirks
- Gondolin routes only the 7 built-ins + `!`; extension tools still run on host (`containerization.md:16`).
- pi-env has no in-repo consumer at HEAD (only lockfile/tsconfig hits) — library for a Durable host (unverified product use; `~/.pi/mobile/tools/` path hints at a mobile host).

## Failures
- [[repo-config-disables-sandbox-plugin]]
