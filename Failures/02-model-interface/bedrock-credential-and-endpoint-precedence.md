---
type: failure
concepts: [credential-resolution, http-transport-hardening]
harnesses: [pi]
---
**Symptom** — Bedrock connected with the wrong identity or to the wrong place:
- The configured AWS profile was ignored whenever ambient access keys were set.
- `us.*`/`eu.*` inference profiles broke because the catalog's us-east-1 endpoint overrode `AWS_REGION`.
- The region was not taken from profile config or inference-profile ARNs.
- Requests carried duplicate `Authorization` headers.
- GovCloud returned validation errors.
- Unauthenticated proxies could not be used.

**Root cause** — The AWS SDK's own precedence rules: explicit `credentials` override `config.profile`, and a pinned endpoint wins over region. In addition, pi's custom bearer middleware duplicated the SDK's auth header.

**Fix · [[pi]]**
- `df527fb98` 2026-02-06 — `AWS_BEDROCK_SKIP_AUTH=1` provides dummy credentials for unauthenticated proxies (#1320) (`packages/ai/src/api/bedrock-converse-stream.ts:183,208-213`).
- `d4b473e29` 2026-03-04 — respect the region from profile config when `AWS_PROFILE` is set (#1800).
- `454b9619c` 2026-04-18 — SDK-native `config.token` with `authSchemePreference:["httpBearerAuth"]`; omit `display` in GovCloud (#3359) (`:184-189,240-243,1244-1252`).
- `5a889ef58` 2026-04-19 — passed `model.baseUrl` as the endpoint (#3402). This **broke** regional inference profiles.
- `a0a16c776` 2026-04-21 — restored regional resolution. Standard endpoints are pinned only when there is no region and no ambient profile; non-standard (VPC/proxy) endpoints are always pinned (`:168-180,1217-1242`).
- `8cef3c8d7` 2026-06-08 — region extracted from inference-profile ARNs (`:193-205,1194-1201`).
- `5a8ea0bc8` 2026-06-23 — honor the scoped AWS profile in endpoint resolution.
- `b63403a50` 2026-07-28 — `options.profile || env.AWS_PROFILE` wins. Explicit credentials are set only when there is no profile (#6957/#7176) (`:157-165,215-218`).

**Lesson** — Know the SDK's credential and endpoint precedence. Pin endpoints only for custom URLs, and use SDK-native auth instead of header middleware.

Related: [[credential-resolution]] · [[http-transport-hardening]] · [[pi--credential-resolution|pi]]
