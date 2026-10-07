---
type: failure
concepts: [secret-handling, shell-execution]
harnesses: [codex]
---
**Symptom** — Harness-internal auth tokens (`NODE_REPL_AUTH_TOKEN`, exec-server Noise relay tokens, launch-context ids) were inherited by model-reachable child processes, where the model could read them with `env`.

**Root cause** — Children inherit the harness environment by default; each new internal credential was added to the parent env without a matching child-env rule, and the user-facing `*KEY*`/`*SECRET*`/`*TOKEN*` filter is off by default (`codex-rs/config/src/shell_environment_policy.rs:136`, `9fb9ed6cea`).

**Fix · [[codex]]** — 2026-08-08 `c4513cb982` "Prevent launch context from reaching child processes"; 2026-08-17 `89e297729e` "Prevent Noise auth tokens from reaching child processes"; 2026-08-18 `fe50b61689` "Prevent Node REPL auth tokens from reaching child processes" — `NON_INHERITABLE_ENV_VARS` removed case-insensitively after shell-environment-policy overrides (`codex-rs/protocol/src/shell_environment.rs:13-40`). Related hardening: 2026-08-18 `a04940cb12` reject symlinks in memory workspaces (consolidation worker links could reach outside files); 2026-08-21 `79b7606803` credentials out of app-server logs; credential broker gives children dummy values ([[egress-policy-proxy]]).

**Lesson** — Every new internal credential must be kept out of child environments; a denylist needs an entry per secret, so prefer an allowlist or proxy-held credentials.

Related: [[secret-handling]] · [[shell-execution]] · [[codex--secret-handling|codex]] · [[hardening-breaks-child-env]]
