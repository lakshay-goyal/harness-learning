---
type: concept
stage: architecture
tier: must-have
aliases: [autoupdate, OPENCODE_DISABLE_AUTOUPDATE, "Installation.method", UpdateAvailable, "opencode upgrade", "pi update", PI_MANAGED_INSTALL_ROOT]
harnesses: [opencode, pi]
---
The CLI detects its own install method and upgrades itself in place.

## Why
- Agent CLIs ship daily; model and provider quirks are fixed in the harness, so stale installs break against live APIs.
- One binary can arrive through many channels (curl script, npm, pnpm, yarn, bun, brew, scoop, choco); upgrading through the wrong one leaves two copies on PATH.
- The update path runs package-manager code with user privileges; it is supply-chain surface ([[supply-chain-pinning]]).

## Design space
- **Detection**: inspect `process.execPath` and query each package manager's global list (opencode) vs managed-install root marker (pi `PI_MANAGED_INSTALL_ROOT`).
- **Policy**: silent auto-install of patch releases, notify for minor/major (opencode default) vs notify only (`autoupdate: "notify"`) vs off (`autoupdate: false`, `OPENCODE_DISABLE_AUTOUPDATE`) vs explicit user command only (pi `pi update`).
- **Install scripts**: run lifecycle scripts (opencode npm/pnpm/bun paths pass no `--ignore-scripts`) vs `--ignore-scripts` on every path (pi).
- **Script installs**: fetch vendor install script and pipe to a shell without integrity check (opencode curl path) vs content-addressed/locked installer (pi managed installer lock).
- **Age gate**: bypass npm `min-release-age` for the harness's own update (pi `--min-release-age=0`).

## Implementations
- [[opencode--self-update|opencode]] — `upgrade()` on start: `Installation.method()` over 8 methods, `getReleaseType` → patch auto-installs, others emit `UpdateAvailable`.
- [[pi--supply-chain-pinning|pi]] — `pi update` per install method with `--ignore-scripts`; managed-install self-update.

## Failures
- none recorded

## Related
[[supply-chain-pinning]] · [[install-telemetry]] · [[layered-settings]] · [[harness-package-distribution]] · [[opencode]] · [[pi]]
