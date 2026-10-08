---
type: implementation
harness: opencode
concept: self-update
commit: ecc4916b5a
files: [packages/opencode/src/cli/upgrade.ts:8-50, packages/opencode/src/installation/index.ts:18, packages/opencode/src/installation/index.ts:126-182, packages/opencode/src/installation/index.ts:145-164, packages/opencode/src/installation/index.ts:258, packages/opencode/src/installation/index.ts:265-290]
---
[[self-update]] in [[opencode]].

## Mechanism
### Legacy runtime
- **Methods**: `curl | npm | yarn | pnpm | bun | brew | scoop | choco | unknown` (`packages/opencode/src/installation/index.ts:18`).
- **Detection** (`packages/opencode/src/installation/index.ts:126-182`): `process.execPath` under `.opencode/bin` or `.local/bin` → curl; else query each manager's global list (`npm list -g --depth=0`, `yarn global list`, `pnpm list -g`, …) and brew formulas (`anomalyco/tap/opencode` vs core `opencode`).
- **Latest version**: per method (e.g. GitHub releases API `https://api.github.com/repos/anomalyco/opencode/releases/latest`, `packages/opencode/src/installation/index.ts:258`).
- **Policy** (`packages/opencode/src/cli/upgrade.ts:8-50`), run at startup:
  - `autoupdate: false` or `OPENCODE_DISABLE_AUTOUPDATE` → nothing.
  - `OPENCODE_ALWAYS_NOTIFY_UPDATE` → emit `UpdateAvailable` only.
  - same version → nothing.
  - `autoupdate: "notify"` or release type ≠ patch → emit `UpdateAvailable` (UI shows it).
  - patch release and known method → **install silently**, then emit an updated event.
- **Upgrade commands** (`packages/opencode/src/installation/index.ts:265-290`): `npm|pnpm|bun install -g opencode-ai@<v>` (scripts enabled); brew with `HOMEBREW_NO_AUTO_UPDATE=1`, tapping `anomalyco/tap` when needed; curl → download `https://opencode.ai/install` and pipe to bash/sh with `VERSION=<v>` (`packages/opencode/src/installation/index.ts:145-164`). choco fails with a hint when not elevated.
- Manual `opencode upgrade` command exists (`packages/opencode/src/cli/cmd/upgrade.ts`).

## Constants
| name | value | path:line |
|---|---|---|
| silent auto-install scope | patch releases only | `packages/opencode/src/cli/upgrade.ts:28-30` |

## Evolution
- Not mined.

## Quirks / drift
- No `--ignore-scripts` and no checksum on the curl path ([[supply-chain-pinning]]).
- Detection by `execPath` substring can misclassify custom install dirs (inferred).

pi contrast: explicit `pi update` only, `--ignore-scripts` on every path, managed-install lock ([[pi--supply-chain-pinning|pi]]).
