---
type: concept
stage: architecture
tier: candidate
aliases: ["pi install", "pi packages", "\"pi\" manifest key", "pi-package keyword", "pi list/remove/update/config"]
harnesses: [pi]
---
Bundling plugins, skills, prompt templates and themes as versioned npm/git/local packages with a manifest, install scopes, filters and pinning.

## Why
- A plugin-first harness needs a distribution channel; otherwise users copy files around and can't update.
- Packages that ship their own copy of host libraries create duplicate module instances ([[duplicate-host-module-instances]]).
- Installs from untrusted repos run code: install scope + trust gating matter ([[project-trust-gate]], [[supply-chain-pinning]]).

## Design space
- **Source**: npm registry, git URL/ref, URL, local path (pi supports all four) vs curated marketplace only.
- **Manifest**: explicit `package.json` key listing resource globs (pi `"pi": {extensions, skills, prompts, themes}`) vs conventional directories (pi fallback).
- **Scope**: user-global vs project-local (pi `-l/--local` writes `.pi/settings.json`); project entry replaces or deltas the user entry.
- **Host deps**: bundled vs peer deps `"*"` with peer installs suppressed (pi).
- **Pinning**: floating vs pinned specs that `update` never moves (pi).
- **Filtering**: all resources vs per-package include/exclude globs that only narrow (pi).
- **Discovery**: registry keyword → gallery (pi `pi-package` → pi.dev/packages).
- **Install scripts**: run lifecycle scripts (pi for third-party, unverified intent) vs `--ignore-scripts` (pi for own installs).
- **Try-before-install**: one-run load (`pi -e npm:…`).

## Implementations
- [[pi--harness-package-distribution|pi]] — `pi install npm:|git:|https:|./path`, manifest + filters + pinning, peer-dep enforcement, gallery keyword.

## Failures
- [[duplicate-host-module-instances]]

## Related
[[runtime-plugin-loading]] · [[replaceable-builtin-extension]] · [[layered-settings]] · [[skill-progressive-disclosure]] · [[prompt-template-expansion]] · [[project-trust-gate]] · [[supply-chain-pinning]]
