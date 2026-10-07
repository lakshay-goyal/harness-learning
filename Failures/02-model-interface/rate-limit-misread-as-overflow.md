---
type: failure
concepts: [context-overflow-detection]
harnesses: [pi]
---
**Symptom** A 429 rate limit, or Bedrock `ThrottlingException: Too many tokens, please wait`, was classified as context overflow. That triggered a needless lossy compaction instead of a backoff (#1038, #2699). Bodyless 400/413 errors from *any* provider were also treated as overflow (#9482).

**Root cause**
- The bodyless-status regex `^4(00|13|29)\s*(status code)?\s*\(no body\)` included 429 on the reasoning that "token rate limiting correlates with overflow".
- The generic `/too many tokens/i` fallback matched throttling text.
- The Cerebras bodyless heuristic was applied to every provider.

**Fix · [[pi]]**
- `25707f9ad` 2026-01-29: removed 429 from the bodyless pattern (#1038).
- `a3bf1eb39` 2026-03-30: `NON_OVERFLOW_PATTERNS` veto (`/^(Throttling error|Service unavailable):/i`, `/rate limit/i`, `/too many requests/i`) is evaluated before the overflow patterns. Bedrock errors are formatted with stable `${name}: ${message}` prefixes (`packages/ai/src/utils/overflow.ts:67-80,141`; `packages/ai/src/api/bedrock-converse-stream.ts:365-410`) (#2699).
- `661619e87` 2026-09-18: Cerebras `/^4(?:00|13)\s*(?:status code)?\s*\(no body\)/i` applies only when `provider === "cerebras"` (`overflow.ts:65,146-148`) (#9482).
- Custom-provider docs: "never rewrite rate limits as overflow" (`packages/coding-agent/docs/custom-provider.md:152-154`).

**Lesson** Throughput limits are not size limits. Run an explicit exclusion list before the generic overflow regexes, and gate provider-specific heuristics on provider identity.

Related: [[context-overflow-detection]] · [[overflow-recovery]] · [[auto-retry-backoff]] · [[error-text-breaks-retry-classification]] · [[overflow-message-not-recognized]] · [[pi--context-overflow-detection|pi]]
