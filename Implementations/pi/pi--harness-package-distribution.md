---
type: implementation
harness: pi
concept: harness-package-distribution
commit: b30a6dd77
files: [packages/coding-agent/docs/packages.md:9-127, packages/coding-agent/src/core/pi-manifest.ts:4-35, packages/coding-agent/src/core/package-manager.ts:50-52, packages/coding-agent/src/core/package-manager.ts:1739-1757, packages/coding-agent/src/core/package-manager.ts:1823-1865, packages/coding-agent/src/core/resource-loader.ts:53-88, packages/coding-agent/docs/cli.md:250-268]
---
[[harness-package-distribution]] in [[pi]].

## Mechanism
- Packages bundle extensions, skills, prompt templates, themes. Install: `pi install npm:@x/y@1.0.0 | git:github.com/x/y@v1 | https://… | ./local`; `pi list/remove/update/config`; `-l/--local` writes to project `.pi/settings.json` (`packages/coding-agent/docs/packages.md:9-37`; `docs/cli.md:250-268`). `pi -e npm:…` loads for one run only.
- Manifest: `package.json` `"pi": {extensions, skills, prompts, themes}` — string arrays with globs (`core/pi-manifest.ts:4-35`, validated since `74caa2649`); without it, conventional dirs. `pi-package` keyword → gallery pi.dev/packages; `pi.image` / `pi.video` previews (`docs/packages.md:74`).
- Host-provided deps (`pi-ai`, `pi-agent-core`, `pi-coding-agent`, `pi-tui`, `typebox`) must be `peerDependencies: "*"`; managed installs disable peer installs (npm `--legacy-peer-deps`, bun/pnpm `--omit=peer`); pi warns when host packages appear in `dependencies` (`docs/packages.md:80-90`; `core/package-manager.ts:1823-1852`; `core/resource-loader.ts:53-88`; `8d897edaa`). Runtime side: virtual modules alias host packages ([[pi--runtime-plugin-loading]]).
- Per-package settings filter object: omit = all, `[]` = none, `!glob` exclude, `+path` force-include, `-path` force-exclude; filters only narrow (`docs/packages.md:96-119`).
- Identity: npm by name, git by URL sans ref, local by absolute path; project entry replaces user entry unless `autoload:false` (delta) (`docs/packages.md:125-127`; `package-manager.ts:1739-1757`).
- Pinning: versioned npm specs and git tags/commits pinned; `update` reconciles but never moves a configured ref (`docs/packages.md:38`).
- Project-scoped packages gated by project trust (`docs/configuration.md:3,47`) → [[project-trust-gate]].
- Third-party installs run lifecycle scripts (no `--ignore-scripts`, `package-manager.ts:1844-1865`) while pi's own release installs forbid them — intent unverified → [[supply-chain-pinning]].
- Experimental Chord plugins: package-based plugins built into content-addressed facet bundles, never installing deps or running scripts (`packages/chord/README.md:183-226`; `429f4e756`).
- **Standalone Bun binary quirks** (`packages/coding-agent/src/bun/`): compiled Bun binaries see an **empty `process.env`** inside sandboxes such as nono (oven-sh/bun#27802); `restoreSandboxEnv()` refills it from `/proc/self/environ` (Linux only) before any module reads env, mirrored by `getBunSandboxEnvValue()` in pi-ai for direct consumers (`bun/restore-sandbox-env.ts:1-36`, `bun/sandbox-env-setup.ts:1-4`). `bun/runtime-setup.ts:1-12` registers Bun OAuth flows, the Bedrock provider module, and the QuickJS WASM embedded in the executable (for [[pi--code-mode|code-mode]]); it also silences `process.emitWarning`.
- Install-location overrides: `PI_PACKAGE_DIR` replaces the detected package dir ("useful for Nix/Guix where store paths tokenize poorly", `src/config.ts:393-397`); `npmCommand` setting = argv used for package lookup/install, e.g. `["mise","exec","node@20","--","npm"]` (`core/settings-manager.ts:153,1144-1146`).

## Constants
| name | value | path:line |
|---|---|---|
| package-manager network timeout | 10 s | `core/package-manager.ts:50` |
| package-manager concurrency | 4 / 4 | `core/package-manager.ts:51-52` |

## Evolution
- 2026-01-20 `b846a4bfc` ResourceLoader + package management + `/reload` (#645).
- 2026-07-30 `74caa2649` validate package manifests.
- 2026-08-30 `429f4e756` package-based plugins for experimental Chord facets.
- 2026-09-23 `8d897edaa` suppress peer installs, warn on host packages in `dependencies` (#9863).

## Evidence commits
`b846a4bfc` `74caa2649` `429f4e756` `8d897edaa`

## Quirks
- Lifecycle scripts asymmetry (own installs `--ignore-scripts`, third-party not) — open question.
- `-l` installs write the package entry into an untrusted-until-trusted project layer.

## Failures
[[duplicate-host-module-instances]]
