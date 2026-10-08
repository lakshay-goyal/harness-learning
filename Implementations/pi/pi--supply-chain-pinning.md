---
type: implementation
harness: pi
concept: supply-chain-pinning
commit: b30a6dd77
files: [.npmrc:1, scripts/generate-coding-agent-install-lock.mjs:17, scripts/generate-coding-agent-install-lock.mjs:304, scripts/check-lockfile-commit.mjs:119, .husky/pre-commit:6, packages/coding-agent/src/config.ts:139, packages/coding-agent/src/package-manager-cli.ts:89, packages/coding-agent/src/core/package-manager.ts:1844, packages/env/src/ssh.ts:370]
---
[[supply-chain-pinning]] in [[pi]].

## Mechanism
- **Repo-level pins**: `.npmrc` `save-exact=true`, `min-release-age=2` (days) (`.npmrc:1-2`, from `17cc86a47`).
- **Lockfile commit guard**: `.husky/pre-commit` runs `scripts/check-lockfile-commit.mjs` first (`.husky/pre-commit:6-9`); any `package-lock.json` change fails the commit with a checklist ("confirm npm age gates were active", "review any new lifecycle scripts", "regenerate/check coding-agent install lock") unless `PI_ALLOW_LOCKFILE_CHANGE=1|true|yes` (`check-lockfile-commit.mjs:5-6,100-120`).
- **CI**: daily `npm audit` workflow installs with `npm ci --ignore-scripts`, runs `scripts/npm-audit.mjs`, then `npm audit signatures --omit=dev`; actions pinned by commit SHA (`.github/workflows/npm-audit.yml`).
- **Installer lock** `scripts/generate-coding-agent-install-lock.mjs` (`622eca760`) → `packages/coding-agent/install-lock/{package.json,package-lock.json}` consumed by the pi.dev managed installer. Generation/`--check` **fails** when (`:300-345`):
  - an entry is a `link` or has a local `resolved` (`file:`, `link:`, `workspace:`, relative/absolute path) (`:304-309`);
  - dev / devOptional / extraneous metadata present (`:310-312`);
  - an internal `@earendil-works/pi-*` or `@earendil-works/chord` version ≠ installer version (`:313-319`; prefixes `:14-15`);
  - any dependency `hasInstallScript` not in `allowedInstallScriptPackages` — "Review it and add it to allowedInstallScriptPackages if intentional." (`:320-333`);
  - an allowlisted package is no longer present ("remove it from the allowlist") (`:336-340`);
  - an internal dependency is missing (`:342-345`).
- **Allowlist (exact name@version + justification)** (`:17-21`):
  - `@google/genai@2.21.0` — "preinstall is a no-op in the published package"
  - `esbuild@0.28.2` — "postinstall selects and verifies the platform-specific esbuild binary"
  - `protobufjs@7.6.6` — "postinstall only warns about protobufjs version scheme mismatches"
- **Install-time scripts disabled for pi itself**: every self-update path (`npm`/`pnpm`/`yarn global add`/`bun`) passes `--ignore-scripts` (`packages/coding-agent/src/config.ts:142,154,164,183`; `a3ebcd232`); managed installer runs `npm ci --ignore-scripts --min-release-age=0 --omit=dev --include=optional` (`packages/coding-agent/src/package-manager-cli.ts:89-99`); docs recommend `npm install -g --ignore-scripts` (`docs/quickstart.md:18`, `docs/containerization.md:44`).
- **Age-gate bypass on self-update**: `--min-release-age=0` — "pi.dev advertises releases immediately, so a configured npm age gate would block the update. npm has no per-package age gate, so this also lets new transitive dependency releases through. Managed installs avoid this." (`config.ts:175-185`).
- **Third-party pi packages (plugins)**: `npm install <specs> --prefix <root> --legacy-peer-deps` / `bun install --omit=peer` / `pnpm install … --config.strict-dep-builds=false` — **no `--ignore-scripts`** (`packages/coding-agent/src/core/package-manager.ts:1844-1865`) → lifecycle scripts of user-chosen packages run, consistent with "extensions are trusted code" (`docs/packages.md:21`). Project-declared packages install only after [[project-trust-gate]].
- **Remote helper binaries (pi-env)**: content-addressed `~/.pi/mobile/tools/pi-env-<sha256[0:32]>` (`packages/env/src/ssh.ts:370-375`); remote hash via `sha256sum`|`shasum -a 256`|`openssl dgst`, else refuse (`nohash`) (`:377-407`); POSIX upload to `mktemp` in 0700 dir, verify hash, `chmod 700`, `mv -f`, prune other `pi-env-*` (`:434-451`); Windows base64 + `PI-ENV-END` marker, `Get-FileHash` verify, ≤20 move retries × 250 ms (`:408-432`); verified before every start (`:480-502`). Rust daemon deps pinned with `=` and toolchain 1.96.0 (`packages/env/daemon/Cargo.toml:8-24`, `rust-toolchain.toml:2`).
- **Chord bundler** never installs deps or runs lifecycle scripts; facet bundles carry `sha256-<base64>` integrity checked before evaluation (`packages/chord/README.md:183-226`; `src/node/bundle.ts:152`, `bundle-loader.ts:25,233-240`).
- **Managed-install self-update** (installer-script installs, distinct from `npm -g`): active only if `PI_MANAGED_INSTALL_ROOT` is set **and** the running package lives under `<root>/releases` (inherited env must not misclassify a source checkout) **and** `<root>/managed-install.json` = `{kind:"pi-managed-install", schemaVersion:1, layout:"releases-v1"}` (`packages/coding-agent/src/package-manager-cli.ts:54-79`). Update = `proper-lockfile` on `<root>/update` (concurrent update → "Another managed Pi update is already running.") → fetch `package.json` + `package-lock.json` from `PI_INSTALLER_API_BASE` (default `https://pi.dev/api/installer/releases/<ver>`) → `npm ci --ignore-scripts --min-release-age=0 --omit=dev --include=optional` in a staging dir → **smoke test** `<bin> --version` must equal the target version → rename to `releases/<ver>` → atomic tmp+rename of `current-version` → prune all releases except the new one and the one currently running ("allows rolling back by editing current-version") (`package-manager-cli.ts:89-155,191-241`). Version strings validated by semver regex before use as paths (`:52,192-194`).
- Windows: native `.node` modules loaded by the running process cannot be deleted during self-update, so files the process has loaded (found via `process.report.getReport().sharedObjects` under the package dir) are moved into `node_modules/.pi-native-quarantine/<ts>-<pid>-<uuid>/` and that dir is removed on a later run (`src/utils/windows-self-update.ts:6-84`).

## Constants
| name | value | path:line |
|---|---|---|
| `min-release-age` (dev) | 2 days | .npmrc:2 |
| `save-exact` | true | .npmrc:1 |
| self-update `--min-release-age` | 0 | packages/coding-agent/src/config.ts:183-184 |
| install-script allowlist | 3 entries | scripts/generate-coding-agent-install-lock.mjs:17-21 |
| lockfile override env | `PI_ALLOW_LOCKFILE_CHANGE=1` | scripts/check-lockfile-commit.mjs:119 |
| pi-env binary name | `pi-env-<sha256[0:32]>` | packages/env/src/ssh.ts:371 |

## Evolution
- 2026-05-20 `17cc86a47` "harden dependency workflows": `.npmrc` min-release-age, pre-commit lockfile guard, npm-audit workflow, shrinkwrap generator. Same day `a3ebcd232` "disable scripts during self-update".
- 2026-06-26 `622eca760` installer lock generation (Armin Ronacher).
- 2026-07-23 `ec1a87e85` protobufjs bump (#7005); 2026-09-08 `4a6ed0194` runtime deps update (#9341) — allowlist versions move with deps (stale-entry check forces it).
- 2026-08-24 `4af9d21d3` managed installations updated in place.
- 2026-10-02 `581e7ba78` npm shrinkwrap removed: "npm installs no longer pin transitive dependencies. Pinned installs come from the pi.dev installer lockfile" (#5653).
- 2026-10-05 `b78e6a908` → `ed94330a2`: pi-env daemon renamed from `pi-env-<VERSION>` to `pi-env-<sha256>`, old binaries pruned ([[remote-binary-trusted-by-version-name]]).

## Evidence commits
`17cc86a47`, `a3ebcd232`, `622eca760`, `ec1a87e85`, `4a6ed0194`, `4af9d21d3`, `581e7ba78`, `b78e6a908`, `ed94330a2`

## Quirks
- Asymmetry: pi's own deps forbid unreviewed lifecycle scripts; plugin packages run theirs (no doc statement of intent found — open question).
- Global `npm install -g` users lost transitive pinning in `581e7ba78`; only the managed installer is pinned.
- `--min-release-age=0` on self-update disables the age gate for *transitive* deps too (acknowledged in comment).

## Failures
- [[remote-binary-trusted-by-version-name]]
