---
id: AUDIT-033
title: Targeted Security Audit v2.12.6 (ESLint-cleanup pass + i18n delta)
version: 2.12.6
date: 2026-05-30
scope: targeted
prior-audit: AUDIT-032 (v2.12.5)
triggers:
  - Release closure for v2.12.6 (chore/review-bot-score-pass commit fef39401)
  - i18n delta from FIX-04-03-08 / FIX-04-03-40 follow-up (commits 09419cf8, 3051f823)
verdict: Green
findings:
  resolved: 0
  deferred: 0
  pending: 0
  info: 1
  total: 1
---

# AUDIT-033 -- v2.12.6 Targeted Security Audit

## Scope and rationale

Two narrow triggers, not a full re-audit:

1. **chore/review-bot-score-pass** (commit `fef39401`, awaiting merge to dev): 16 files touched in a Review-Bot score pass that brings non-test ESLint errors from 22 to 0 and removes one duplicate CSS selector. All changes are type-only, code-style, or string-coercion edits with the explicit commit footer "No runtime behavior change."
2. **i18n delta from FIX-04-03-40 follow-up** (commits `3051f823` and `09419cf8`): the `modal.modelConfig.noModelsUrl` hint string was rewritten and then sentence-cased. Cosmetic UI text only.

Prior audit AUDIT-032 (v2.12.5, 2026-05-29) closed the `tmp` CVE override and confirmed the FIX-04-03-07 reasoning-content delta clean. Between AUDIT-032 and v2.12.6 the only prod-code changes are the items above plus regression tests for FIX-01-05-01 and FIX-01-07-01 (test-only commit `17d5017a`, out of scope for SAST per the audit conventions).

## Tech stack (unchanged from AUDIT-030 through AUDIT-032)

- Language: TypeScript (strict)
- Runtime: Electron (Obsidian), Node 20 in CI
- Major runtime deps: `@anthropic-ai/sdk`, `openai`, `@modelcontextprotocol/sdk`, `express` (only for MCP HTTP transport), `body-parser`, `sql.js`, `vectra`, `exceljs`
- 21 runtime deps, 19 dev deps; 461 production TS files, ~106k LOC outside tests
- Existing controls (carried from AUDIT-030): SafeStorage for API keys, `requestUrl` instead of `fetch`, atomic SQLite writes with journal, MCP token-in-URL auth, path-traversal protection on binary writes, npm overrides for prior CVEs (protobufjs, hono, dompurify, undici, brace-expansion, qs, tmp)

## SCA -- Software Composition Analysis

`npm audit` reports zero vulnerabilities across all severities (info, low, moderate, high, critical). No change since AUDIT-032; the `tmp >= 0.2.6` and `qs >= 6.15.2` overrides remain in place. `npm outdated` lists 24 packages, all minor or patch updates with no associated advisories; treated as routine maintenance, not as audit findings.

## Cleanup-pass review (commit fef39401)

Each of the 16 file changes was checked for security-relevant side effects. Walk-through by category:

### Type-assertion removals (no change in runtime behavior)

| File:line | Removed | Reason it was safe |
|---|---|---|
| `src/main.ts:3004` | `as ArrayBuffer` on `d.buffer.slice(...)` | `ArrayBuffer.prototype.slice()` already returns `ArrayBuffer`; the cast was a no-op |
| `src/ui/settings/utils.ts:91` | `as HTMLHeadingElement` on the return | `heading` is already typed `HTMLHeadingElement` from the `createEl('h2')` call upstream |
| `src/core/skills/SelfAuthoredSkillLoader.ts:914` | `as SelfAuthoredSkill['source']` | `fm.source` is already that union type from the frontmatter parser |
| `src/ui/settings/BackupTab.ts:293` | non-null `!` on `settings.backup` | A defensive `if (!settings.backup) settings.backup = {...}` block runs five lines earlier; control-flow guarantees non-null at the read site |
| `src/core/utils/NoticeCapture.ts:195, 217` | `as unknown` on the global Notice swap | The `globalRef.Notice` slot is already typed `unknown`; the cast was a redundant narrowing-then-widening |

None of these removals weaken any trust boundary; they remove redundant declarations the TypeScript compiler had already inferred.

### Type-assertion additions (narrowing, not widening)

| File:line | Added | Why safe |
|---|---|---|
| `src/core/backup/BackupExportService.ts:328` | `as BackupManifest` on `JSON.parse(...)` return | Pre-existing trust pattern: `readManifest` is a UI-side inspection helper; the actual ZIP integrity and schema-version check happens downstream in `unpackZip` (line 224, unchanged) before any manifest value is acted on. Cast widens nothing; it makes the implicit return type explicit. |
| `src/core/utils/NoticeCapture.ts:116` | `& { prototype: object }` on `OriginalNotice` cast | Allows the `PatchedNotice.prototype = OriginalNotice.prototype` mirror line to type-check without touching runtime semantics; the original cast already constrained `OriginalNotice` to a constructor type |
| `src/core/utils/NoticeCapture.ts:206-209` | Explicit `(cb, ms) => unknown` signature on the `setTimeout` fallback union | Eliminates the unsafe-call on the discriminated branch; no change in dispatch logic (`window.setTimeout` vs `globalThis.setTimeout` chosen by the same `typeof window !== 'undefined'` test as before) |
| `src/core/sandbox/RunSkillScriptCache.ts:83` | `as string \| undefined` on `Map.keys().next().value` | The store is typed `Map<string, string>`; the cast surfaces the type narrowing the iterator protocol already guarantees |
| `src/core/skills/SkillWriteInterceptor.ts:48, 50` | `as AdapterLike['write']` and `as AdapterLike['writeBinary']` on `.bind(...)` results | `Function.prototype.bind` returns a less-specific signature than the bound method; the cast recovers the precise signature without bypassing structural checks |

All additions are narrowing casts that re-state types the boundary already promises. None of them grant access to a broader value space at runtime.

### String coercion

| File:line | Change | Why safe |
|---|---|---|
| `src/core/backup/BackupExportService.ts:225` | `String(manifest.schemaVersion)` inside the error template | Defensive: schema-version is a typed numeric literal but TypeScript narrows it to `never` after the `!== 1` guard. `String(...)` keeps the error message readable; no injection vector since the value is a plain numeric field of the parsed manifest, not user input. |

### eslint-disable directive cleanup

| File:line | Change | Why safe |
|---|---|---|
| `src/core/checkpoints/GitCheckpointService.ts:29` | `eslint-disable-next-line @typescript-eslint/no-require-imports` moved to sit directly above the `require('fs')` line (was previously bridged by a multi-line comment, so the disable matched the comment line and never reached the code) | The `require('fs')` itself was already an established exception (isomorphic-git needs raw Node fs, repo lives outside the vault); only the disable-direction was wrong. Plugin is `isDesktopOnly: true`, so `require('fs')` is reachable in production. No new code path. |
| `src/core/sandbox/EsbuildWasmManager.ts:34` | Same fix for the cache-directory `require('fs')` | Same rationale; the cache-directory is in `.vault-operator/cache/` (outside the vault), so vault.adapter is not available. Pre-existing pattern. |
| `src/ui/settings/PermissionsTab.ts:48` | `eslint-disable-next-line prefer-const` on a forward-declared `let categoryContainer` | The variable is assigned exactly once (line 84) and only read inside a closure that fires after that assignment via toggle `onChange`. No runtime risk; structurally `let` is required to preserve DOM order (createDiv must come after the master toggles) |
| `src/core/tools/agent/ProbePluginTool.ts:57` | `eslint-disable-next-line @typescript-eslint/require-await` on `async execute` | The `AgentTool` interface contract requires `Promise<void>`; this tool happens to read in-memory app state synchronously. No promise-related race introduced. |

The file-level disable in `src/core/knowledge/RerankerService.ts` covers only `no-unsafe-assignment`, `no-unsafe-member-access`, `no-unsafe-call`, `no-unsafe-argument`, `no-unsafe-return`. It deliberately omits `no-explicit-any` (Pattern B from the review-bot skill); the bot prohibits disabling that rule even at file level. The boundary the disable covers is the untyped transformers.js / onnxruntime-web SDK whose model and tokenizer outputs cannot be statically typed. Inputs (query + candidate strings) are validated by callers in `SemanticIndexService`; outputs (scores) cross the boundary as plain numbers after a `Number()` coercion. No new attack surface.

### Arrow-function rewrite in SkillWriteInterceptor

```diff
-        const self = this;
-        this.adapter.write = async function (path, content) {
-            await self.maybeSnapshot(path);
-            return self.originalWrite!(path, content);
-        };
+        this.adapter.write = async (path, content) => {
+            await this.maybeSnapshot(path);
+            return this.originalWrite!(path, content);
+        };
```

Lexical-`this` arrow functions are semantically identical to the previous `const self = this` capture for this monkey-patch use. The patched callable runs with `this` bound to the SkillWriteInterceptor instance (intended) instead of the adapter (which the previous code already worked around with `self`). No change in patch behavior, no change in the call sequence (`maybeSnapshot` is still awaited before the delegated write).

### CSS-only change

`styles.css`: the duplicate `.agent-slider-value-editing` block was merged in place (Pattern N from the review-bot skill). No declarations or rules added; the union of properties from the two blocks is identical to the prior cascade. No selector-injection vector since the file is bundled, not user-authored.

### Async-callback rewrite in ExecuteCommandTool

```diff
-            const capture = await withNoticeCapture(
-                globalThis as { Notice?: unknown },
-                async () => {
-                    this.app.commands.executeCommandById(commandId);
-                },
-            );
+            const capture = await withNoticeCapture(
+                globalThis as { Notice?: unknown },
+                () => {
+                    this.app.commands.executeCommandById(commandId);
+                },
+            );
```

`withNoticeCapture` accepts `() => Promise<T> | T` (NoticeCapture.ts:106), so dropping the `async` from a callback that never `await`s anything is signature-compatible. The patch / restore semantics in `withNoticeCapture` are unchanged. No change to the AUDIT-FEAT-29-03+04 controls (token-format redaction, max-captures bound, per-notice 500-char cap, module-level singleton guard).

## i18n delta review (commits 3051f823, 09419cf8)

Single string changed in `src/i18n/locales/en.ts:1255`:

```diff
-'No models found at this base URL. If this endpoint does not implement /v1/models, just type the model ID into the Model ID field above and save.'
+'No models found at this base URL. If this endpoint does not implement /v1/models, just type the model ID into the field above and save.'
```

Pure cosmetic edit. The string is consumed by `ModelConfigModal` and rendered via `setText` on a `<p>` element (no `innerHTML`, no template interpolation). No security impact.

## Categories checked

| Category | Verdict | Notes |
|---|---|---|
| Injection (A03 OWASP) | Clean | No new string-template flows; the one new template literal (`String(manifest.schemaVersion)`) reads a typed numeric field from a JSON-parsed manifest already validated by `JSZip.loadAsync` |
| Broken authentication (A07) | Clean | No auth-flow touches |
| Cryptographic failures (A02) | Clean | No crypto-relevant changes |
| Insecure design (A04) | Clean | All cleanup is structural; no design decisions altered |
| Security misconfiguration (A05) | Clean | The two repositioned eslint-disable directives now match the line they intend to disable. No effective change in code behavior, only in lint coverage. |
| Vulnerable & outdated components (A06) | Clean | `npm audit` zero findings; outdated list contains no advisories |
| Identification & authentication failures (A07) | Clean | No auth changes |
| Software & data integrity failures (A08) | Clean | `readManifest` integrity is enforced downstream in `unpackZip` (schema-version check), unchanged. The new explicit cast on `readManifest` does not bypass any validation. |
| Logging & monitoring (A09) | Clean | No log-format changes |
| SSRF (A10) | Clean | No URL handling changes |
| Prompt injection (LLM01) | Clean | No prompt-template changes |
| Insecure output handling (LLM02) | Clean | No output rendering changes |
| Sensitive info disclosure (LLM06) | Clean | NoticeCapture redaction patterns unchanged (keywords + GitHub PAT, OpenAI sk, JWT, generic hex); the only NoticeCapture edits were the type cast and the setTimeout-fallback signature, both type-only |
| Insecure plugin design (LLM07) | Clean | ProbePluginTool and ExecuteCommandTool retain the same allow-list enforcement; only the `async` decoration changed |
| Excessive agency (LLM08) | Clean | No tool-permission changes |
| XSS in UI | Clean | i18n string is rendered via `setText`; no `innerHTML` paths added |
| Race conditions | Clean | The arrow-function rewrite in `SkillWriteInterceptor` preserves the await-before-write sequence; `withNoticeCapture` still uses the module-level singleton guard from M-1 |
| Hardcoded credentials | Clean | No new strings |

## Verdict

**Green.** Zero new findings across all checked categories. The cleanup commit is type-only and code-style; the i18n delta is cosmetic. All trust boundaries that the changes touch (BackupExportService manifest read, SkillWriteInterceptor monkey-patch, NoticeCapture global swap, RerankerService SDK boundary, sandbox require) carry over from prior audits unchanged.

## Findings

### I-1 -- i18n cosmetic edit (`modal.modelConfig.noModelsUrl`)

- **Severity:** Info
- **CWE:** n/a
- **Source:** Commits `3051f823`, `09419cf8`
- **Affected file:** `src/i18n/locales/en.ts:1255`
- **Scope:** UI string only
- **Status:** Documented, no remediation needed
- **Risk:** None. The string is rendered via `setText` (no innerHTML), interpolates no user input, and references the same UI field as before.
- **Note:** Listed for completeness so the post-AUDIT-032 delta is fully accounted for.

## Release recommendation

Merge `chore/review-bot-score-pass` into dev and roll into the next release alongside the regular `dev -> main` flow. No standalone release needed for the audit itself. The release notes for v2.12.6 should reference the Review-Bot score pass (22 ESLint errors to 0, one CSS duplicate removed); no security note required since no advisory closed in this audit.

The next periodic full-codebase re-audit becomes due around 2026-06-19 (one month after AUDIT-030). Until then, continue with targeted per-release audits when a Dependabot alert opens or a non-trivial feature lands.
