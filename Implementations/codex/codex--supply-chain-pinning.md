---
type: implementation
harness: codex
concept: supply-chain-pinning
commit: 622e9e3696
files: [codex-rs/deny.toml:65, codex-rs/deny.toml:151, codex-rs/deny.toml:196, codex-rs/deny.toml:299, codex-rs/rust-toolchain.toml:1, pnpm-workspace.yaml:6, codex-rs/shell-escalation/README.md:20, codex-rs/linux-sandbox/README.md:10]
---
[[supply-chain-pinning]] in [[codex]].

## Mechanism
- **Rust (cargo-deny)** `codex-rs/deny.toml`: `[advisories]` with 10 justified `RUSTSEC` ignores (each with reason + removal condition, e.g. starlark's unmaintained `derivative`/`fxhash`/`paste`/`atomic-polyfill`, syntect's `yaml-rust`/`bincode`, rama's `hickory-proto`) (`:65-85`); `[licenses]` allowlist with `confidence-threshold = 0.8` (`:96-151`); `[bans] multiple-versions = "warn"` (`:194-196`); `[sources] unknown-registry = "deny"`, `unknown-git = "deny"` + `allow-org` (`:299-325`). Added `ec49b56874`.
- **Toolchain pin**: `channel = "1.95.0"` (`codex-rs/rust-toolchain.toml:1-3`); `Cargo.lock`, `MODULE.bazel.lock`, `flake.lock`, `pnpm-lock.yaml` committed (repo root).
- **npm/pnpm** (`pnpm-workspace.yaml:6-19`): `ignoredBuiltDependencies: [esbuild]`; `minimumReleaseAge: 10080` (7 days, no exclusions); `blockExoticSubdeps: true`; `strictDepBuilds: true`; `trustPolicy: no-downgrade` with `trustPolicyIgnoreAfter: 10080`; `allowBuilds` only for one MCP conformance CLI pinned by tarball commit hash.
- **Bundled security binaries**: vendored bubblewrap compiled in (`codex-rs/bwrap/`); system `bwrap` only from a PATH entry outside the cwd (`codex-rs/linux-sandbox/README.md:10-16`); patched zsh pinned to commit `77045ef899e5…`, released via `codex-zsh-vX.Y.Z` tags + checked-in DotSlash manifests (`codex-rs/shell-escalation/README.md:20-34`).
- **Distribution**: npm package `codex-cli/` wraps a native binary (`codex-cli/bin/codex.js`); cloud-delivered managed config bundle is HMAC-signed with an embedded key ([[layered-settings]], `codex-rs/cloud-config/src/cache.rs:27-33`).

## Constants
| name | value | path:line |
|---|---|---|
| pnpm minimum release age | 10080 min (7 d) | `pnpm-workspace.yaml:9` |
| cargo-deny license confidence | 0.8 | `codex-rs/deny.toml:151` |
| unknown registry / git | deny / deny | `codex-rs/deny.toml:302-305` |
| Rust toolchain | 1.95.0 | `codex-rs/rust-toolchain.toml:2` |

## Evolution
- 2025-11-24 `ec49b56874` "add cargo-deny configuration (#7119)".
- 2026-01-27 `dd24ac6b26` pnpm 10.28.2 "to address security issues".
- 2026-02-02 `f956cc2a02` vendor bubblewrap; 2026-03-26 `b6050b42ae` bwrap from trusted PATH entry.
- 2026-04-12 `a4d5112b37` "require reviewed dependency build scripts" (`strictDepBuilds`).
- 2026-04-24 `dee5f5ea38` "Harden package-manager install policy (#19163)" (`minimumReleaseAge`, `trustPolicy`).

## Versus pi
- [[pi--supply-chain-pinning]]: save-exact, 2-day `min-release-age`, `--ignore-scripts` on self-update, release-time install-script allowlist, content-addressed remote binaries. Codex: 7-day age gate + build-script allowlist in pnpm, cargo-deny source/advisory policy, pinned toolchain, vendored/pinned security-critical binaries (bwrap, zsh).
