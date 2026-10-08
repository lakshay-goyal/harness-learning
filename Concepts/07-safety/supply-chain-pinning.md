---
type: concept
stage: permissions
tier: must-have
aliases: [install-lock, allowedInstallScriptPackages, --ignore-scripts, min-release-age, save-exact, PI_ALLOW_LOCKFILE_CHANGE, shrinkwrap, pi-env-<sha256>, install-script-allowlist, content-addressed-agent-deploy, ignoreScripts, cargo-deny, deny.toml, minimumReleaseAge, strictDepBuilds, trustPolicy, rust-toolchain.toml]
harnesses: [pi, opencode, codex]
---
Treat the harness's own install/update path as attack surface: exact-pinned dependencies, lockfile-pinned installer, lifecycle scripts disabled at install time and rejected at release time unless explicitly allowlisted with a justification, and remote helper binaries addressed and verified by content hash.

## Why
- An agent CLI runs with full user privileges and self-updates; a compromised transitive dependency's `postinstall` runs before any agent safety logic.
- Fresh malicious releases are typically caught within days; age gates and exact pins narrow the window.
- Remote helper binaries named by version can collide or be stale and give no integrity guarantee ([[remote-binary-trusted-by-version-name]]).

## Design space
- **Floating semver deps** vs `save-exact` + committed lockfile (✔ pi; ✔ codex: Cargo/pnpm/Bazel/flake lockfiles, pinned Rust toolchain 1.95.0).
- **Shrinkwrap in the published package** (pi `17cc86a47`, removed `581e7ba78`) vs separate installer lockfile from the vendor site (pi now).
- **Install-time scripts**: allow (npm default) vs `--ignore-scripts` (pi self-update + managed installer) vs pnpm-style approve-list (✔ codex `strictDepBuilds` + `allowBuilds` for one hash-pinned package, `ignoredBuiltDependencies: [esbuild]`).
- **Release-time check**: fail the lock generation on any `hasInstallScript` not in a justified allowlist; also fail on stale allowlist entries (pi).
- **Age gate**: `min-release-age` for developers (pi `.npmrc` = 2 days) but bypassed (`=0`) for self-update so new pi releases install immediately; codex `minimumReleaseAge: 10080` (7 days) + `trustPolicy: no-downgrade`.
- **Source/advisory policy for compiled deps**: cargo-deny denies unknown registries and git sources, license allowlist, advisories with justified ignores (✔ codex).
- **Security-critical helper binaries**: vendored and compiled in (codex bubblewrap) or pinned to an upstream commit with tagged releases (codex patched zsh); system copy only from a PATH entry outside the workspace.
- **Third-party plugin packages**: same strictness vs trusted-code stance (pi: plugin installs do *not* pass `--ignore-scripts`).
- **Remote binaries**: version-named vs SHA-256 content-addressed + verify-before-every-start + atomic upload (pi-env).
- **Asymmetry reversed**: third-party plugin installs with scripts disabled (opencode arborist `ignoreScripts: true`) while the harness's own npm self-update runs scripts and the curl path pipes an unchecksummed script to a shell (opencode).
- **Remote content**: skill indexes downloaded by version marker without hash verification (opencode `skills.urls`).

## Implementations
- [[pi--supply-chain-pinning|pi]] — `.npmrc` save-exact/min-release-age=2; pre-commit lockfile guard; `generate-coding-agent-install-lock.mjs` allowlist (3 packages); `--ignore-scripts` on every self-update path; pi-env `pi-env-<sha256[0:32]>`.
- [[codex--supply-chain-pinning|codex]] — `codex-rs/deny.toml` (unknown registry/git denied, 10 justified RUSTSEC ignores, license confidence 0.8); pnpm 7-day age gate + reviewed build scripts; toolchain pin; vendored bwrap, pinned patched zsh.
- [[opencode--supply-chain-pinning|opencode]] — plugins via arborist `ignoreScripts: true` under a lock; self-update via `npm/pnpm/bun install -g` (scripts on) or `curl | bash`; remote skills unverified.

## Failures
- [[remote-binary-trusted-by-version-name]]
- [[search-binary-bootstrap-failures]] (03-tools) — The auto-download of ripgrep/fd (needed by grep/find) failed in several environment-specific ways: startup…

## Related
[[harness-package-distribution]] · [[project-trust-gate]] · [[remote-host-trust]] · [[remote-execution-env]] · [[install-telemetry]] · [[os-level-sandbox]] · [[per-exec-interception]] · [[self-update]]
