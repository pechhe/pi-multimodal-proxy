# Security Review — pi-vision-proxy v1.4.0

**Reviewer**: AI-assisted code audit
**Date**: 2026-05-03
**Scope**: Full codebase — `extensions/internal.ts`, `extensions/vision-proxy.ts`, `package.json`
**Classification**: Extension runs locally in user's Pi agent — not a network service

---

## Executive Summary

The extension is well-designed with multiple layers of defense. Three issues require
immediate fixes (1 critical, 1 high, 1 medium). Several low-severity hardening
recommendations are also included.

---

## Findings

### CRITICAL

#### SEC-1: Interactive consent toggle missing `provider` field

**File**: `vision-proxy.ts:1732-1735`
**Severity**: CRITICAL
**CVSS**: N/A (local extension — consent bypass)

The interactive menu's consent toggle (triggered when user selects "Consent" from
the `/vision-proxy` config menu) creates a consent entry without the `provider` field:

```typescript
if (choice.startsWith("Consent")) {
    const granted = !hasConsent(entries);
    pi.appendEntry<ConsentEntry>(CUSTOM_TYPE_CONSENT, { granted });  // ← no provider!
    ...
}
```

This re-introduces the exact bug fixed in commit `8629d2c` for the `/vision-proxy consent`
slash command. Per-provider `hasConsent()` checks will skip provider-less entries, so
the consent is silently ineffective — the user believes consent is granted but auto-proxy
never activates.

**Fix**: Add `provider: effective.provider` to the `appendEntry` call.

---

### HIGH

#### SEC-2: `extractCandidateImagePaths` auto-reads files from `before_agent_start` without `..` check

**File**: `vision-proxy.ts:854-870`, `internal.ts:729-744`
**Severity**: HIGH (mitigated by `isPathAllowed`)
**Attack vector**: Model generates prompt text containing paths like `../../etc/shadow.png`

The `before_agent_start` handler auto-detects file paths in the user's prompt via
`extractCandidateImagePaths()` and reads them with `readImageFileWithReason()`. Unlike
`handleAnalyzeImage()` (line 590) and the describe handler (line 1435), this path does
**not** check for `..` in the path before reading.

**Current mitigation**: `isPathAllowed()` checks that the resolved path is under
`cwd`, `tmpdir`, or `homedir` (if opted in). The `realpath()` canonicalization prevents
symlink-based escapes. However, a model could craft a prompt containing
`/tmp/../../etc/passwd.png` — the `realpath` resolves to `/etc/passwd.png` which is
outside the allowed directories and would be rejected. This defense is sufficient.

**Recommendation**: Add an explicit `..` check for defense-in-depth, consistent with
the tool and describe handlers:

```typescript
for (const fp of filePaths) {
    if (fp.includes("..")) continue;  // ← add this
    const r = await readImageFileWithReason(fp);
    ...
}
```

---

### MEDIUM

#### SEC-3: `reason` parameter in `analyze_image` tool is stored in session entries without sanitization

**File**: `vision-proxy.ts:642,772`
**Severity**: MEDIUM
**Attack vector**: Model crafts a `reason` string containing custom entry data that could confuse other extensions or downstream consumers

The `reason` parameter (described as "logged for analytics only") is sliced to 200
characters and stored in `CUSTOM_TYPE_TOOL_CALL` session entries. While it doesn't
affect vision-proxy behavior, the unsanitized text could contain arbitrary content.

**Recommendation**: No immediate action needed — the field is purely for debugging and
is never rendered or executed. Consider adding a comment documenting the trust boundary.

---

### LOW

#### SEC-4: `readPersistentFile` trusts file content without schema validation

**File**: `internal.ts:396-405`
**Severity**: LOW (local attack only — attacker already has file-system write access)

`JSON.parse(raw)` output is cast directly to `Partial<VisionConfig>` and later
sanitized. However, between parse and sanitize, the object is spread into config:

```typescript
return sanitize({ ...DEFAULT_CONFIG, ...fileConfig, ...readPersistedConfig(entries), ...readEnvOverrides(env) });
```

A malicious `vision-proxy.json` could include `__proto__`, `constructor`, or
unexpected keys. Modern Node.js (v22) is not vulnerable to `__proto__` pollution via
spread, and `sanitize()` validates all fields. No practical attack vector exists.

**Recommendation**: Add explicit key filtering in `readPersistentFile` for defense-in-depth:

```typescript
const ALLOWED_KEYS = new Set(["mode","provider","modelId","systemPrompt","includeContext",...]);
const filtered = Object.fromEntries(
    Object.entries(parsed).filter(([k]) => ALLOWED_KEYS.has(k))
);
```

---

#### SEC-5: `stripImagePaths` uses user-controlled path strings as regex input

**File**: `internal.ts:759-768`
**Severity**: LOW (paths come from filesystem, not adversarial input)

The function escapes regex metacharacters (`.*+?^${}()|[]\`) before building a RegExp.
However, it does not escape `{` and `}` as these are only special in specific contexts.
This is safe for filesystem paths (which don't contain these characters in practice).

**Recommendation**: No action needed.

---

#### SEC-6: Null bytes in filenames not escaped by `escapeAttr`

**File**: `internal.ts:784-785`
**Severity**: INFORMATIONAL

`escapeAttr` does not escape null bytes (`\x00`). Node's `basename()` preserves null
bytes in filenames. In XML attribute contexts, null bytes are invalid but typically
handled gracefully by LLM consumers. Not a practical attack vector.

**Recommendation**: Consider adding `\x00` → `\uFFFD` replacement in `escapeAttr`.

---

#### SEC-7: No rate limiting on `analyze_image` tool calls

**File**: `vision-proxy.ts:503-782`
**Severity**: INFORMATIONAL

A model could call `analyze_image` in a loop, incurring API costs. The LRU cache
mitigates repeated identical calls, but different questions bypass the cache.

**Recommendation**: Consider adding a per-turn call limit (e.g., max 5 tool calls per
agent turn) in a future version.

---

## Positive Security Observations

1. **Defense-in-depth on file reads**: Three layers — `..` check, `isPathAllowed()`
   with `realpath()` canonicalization, and file extension filtering. Only one path
   (`before_agent_start` auto-detection) lacks the `..` check.

2. **Fence neutralisation**: `fenceUntrusted()` properly breaks closing tags for all
   three fence types, preventing injection attacks from vision model output. Tested
   with adversarial inputs — all pass.

3. **Consent per-provider**: The `hasConsent()` function correctly validates that
   consent entries include a matching `provider` field, preventing cross-provider
   consent leakage.

4. **Attribute escaping**: `escapeAttr()` covers `&`, `"`, `<`, `>` — sufficient for
   double-quoted XML attributes. The `dimensions` JSON in joint fences uses `&#39;`
   for single-quote escaping in the outer attribute.

5. **Sanitized user prompts**: `sanitizeXml()` wraps all user/model input in
   `<user_message>` / `<question>` tags with proper angle-bracket escaping.

6. **Memory bounds**: `_imageMeta` capped at 500 entries, `_toolCache` bounded by
   configurable `cacheSize` (default 50, max 500).

7. **File size limits**: `maxImageFileBytes()` defaults to 10 MB, configurable via
   `PI_VISION_PROXY_MAX_IMAGE_BYTES`.

8. **Persistent file path**: Hardcoded to `~/.pi/agent/vision-proxy.json` — no path
   injection possible.

9. **Input validation**: All numeric config values are range-checked. Provider/model
   strings validated against allowlist regex patterns (`PROVIDER_PATTERN`,
   `MODEL_ID_PATTERN`).

10. **No network surface**: The extension has zero listening ports, no webhooks, no
    HTTP server. All network calls go through Pi's `complete()` API.

---

## Dependency Audit

| Package | Version | Risk |
|---------|---------|------|
| `imagescript` | ^1.3.1 | Pure JS image codec. No native deps. Processes untrusted image data — potential for decode bombs (mitigated by 10 MB file limit). |
| `imghash` | ^1.1.4 | Pure JS perceptual hash. Lazy-loaded, wrapped in try/catch. |
| `image-size` | ^2.0.2 | Header-only dimension extraction. Minimal attack surface. |

**Note**: `imagescript` is AGPL-3.0 — acceptable for a local extension, but worth
documenting for users who may bundle it.

**Image decode bomb**: `imagescript` decodes the full image into memory. A 10 MB PNG
can decompress to hundreds of MB of pixel data. The `maxImageFileBytes` limit only
checks the compressed size. Consider adding a pixel-count limit (e.g., max 16K × 16K).

---

## Summary Table

| ID | Severity | Finding | Status |
|----|----------|---------|--------|
| SEC-1 | CRITICAL | Interactive consent missing `provider` | ❌ Fix required |
| SEC-2 | HIGH | `before_agent_start` lacks `..` check | ⚠️ Mitigated, add defense-in-depth |
| SEC-3 | MEDIUM | `reason` field stored unsanitized | ℹ️ Low risk — informational |
| SEC-4 | LOW | Persistent config lacks schema validation | ℹ️ Hardening |
| SEC-5 | LOW | Regex from path strings | ✅ Safe (metacharacters escaped) |
| SEC-6 | INFO | Null bytes in filenames | ℹ️ No practical impact |
| SEC-7 | INFO | No tool call rate limit | ℹ️ Future consideration |
