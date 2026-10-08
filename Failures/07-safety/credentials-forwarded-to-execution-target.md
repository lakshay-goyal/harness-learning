---
type: failure
concepts: [secret-handling, remote-execution-env, tool-only-isolation]
harnesses: [opencode]
---
**Symptom** — (Design property observed at `ecc4916b5a`, no incident reported.) Creating a control-plane workspace hands every stored provider credential to whatever the workspace adapter provisions — a worktree, a folder, or a plugin-defined VM/container/remote host.

**Root cause** — Remote targets run a full opencode server (model calls included), so the provisioning env carries `OPENCODE_AUTH_CONTENT = JSON.stringify(auth.all())` plus OTEL headers (`packages/opencode/src/control-plane/workspace.ts:529-536`), and the remote reads credentials from that variable (`packages/opencode/src/auth/index.ts:59-61`). Adapters are plugin-registered (`experimental_workspace.register`, `packages/plugin/src/index.ts:61`; example `packages/plugin/src/example-workspace.ts:4-5`) and receive that env via `WorkspaceAdapterRuntime.create(adapter, config, env)` (`packages/opencode/src/control-plane/workspace.ts:538`).

**Fix · [[opencode]]** — none (experimental feature, `OPENCODE_EXPERIMENTAL_WORKSPACES`).

**Lesson** — Moving the whole agent into an execution target moves its credentials too; scope or proxy credentials per target (tool-only isolation, placeholder credentials swapped by an egress proxy) instead of exporting the full store.

Related: [[secret-handling]] · [[remote-execution-env]] · [[tool-only-isolation]] · [[opencode--remote-execution-env|opencode]] · [[pi--tool-only-isolation|pi]]
