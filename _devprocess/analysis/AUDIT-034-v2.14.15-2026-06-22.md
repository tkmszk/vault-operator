---
id: AUDIT-034
date: 2026-06-22
scope: Full-codebase audit v2.14.15 since AUDIT-033 (delta v2.12.6 -> v2.14.15)
auditor: sebastian-claude-opus-4-7
audit-type: periodic-full (branch feature/audit-2026-06-22)
prior-audits:
  - AUDIT-033 v2.12.6 (targeted, Green)
  - AUDIT-024 v2.11+ (full, LOW)
verdict: Resolved
findings:
  critical: 0
  high: 5
  medium: 28
  low: 36
  info: 10
  resolved: 32
  total: 79
---

# AUDIT-034 -- Vault Operator v2.14.15 Full Security Audit

## Scope and rationale

This audit covers the delta from v2.12.6 (last targeted clean line in AUDIT-033, 2026-05-30) to v2.14.15 (current release on dev, 2026-06-21). The delta contains Retrieval Wave 1 (Opener-Excerpt, weighted RRF, batched reranker, graph verb labels), Memory v2 Phase 6 (HistoryDB + HistoryIndexer + search_history), EPIC-32 Stigmergy integration (StigmergyAdapter, precedenceResolver, Pinned-Sequence Direct Promotion), the plan_presentation + create_pptx pipeline maturity (Template + Adhoc modes), reasoning-effort steering across all default providers (Anthropic, OpenAI, Bedrock, OpenRouter, Codex OAuth, Kilo Gateway, GitHub Copilot), Bedrock extended thinking via budget_tokens, plus a string of FIX items including FIX-04-03-07 (ThinkingBlock passback), FIX-01-07-03 (Editor-View cache write coherence), and several provider corrections.

Methodology follows the five-phase pattern from AUDIT-024: (1) SAST against the new and changed source under src/, (2) OWASP Top 10 mapped against runtime trust boundaries, (3) OWASP LLM Top 10 mapped against agent and prompt flows, (4) SCA against package.json plus transitive lockfile entries, (5) Zero Trust + code quality across module boundaries. Every confirmed finding was passed through an adversarial verifier that re-reads the cited lines and either confirms the technical claim and severity calibration, or rejects the finding into the dropped-candidates appendix. Dropped candidates remain searchable for traceability.

## Executive Summary

| Analysis Domain | Critical | High | Medium | Low | Info |
| --- | --- | --- | --- | --- | --- |
| SAST | 0 | 2 | 5 | 4 | 3 |
| OWASP Top 10 | 0 | 1 | 7 | 8 | 5 |
| OWASP LLM Top 10 | 0 | 0 | 2 | 1 | 0 |
| SCA | 0 | 0 | 0 | 1 | 1 |
| Zero Trust + Code Quality | 0 | 2 | 14 | 22 | 1 |
| **Total** | **0** | **5** | **28** | **36** | **10** |

Overall risk verdict is High, driven by five credential-storage and path-traversal findings clustered on two architectural seams: the GlobalSettingsService dual-write that bypasses encryption applied in main.ts, and the EditFileTool/ReadFileTool/WriteFileTool path validation gap on hidden-folder fallbacks. Release recommendation is YELLOW: do not ship v2.15.0 publicly without resolving the four credential-encryption Highs (SAST-C-01, SAST-C-02, OWASP-A02-1, plus the EditFileTool path-traversal SAST-1-02). The biggest concern is the desynchronization between main.ts encryptSettingsForSave (covers chatgptOAuth* and providerConfigs) and GlobalSettingsService.encryptGlobal (does not). The most cost-effective fix is mirroring the main.ts walker into GlobalSettingsService, which closes SAST-C-01 + SAST-C-02 + OWASP-A02-1 in a single PR. Carryover from prior audits: AUDIT-003 H-1 (prompt-injection-by-design at permissive) remains by design; AUDIT-027 H-1 (provider credential encryption) is resolved on the data.json path but the global path was missed; FIX-12 atomic SQLite write is verified intact and now applies to HistoryDB by inheritance.

## Tech stack (delta from AUDIT-033)

- @huggingface/transformers 4.2.0 (new): local cross-encoder reranker (RerankerService). ONNX runtime loaded via OptionalAssetManager with SHA-256 integrity gate (RERANKER_WASM_SHA256).
- @agentic-stigmergy/client + @agentic-stigmergy/loop (new): local-socket-only daemon recall layer. Dynamic import with no-op fallback, unix-socket transport via STIGMERGY_SOCK or ~/.stigmergy/pheromone.db.daemon.sock.
- @smithy/node-http-handler bump for Bedrock extended-thinking streaming.
- @aws-sdk/client-bedrock-runtime 3.1036+: budget_tokens parameter, message_delta stop_reason wiring for truncation-recovery.
- openai 5.23.2 pin held (transitive zod advisory, see OWASP-A06-01).
- pptx-automizer + PptxGenJS continue as the dual-mode PPTX backend; pdfjs-dist + jszip continue as document parser deps.

## Existing controls (carried)

- SafeStorage encrypt/decrypt round-trip on data.json continues to function on macOS and Windows (SafeStorageService.ts:53-95). Linux without libsecret still triggers the documented plaintext fallback (see OWASP-A07-02).
- requestUrl from obsidian replaces fetch for all first-party HTTP. WebFetchTool keeps the two-phase isPrivateIP + DNS-resolve SSRF guard (WebFetchTool.ts:108-142), with gaps noted in SAST-1-04 and SAST-R-08.
- Atomic SQLite write + integrity_check + auto-recovery + lock-file + daily snapshots (FEATURE-0314 / ADR-079) verified intact in KnowledgeDB. HistoryDB inherits the per-write atomic guard but is missing from the snapshot target list (OWASP-A04-4).
- MCP token-in-URL auth + constant-time compare + 127.0.0.1 bind verified intact (McpBridge.ts:399, RelayClient.ts:38-51).
- Path-traversal helper validateVaultRelativePath (pathValidation.ts:22-29) is the project-standard control. Used by IngestTriageTool, MarkNoteAsMemorySource, UnmarkNoteAsMemorySource, writeBinaryToVault. NOT used by WriteFileTool, EditFileTool, ReadFileTool (see SAST-1-01, SAST-1-02, SAST-1-03).
- npm overrides for protobufjs, hono, dompurify, plus 18 prior CVE overrides remain in place.
- Approval pipeline (ToolExecutionPipeline.checkApproval) continues to gate write/web/MCP tools when autoApproval sub-flags are off, which they are by default.
- DEGRADED: encryptGlobal in GlobalSettingsService has not kept pace with main.ts encryptSettingsForSave. Five credential-class fields (chatgptOAuth*Token x 3, plus providerConfigs[].credentials, plus legacy_active_models_backup) are encrypted on the data.json path but plaintext on the global settings.json path. This is a regression of an established control.

## Findings

### H-1: EditFileTool hidden-folder fallback reads and modifies arbitrary filesystem paths

- **ID**: SAST-1-02
- **CWE / Category**: CWE-22 Path Traversal
- **Severity**: High
- **Confidence**: Confirmed
- **Location**: src/core/tools/vault/EditFileTool.ts:86-101, 129, 169
- **Risk**: `isHidden = path.split('/').some((seg) => seg.startsWith('.'))` routes any path whose any segment starts with a dot through vault.adapter.read/write with no `..` rejection. A path like `.foo/../../etc/passwd` (segment `.foo` starts with `.`, isHidden true) skips Obsidian's TFile index and reads the absolute filesystem location. The corresponding modify step writes back via the same adapter. Both a read and a write primitive outside the vault, gated only by an approval that is routinely auto-enabled for note edits.
- **Remediation**: Insert `validateVaultRelativePath(path)` at the top of execute, reject on null, and use cleanPath for the hidden-folder check, exists call, read call, and every subsequent adapter.write / vault.modify. Mirror the pattern already shipped in MarkNoteAsMemorySourceTool, UnmarkNoteAsMemorySourceTool, and IngestTriageTool. Add a unit test that calls edit_file with `.foo/../../etc/passwd` and asserts the tool throws before touching the adapter.
- **Verifier note**: Confirmed. Gate is permissive (any dot segment passes), no NUL-byte check, no encoded-traversal check. Reachable via prompt injection plus autoApproval.noteEdits, which is commonly enabled. The Obsidian FileSystemAdapter on desktop joins via Node which collapses `..`, giving arbitrary read AND write outside the vault.
- **Status**: Resolved

### H-2: ChatGPT OAuth tokens written plaintext to global settings.json

- **ID**: SAST-C-01
- **CWE / Category**: CWE-256 Plaintext Storage of Credentials
- **Severity**: High
- **Confidence**: Confirmed
- **Location**: src/core/storage/GlobalSettingsService.ts:155-197
- **Risk**: main.ts encryptSettingsForSave correctly encrypts chatgptOAuthAccessToken, chatgptOAuthRefreshToken, chatgptOAuthIdToken via SafeStorage (lines 2982-2991). plugin.saveSettings then ALSO calls GlobalSettingsService.saveGlobal (main.ts:3014), which writes a parallel copy to {vault-parent}/vault-operator-shared/settings.json. encryptGlobal does not include the three chatgptOAuth* fields, and VAULT_LOCAL_KEYS does not list them either, so they land plaintext. Refresh token is long-lived (~30d), id_token carries email and accountId. The global file sits in a directory the codebase markets as syncing via iCloud/OneDrive.
- **Remediation**: Add encrypt/decrypt branches for the three chatgptOAuth* token fields in encryptGlobal and decryptGlobal, mirroring the main.ts pattern. Add a one-time onload migration to rewrite the existing global file. Add a round-trip unit test that writes a settings blob with these fields via saveGlobal, reads the raw file, asserts the plaintext substring is absent.
- **Verifier note**: Confirmed via direct read of encryptGlobal (only walks the 8 listed fields, not chatgptOAuth*). VAULT_LOCAL_KEYS does not list them. saveGlobal copies the in-memory plaintext. Structurally identical to the AUDIT-007 H-1 precedent the file comments cite.
- **Status**: Resolved

### H-3: providerConfigs[] credentials written plaintext to global settings.json

- **ID**: SAST-C-02
- **CWE / Category**: CWE-256 Plaintext Storage of Credentials
- **Severity**: High
- **Confidence**: Confirmed
- **Location**: src/core/storage/GlobalSettingsService.ts:155-197 (vs src/core/security/providerCredentialCrypto.ts:32-77)
- **Risk**: EPIC-26 introduced providerConfigs[] with seven credential fields (apiKey, awsApiKey, awsAccessKey, awsSecretKey, awsSessionToken, gatewayHeaderValue, oauthToken). main.ts:2962 routes the full settings copy through encryptProviderCredentialsInPlace before saveData. GlobalSettingsService.encryptGlobal does not import or call this walker, so providerConfigs[] is written plaintext to vault-operator-shared/settings.json. AWS secret access keys grant billable InvokeModel access against the user's Bedrock account. Same exposure applies to legacy_active_models_backup. File mode 0o644 in a sync-prone folder amplifies blast radius.
- **Remediation**: Import encryptProviderCredentialsInPlace / decryptProviderCredentialsInPlace in GlobalSettingsService.ts and call them inside encryptGlobal / decryptGlobal. Walker is idempotent thanks to isEncrypted guards, so a one-time re-encrypt of the existing global settings.json is safe.
- **Verifier note**: Confirmed. encryptGlobal only walks 8 leaf fields (activeModels[].apiKey, webTools.{brave,tavily}ApiKey, githubCopilot* x 2, kiloToken, cloudflareApiToken, relayToken, mcpServerToken). providerConfigs[] and legacy_active_models_backup not in VAULT_LOCAL_KEYS, so saveGlobal copies them and encryptGlobal leaves them plaintext. The walker exists, is tested, and is the right fix.
- **Status**: Resolved

### H-4: GlobalSettingsService writes ChatGPT OAuth tokens AND providerConfigs[] credentials in plaintext

- **ID**: OWASP-A02-1
- **CWE / Category**: CWE-312 Cleartext Storage of Sensitive Information
- **Severity**: High
- **Confidence**: Confirmed
- **Location**: src/core/storage/GlobalSettingsService.ts:24-48, 155-197; cf. src/main.ts:2940-3005
- **Risk**: Composite OWASP A02 framing of SAST-C-01 + SAST-C-02. data.json path is encrypted; the dual-written global file contains the same secrets in cleartext on the user's home drive. Anyone with file-system access (backup, malware, sync provider) reads them directly. decryptGlobal is also missing the same fields, so a future re-encryption attempt would misread the file. The pattern that mcpServerToken / relayToken / kiloToken / githubCopilot* are carefully encrypted in both places makes the omission look like a regression, not an intentional gap.
- **Remediation**: Mirror main.ts in GlobalSettingsService.encryptGlobal and decryptGlobal: encrypt/decrypt the three chatgptOAuth* fields, plus call encryptProviderCredentialsInPlace / decryptProviderCredentialsInPlace on the global subset. Extract the walker into a shared helper so the next added field cannot desync the two paths. Add a regression test that round-trips a blob containing providerConfigs[].credentials.apiKey and an OAuth refresh token through saveGlobal and asserts the plaintext substring is absent in the raw file.
- **Verifier note**: Confirmed via direct read. Both paths in GlobalSettingsService skip these fields; main.ts encryptSettingsForSave covers them. Structurally identical to AUDIT-027 H-1.
- **Status**: Resolved

### H-5: openai 5.23.2 transitively pulls vulnerable zod (CVE-2026-6991)

- **ID**: OWASP-A06-01
- **CWE / Category**: CWE-1395 Vulnerable and Outdated Components
- **Severity**: High (npm-audit-assigned; effective Low given no reachable sink)
- **Confidence**: Confirmed
- **Location**: package.json:74 ("openai": "^5.23.2"); npm ls openai -> openai@5.23.2 -> zod@3.25.76
- **Risk**: npm audit reports advisory range openai 4.55.0 - 5.23.2 caused by zod <=4.4.0-canary, CWE-74 Injection / CWE-89, CVSS 5.3. Suggested fix is openai@6.44.0 (SemVer major). The plugin's 4 provider files import the OpenAI default client; none uses openai/helpers/zod, and no `from 'zod'` runtime import exists in src/. Vulnerability exists in the dependency tree but no reachable sink + sensitive-sink combination exists.
- **Remediation**: Add an npm override pinning zod to the patched line. Steps: verify zod's patched release via `npm view zod versions --json` and the advisory's GHSA detail, add `"zod": "<patched-version-spec>"` to package.json overrides between `"js-yaml"` and `"esbuild"`, rm -rf node_modules package-lock.json and npm install, re-run npm audit to confirm the advisory drops out, npm run build + npm run test. Track as FIX-SCA-A06-01 with effective severity Low. If the override path fails due to a hard openai 5.x pin, fall back to the openai 5.x -> 6.x bump and schedule it as a v2.15.x minor with provider regression tests.
- **Verifier note**: Advisory real, no reachable sink in this codebase. Confirmed because it sits in the supply chain and clean SCA is part of the project's track record (18 prior CVE overrides). Override is lower blast radius than the SemVer-major openai bump.
- **Status**: Resolved

### M-1: WriteFileTool.writeViaAdapter bypasses path validation via prefix-only check on configDir/agentDir

- **ID**: SAST-1-01
- **CWE / Category**: CWE-22 Path Traversal
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/tools/vault/WriteFileTool.ts:141-210
- **Risk**: WriteFileTool routes any path beginning with `${cfgDir}/` (default `.obsidian/`) or `${agentDir}/` through writeViaAdapter, which calls vault.adapter.write and adapter.mkdir with the raw input. The branch is gated by `path.startsWith(...)` only, no `..` or absolute-path rejection. An LLM tool-call with `path = '.obsidian/../../tmp/pwned.md'` satisfies the prefix check; Obsidian's FileSystemAdapter resolves relative to basePath, writing outside the vault. The Vault.create / Vault.modify branch is safe; the adapter fall-through punches through that guard. ADR-031 documents writeBinaryToVault doing the right thing but this path is not covered.
- **Remediation**: Apply `validateVaultRelativePath` at the top of WriteFileTool.execute and inside writeViaAdapter as defense-in-depth. Use the normalized return value for all downstream calls (existence check, adapter.write, vault.create, vault.modify, ensureFolderExists).
- **Verifier note**: Confirmed. IgnoreService.normalize only collapses backslashes and leading slashes (no `..` resolution); inputSchemaValidator has no path validation; Pipeline.validatePaths delegates to IgnoreService (same gap); approval gate auto-approves when noteEdits is on; MCP boundary is safe via the fail-closed branch. The fix already ships next door as validateVaultRelativePath and writeBinaryToVault.
- **Status**: Resolved

### M-2: ReadFileTool adapter fallback reads arbitrary files outside the vault

- **ID**: SAST-1-03
- **CWE / Category**: CWE-22 Path Traversal
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/tools/vault/ReadFileTool.ts:96-122
- **Risk**: When getAbstractFileByPath misses, ReadFileTool falls back to vault.adapter.exists then vault.adapter.read with the unmodified input. looksLikeExternalisedTmpPath rejects `..` only on the BUG-020 redirect branch; the primary fallback accepts any path. Obsidian's FileSystemAdapter resolves relative to basePath via Node which collapses `..`, so injected `read_file({path:'../../etc/passwd'})` returns the file body to the agent. The agent can then exfiltrate via web or MCP tools.
- **Remediation**: Apply validateVaultRelativePath at the entry of ReadFileTool.execute and use the returned normalized path for both the getAbstractFileByPath lookup and the adapter fallback. Extend to EditFileTool's adapter fallback (covered in H-1). Add a regression test that calls `read_file({path:'../outside.md'})` against a fake adapter and asserts a validation error rather than a read result.
- **Verifier note**: Confirmed. ReadFileTool is isWriteOperation=false, no approval card. MCP wire path is guarded by validateMcpVaultPath; the in-process agent path is not.
- **Status**: Resolved

### M-3: WebFetchTool private-IP allowlist misses CGNAT, IPv4-mapped IPv6, and broadcast

- **ID**: SAST-1-04
- **CWE / Category**: CWE-918 SSRF
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/tools/web/WebFetchTool.ts:20-42
- **Risk**: isPrivateIP covers RFC1918 + loopback + link-local + IPv6 ULA/link-local + ::1, but misses 100.64.0.0/10 (CGNAT), 255.255.255.255 (limited broadcast), 224.0.0.0/4 (multicast), and `::ffff:127.0.0.1` (IPv4-mapped IPv6 loopback). The IPv6 link-local check uses `startsWith('fe80')` which misses fe81-febf. Verified IPv4-mapped IPv6 bypass: `http://[::ffff:7f00:0001]/admin` parses with hostname `[::ffff:7f00:1]`, isPrivateIP returns false on both branches, DNS phase 2 throws and is swallowed, requestUrl reaches loopback.
- **Remediation**: After URL parse, strip IPv6 brackets before isPrivateIP. Add IPv4-mapped IPv6 recursion. Tighten the IPv4 branch with CGNAT (100.64-127), multicast (224-239), and broadcast (240-255). Tighten IPv6 link-local to fe80::/10 strictly via `/^fe[89ab][0-9a-f]:/`. Add unit tests for: 100.64.0.1, 255.255.255.255, 224.0.0.1, [::ffff:127.0.0.1], [::ffff:7f00:1], febf::1, [fe80::1%eth0].
- **Verifier note**: Confirmed. Web tools default-disabled lowers baseline but the threat applies the moment the user enables them. Loopback reachability is the strongest gap.
- **Status**: Resolved

### M-4: ToolStepsHtml rehydration deserializes stored HTML into the live DOM

- **ID**: SAST-1-01 (display dup; canonical id is the AgentSidebarView XSS rehydration item)
- **CWE / Category**: CWE-79 DOM XSS via stored HTML
- **Severity**: Low (verifier-revised; finding originally Low)
- **Confidence**: Confirmed
- **Location**: src/ui/AgentSidebarView.ts:3355-3373 (renderMarkdownMessage) + capture sites 2300, 2501
- **Risk**: On every assistant turn, stepsBlockEl.outerHTML is serialized into uiMessages and persisted to history JSON. On reopen, the stored string is parsed via DOMParser and inserted via importNode + appendChild. DOMParser parses inertly but preserves event-handler attributes, javascript: URLs, and active iframe srcdoc. All current writers are first-party safe; the live risk today is theoretical. If an attacker ever gains write access to the conversation JSON (untrusted sync, malicious MCP saveConversation-like flow), the next chat reload silently activates DOM-XSS in the Electron renderer.
- **Remediation**: Stop persisting raw HTML and rebuild from a structured toolLedger snapshot on rehydration, OR sanitize the parsed tree before append (strip script/iframe/object/embed/link/meta + remove on* attributes + reject javascript: URLs). Route through DOMPurify 3.4.7 which is already pinned via npm override.
- **Verifier note**: Confirmed pattern. All current writers are setText/createEl/createSpan, MCP saveConversation does not populate toolStepsHtml; exploitation requires direct JSON tampering. Defensive boundary is missing.
- **Status**: Resolved

### M-5: Plaintext fallback path stores API keys plaintext when SafeStorage is unavailable, no UI signal

- **ID**: SAST-C-03 (composite with OWASP-A07-02)
- **CWE / Category**: CWE-256 Plaintext Storage of Credentials
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/security/SafeStorageService.ts:67-76 + src/main.ts:2940-3005 + src/core/storage/GlobalSettingsService.ts:155-159
- **Risk**: When safeStorage.isEncryptionAvailable returns false, SafeStorageService.encrypt silently returns plaintext (line 68) and the settings copy is marked _encrypted=false. The user is warned only via console.warn, never via UI Notice. API keys (Anthropic / OpenAI / Bedrock / Brave / Tavily), OAuth refresh tokens, MCP relay/server tokens are written PLAINTEXT to data.json and global settings.json. Triggered by Linux without libsecret, macOS Keychain edge cases, headless CI. data.json under .obsidian/plugins/vault-operator/ is replicated by Obsidian Sync.
- **Remediation**: Surface the degraded state via a persistent Notice + a settings-tab banner when isAvailable returns false, on every onload where _encrypted=false AND any secret field is non-empty. Add a "Secrets unprotected" badge in Provider settings. As a stronger option, refuse to persist new secrets and keep them in-memory only when no keychain is available.
- **Verifier note**: Confirmed at all three locations. Plaintext fallback verified, no Notice / banner / modal exists. Real on Linux installs; OAuth refresh tokens and MCP relay/server tokens are long-lived, so blast radius is meaningful.
- **Status**: Partial (deferred to backlog)

### M-6: Global settings.json written with default umask, no 0o600 clamp

- **ID**: SAST-C-04
- **CWE / Category**: CWE-732 Insecure File Permissions
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/storage/GlobalFileService.ts:117-122
- **Risk**: GlobalFileService.write writes settings.json (plus rules, workflows, skills, memory, history) via fs.promises.writeFile with no mode option. On Unix this leaves files at the umask default (0o644, world-readable). McpBridge.writeMcpTokenFile already uses { mode: 0o600 } and migrateAgentLayout.ts:185-194 already chmods backups, so the precedent and pattern exist. Even after H-2/H-3 are fixed, the encrypted blobs are tied to the OS keychain identity; on a shared workstation any local user account can still read account email, plan tier, accountId, conversation history, memory facts (all PII).
- **Remediation**: In write, writeBinary, and append, pass `{ mode: 0o600 }` on create AND chmod after every overwrite (writeFile's mode option only applies on create). Skip chmod on Windows (relies on user-profile ACLs). Reuse the pattern from migrateAgentLayout.ts:190.
- **Verifier note**: Confirmed at all three call sites. PII at risk verified (ChatGptOAuthService.ts:153-155 persists accountId and planTier; history transcripts and memory facts go through the same write path).
- **Status**: Resolved

### M-7: ZIP decompression bomb check happens AFTER full decompression in OOXML helpers

- **ID**: SAST-R-01
- **CWE / Category**: CWE-400 Uncontrolled Resource Consumption
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/document-parsers/parsers/ooxmlHelpers.ts:42-46
- **Risk**: openZipSafe / getXmlDoc rely on a cumulative MAX_DECOMPRESSED_SIZE (500 MB) check, but `file.async('text')` materializes the FULL decompressed text in V8 memory before the tracker updates. A single XML entry crafted with a 99.99% compression ratio (KB-sized ZIP expanding to >500 MB inside one entry) blows past the cap before the check runs. Reachable from AttachmentHandler.processFile, read_document, ingest_document, and SemanticIndexService auto-indexing.
- **Remediation**: Apply the pre-decompression size check pattern from src/core/utils/extractZip.ts:85-94. In openZipSafe, iterate zip.files once, sum `_data.uncompressedSize` per entry, reject when any single entry exceeds 50 MB OR cumulative advertised size exceeds 500 MB. Helper already exists at extractZip.ts:222-225 and can be lifted into ooxmlHelpers.ts. Add a regression test under document-parsers/parsers/__tests__/.
- **Verifier note**: Confirmed. JSZip's `_data.uncompressedSize` is exposed and the codebase already uses it correctly in extractZip.ts. Same fix is mechanical. DoS only (Electron renderer OOM), no RCE.
- **Status**: Resolved

### M-8: XlsxParser allocates O(maxCol) cells per row based on attacker-controlled column reference

- **ID**: SAST-R-02
- **CWE / Category**: CWE-400 Uncontrolled Resource Consumption
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/document-parsers/parsers/XlsxParser.ts:88-133
- **Risk**: colLetterToIndex accepts any string of column letters; `r="ZZZZZZ1"` yields colIdx ~321M, and the rendering loop iterates `c=0..maxCol` for every kept row. 200 rows * 321M = 64 billion cells.push calls. Single hostile cell hangs the renderer and exhausts heap. Same three entry points as M-7. Row index has the same shape via `r="A99999999999"` (verifier addition).
- **Remediation**: Clamp BOTH column (EXCEL_MAX_COL = 16384) and row (EXCEL_MAX_ROW = 1048576) indices to Excel hard limits, plus validate the cell-ref format via `/^[A-Z]{1,3}[1-9]\d{0,6}$/` before int conversion. Apply inside the cells loop before updating maxCol/maxRow. Tests for `ZZZZZZ1` skipped, `A99999999999` skipped, `XFD1048576` accepted, malformed `""` / `1A` / `A0` skipped.
- **Verifier note**: Confirmed. The row dimension is the same shape and the proposed col-only clamp is incomplete; both clamps are required. Reliably exploitable with attacker-controlled file content.
- **Status**: Resolved

### M-9: Custom-Agent toolGroups filter is schema-only; pipeline never enforces it at runtime

- **ID**: OWASP-A01-1
- **CWE / Category**: CWE-285 Improper Authorization
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/tool-execution/ToolExecutionPipeline.ts:357-377; src/core/modes/ModeService.ts:133
- **Risk**: ModeService.getToolDefinitions filters the LLM schema by mode.toolGroups, but executeTool only checks toolRegistry.getTool. modeHasTool exists but has zero production callers. Three dispatch paths bypass: the model path forwards the tool name verbatim, FastPathExecutor dispatches recipe-selected tools regardless of mode, executeVaultOp lets external MCP clients call any registered tool. A read-only Custom Agent built by stripping `edit` from toolGroups is not actually protected; the model can still emit write_file. autoApproval.noteEdits on the Default agent silently auto-approves writes from the supposedly read-only agent.
- **Remediation**: Gate executeTool by mode for model-driven and MCP-client dispatch. Inject ModeService, add the check after registry lookup with a source-based bypass: when opts.source is 'model' (or undefined) or 'mcp', enforce; when opts.source is 'fastpath' or 'recipe', allow (recipes are user-authored). Pass source 'mcp' from executeVaultOp. Subtask spawner reads the active mode at dispatch time. Tests for hallucinated write_file rejection, FastPath/recipe bypass, MCP rejection, subtask gating. Update i18n to clarify recipes can override the boundary.
- **Verifier note**: Confirmed. UI explicitly advertises the boundary (i18n/locales/en.ts:266). Contract violation is real and user-observable.
- **Status**: Resolved

### M-10: IgnoreService validation only inspects input.path; multi-path write tools bypass governance files

- **ID**: OWASP-A01-2
- **CWE / Category**: CWE-22 Path Traversal (governance bypass)
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/tool-execution/ToolExecutionPipeline.ts:635-651; MoveFileTool.ts:36-93; ExtractZipTool.ts:68-104
- **Risk**: validatePaths reads only toolCall.input?.path. move_file (source/destination), extract_zip (zip_path/target_folder), generate_canvas (source), plan_presentation (source) never pass through it. A user with `Private/` in .obsidian-agentignore is still readable via move_file (rename to a non-ignored path then read), and a protected file is still moveable or overwriteable via move_file. extract_zip can drop files into ignored/protected folders. None of these tools consult IgnoreService internally.
- **Remediation**: Extend validatePaths to iterate a per-tool key list: move_file -> [source, destination], extract_zip -> [zip_path, target_folder], generate_canvas -> [source, output_path], plan_presentation -> [source], restore_checkpoint -> [path]. Run isIgnored on read-side keys and isIgnored+isProtected on write-side keys. Add a regression test that asserts move_file(source=Private/x.md, destination=Public/x.md) is denied. Consider hard fail-closed when a tool is missing from the key list AND has unknown path-like inputs to prevent future drift.
- **Verifier note**: Confirmed. With vaultChanges auto-approval, exploitation requires only prompt injection. Defeats user-configured .obsidian-agentignore / .obsidian-agentprotected.
- **Status**: Resolved

### M-11: configure_model is auto-approved despite isWriteOperation=true

- **ID**: OWASP-A01-3
- **CWE / Category**: CWE-269 Improper Privilege Management
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/tool-execution/ToolExecutionPipeline.ts:166-200, 664-682; src/core/tools/agent/ConfigureModelTool.ts:20-27
- **Risk**: ConfigureModelTool sets isWriteOperation=true, but TOOL_GROUPS classifies configure_model as 'agent', and checkApproval returns `{ decision: 'auto' }` for every agent-group tool except update_settings touching autoApproval. configure_model can add a new model + apiKey + baseUrl, switch the active model, or overwrite apiKey/baseUrl with no user prompt. SSRF guard validateProviderUrl catches the worst pivot, but an attacker-controlled apiKey on a legitimate provider host still exfiltrates the next createMessage payload to the wrong account. `select` action silently switches the user onto a previously-injected model.
- **Remediation**: Add a carve-out in checkApproval mirroring the update_settings pattern: when group is 'agent' and tool is 'configure_model', call onApprovalRequired. Applies to all three actions (add / select / test). One-line fix keeps the rest of agent-group auto-approval intact.
- **Verifier note**: Confirmed. The tool's header comment states the approval surface should fire; the pipeline does not honor it. Analogous control already exists for update_settings (AUDIT-006 H-3).
- **Status**: Resolved

### M-12: OperationLogger SENSITIVE_KEYS list misses camelCase credential keys

- **ID**: OWASP-A02-2 + OWASP-A09-1 (consolidated)
- **CWE / Category**: CWE-312 / CWE-117 / CWE-532 Insertion of Sensitive Information into Log File
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/governance/OperationLogger.ts:56-85
- **Risk**: sanitizeParams uses exact-set Set.has against {password, token, api_key, secret, key, auth, authorization}. configure_model (api_key) is covered. Missed: apiKey, awsAccessKey, awsSecretKey, awsSessionToken, accessToken, refreshToken, bearerToken, clientSecret, subscriptionKey, gatewayHeaderValue, plus header blocks. ManageMcpServerTool declares an `headers` input schema property explicitly for authentication. When the agent calls manage_mcp_server with `headers: { Authorization: 'Bearer eyJ...' }`, sanitizeParams iterates only top-level keys; `headers` is not in SENSITIVE_KEYS, the value is an object so the >500 chars truncation never triggers, and the bearer token is JSON.stringified verbatim into the daily JSONL audit log. LogTab UI ships an explicit Download button.
- **Remediation**: Replace the Set with a regex predicate matching credential-shaped key segments: `/(password|passphrase|secret|bearer|credential|authoriz|^auth$|api[_-]?key|access[_-]?key|secret[_-]?key|access[_-]?token|refresh[_-]?token|session[_-]?token|gateway[_-]?header[_-]?value)/i`. Redact `headers`, `request_headers`, `http_headers`, `cookies` as whole blocks. Walk nested objects/arrays with depth cap 4 and visited Set. Add value-layer scrubbing of well-known patterns: Bearer prefix, sk- (OpenAI), xox[bpoasr]- (Slack), ghp_/gho_/ghu_/ghs_/ghr_/github_pat_ (GitHub), AKIA/ASIA (AWS). Unit tests for apiKey/accessToken/clientSecret/nested credentials redaction AND non-redaction of cache_key/sort_key/keyword.
- **Verifier note**: Confirmed at OperationLogger.ts:56-85. ManageMcpServerTool headers path is a concrete live exploit vector. Log dir excluded from Obsidian Sync but is exported via Download button.
- **Status**: Resolved

### M-13: SemanticIndexService enrichChunkWithContext sanitizer is far weaker than sanitizeVaultContentForLLM

- **ID**: OWASP-A03-2
- **CWE / Category**: CWE-94 Improper Control of Generation of Code (stored prompt injection)
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/semantic/SemanticIndexService.ts:1261-1303
- **Risk**: Pass-2 contextual-retrieval enrichment sends vault note content to an LLM. The inline sanitize only strips backticks and `^(system|assistant|user):` line prefixes. It misses the entire INJECTION_PATTERNS suite that sanitizeVaultContentForLLM.ts covers (ignore previous instructions, <system>..</system>, you are now, forget all prior, [[system]], etc.). A vault note containing one of these (web-clipped article, third-party import) can hijack the per-chunk generation. The compromised output is (1) STORED as the chunk's enriched text, (2) RE-EMBEDDED biasing the vector, (3) RETURNED later as an `excerpt` from search_vault and semantic_search results that flow into the main agent LLM. Second-order prompt injection with cross-session persistence.
- **Remediation**: Import sanitizeWithDetails from sanitizeVaultContentForLLM in SemanticIndexService.ts:1252-1303, replacing the inline sanitize for fullContent AND per-chunk slices before embedding in the prompt. Log safeDoc.redactedCount via console.debug for audit visibility. Add a unit test mirroring the memory-layer test (an INJECTION_PATTERNS string must come out replaced with `[redacted -- prompt-injection-pattern]`).
- **Verifier note**: Confirmed. Stronger sanitizer ships next door; the fix is mechanical. Contextual retrieval is opt-in, contextual handler is small/cheap, but persistence + symmetry argument keeps it Medium.
- **Status**: Resolved

### M-14: MCP client obsidianFetch issues arbitrary HTTP requests with NO SSRF guard

- **ID**: OWASP-A10-1
- **CWE / Category**: CWE-918 SSRF
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/mcp/obsidianFetch.ts:22-120, src/core/mcp/McpClient.ts:65-92
- **Risk**: External MCP servers are user-configured via McpServerConfig.url and connected via obsidianFetch, which uses raw node:http/node:https. No hostname check, no IP check, no DNS-rebinding check, no protocol restriction beyond http/https. WebFetchTool's two-phase SSRF guard is not applied. providerUrlGuard exists at src/api/providers/providerUrlGuard.ts with BLOCKED_HOSTNAMES (including 169.254.169.254 and metadata.google.internal) and isPrivateIpHostname, but is not wired into the MCP path. The LLM-driven mass-exploit chain via update_settings is closed today (mcpServers.* not in WRITABLE_PATHS), but the user-typed URL via McpTab modal (line 391, zero validation) and settings-file tampering paths remain.
- **Remediation**: In McpClient.connect, before constructing the SDK transport, call `validateProviderUrl('custom', config.url, { allowLocalhost: this.plugin.settings.mcp?.allowLocalUrls === true })`. Default allowLocalUrls=false; surface an explicit opt-in checkbox in McpTab for local dev MCP. In obsidianFetch.ts:22-48 after `new URL`, reject if protocol is not http: or https: and reject if hostname matches BLOCKED_HOSTNAMES or isPrivateIpHostname unless allowLocal is threaded through. Add the same guard to McpTab saveBtn so the modal rejects a hostile URL at type-in.
- **Verifier note**: Confirmed gap. The original LLM-prompt-injection-to-IMDS chain claimed by the finding is partially closed today; remaining vectors are user-typed URL via McpTab and settings tampering. Existing providerUrlGuard module reduces fix cost to near-zero.
- **Status**: Partial (deferred to backlog)

### M-15: Tokens stored in plaintext when OS keychain is unavailable, no UI surfaced warning

- **ID**: OWASP-A07-02
- **CWE / Category**: CWE-312 Cleartext Storage of Sensitive Information
- **Severity**: Medium
- **Confidence**: Confirmed
- **Location**: src/core/security/SafeStorageService.ts:53-55, 67-76, 82-95
- **Risk**: Operational pair of SAST-C-03. ChatGPT OAuth refresh tokens, GitHub PAT, GitHub Copilot bearer, Bedrock awsSecretKey/awsSessionToken/gatewayHeaderValue, and every providerConfigs[].apiKey are persisted as bare strings to data.json when safeStorage.isAvailable is false. Comprehensive grep across src/ui/ returns zero references to safeStorage / keychain / plaintext, so the user receives no UI signal at all. Linux installs without libsecret are the common trigger. data.json sits in the vault and is replicated by Obsidian Sync and external sync providers.
- **Remediation**: Two changes, both UI-focused so the plugin still functions on Linux installs without a keychain. (1) Persistent dismissible-only-per-session banner in ProvidersTab and any tab that renders API key inputs whenever isAvailable returns false. (2) On the first settings save after detecting the unavailable state, surface a one-time confirmation modal: "Continue (store plaintext)", "Use session-only memory storage", "Cancel and install a keychain". Persist the _encrypted=false state visibly in Debug/Status.
- **Verifier note**: Confirmed. Severity Medium is calibrated; OAuth refresh tokens grant long-lived access and the silent fallback violates the documented confidentiality promise.
- **Status**: Partial (deferred to backlog)

### L-1: SandboxBridge validateVaultPath uses substring check that mis-handles edge cases

- **ID**: SAST-1-06
- **CWE / Category**: CWE-22 Path Traversal (code consistency)
- **Severity**: Low (verifier-revised to Info, but kept Low for backlog visibility)
- **Confidence**: Confirmed
- **Location**: src/core/sandbox/SandboxBridge.ts:329-343
- **Risk**: `path.includes('..')` rejects legitimate filenames like `Foo..Bar.md` (false positive). No percent-encoded or NUL-byte coverage. Two divergent implementations of the same guard make audits harder. Sandbox passes paths to vault adapter which does not URL-decode, so no real exploitable bypass; this is hygiene only.
- **Remediation**: Replace the inline check with a call to validateVaultRelativePath. Treat null as rejection. Unifies the rule set across SandboxBridge, MCP tools, and vault tools.
- **Verifier note**: Hygiene/consistency only. No exploitable path-traversal bypass.
- **Status**: Resolved

### L-2: Token-exchange and refresh failures emit raw provider body

- **ID**: SAST-C-05
- **CWE / Category**: CWE-209 Information Disclosure via Logs
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/core/auth/ChatGptOAuthService.ts:253-256, 375-378
- **Risk**: exchangeCodeForTokens and refreshAccessToken format the upstream error body via a naive 500-char truncate and embed it in the thrown Error. OpenAI's auth.openai.com today returns only error/error_description/error_uri, so no secret currently leaks. The truncation is one upstream-schema-change away from embedding a submitted code_verifier or refresh_token in an error body.
- **Remediation**: Apply field-aware extractOAuthError at the two safeJsonString call sites (ChatGptOAuthService.ts:254 and :376). Drop everything that is not a documented OAuth error field. Optionally harden chatgpt-oauth.ts enhanceError (line 449-493). Do NOT touch CloudflareDeployer.ts:224-232; it already does field-aware extraction.
- **Verifier note**: Hardening is defense-in-depth, no current exploit.
- **Status**: Resolved

### L-3: PdfParser iterates all numPages without a per-document cap

- **ID**: SAST-R-03
- **CWE / Category**: CWE-400 Uncontrolled Resource Consumption
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/core/document-parsers/parsers/PdfParser.ts:78-89
- **Risk**: Unbounded loop awaiting getPage + getTextContent per page, accumulating into parts[] with no page-count cap and no joined-output byte cap. A 200 MB attacker PDF can advertise 100k+ pages; loop takes hours. Reachable from chat attachment, read_document, and SemanticIndexService bulk indexing.
- **Remediation**: Cap MAX_PAGES (e.g. 2000) and MAX_TEXT_BYTES (e.g. 50 MB). On exceedance, push a "(truncated)" sentinel and break. Consistent with the existing MAX_DECOMPRESSED_SIZE=500 MB ZIP-bomb guard.
- **Verifier note**: DoS impact is the user's own Obsidian process; not a multi-tenant server. Bounded by AttachmentHandler 200 MB cap which is decoupled from page count.
- **Status**: Resolved

### L-4: McpClient logs raw connection errors that may include URLs with embedded auth tokens

- **ID**: SAST-R-04
- **CWE / Category**: CWE-209 Information Disclosure via Logs
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/core/mcp/McpClient.ts:111-113
- **Risk**: On MCP connection failure, raw error message goes to conn.error (Settings UI) and console.error. RelayClient applies redactToken at this exact sink (AUDIT-005 H-2/H-3); McpClient does not. Verified the MCP SDK transports do not embed `this._url.toString()` in errors today and McpServerConfig uses header-based bearer auth (not URL query), so the realistic residual leak is upstream-server-body reflection. Defense-in-depth gap.
- **Remediation**: Inline a Bearer-prefix and token-query-param redaction at the two sinks. Pass the redacted string only to console.error (not the raw Error object whose stack still hits devtools). Add a unit test against a stub server that echoes the Authorization header in a 401 body.
- **Verifier note**: AUDIT-005 standard is project-internal and violated here. No current Authorization-via-URL pattern in this codebase.
- **Status**: Resolved

### L-5: WebFetchTool isPrivateIP accepts IPv4 with non-numeric octets and skips IPv4-mapped IPv6

- **ID**: SAST-R-08
- **CWE / Category**: CWE-704 Improper Type Conversion
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/core/tools/web/WebFetchTool.ts:20-42
- **Risk**: Two false-alarm sub-claims (127.0.0.1.evil.com and 0.0.0.0) are already handled by phase-1 + phase-2 DNS resolution and `a === 0`. The real gap is IPv4-mapped IPv6 `[::ffff:127.0.0.1]`: split(.) yields 4 parts where the first is NaN, returns false, IPv6 branch never reached, Phase 2 DNS throws and is swallowed, requestUrl dials loopback via IPv6 stack. Compounded by bracket-stripping issue: `parsedUrl.hostname` returns `[::1]` with brackets in some runtimes, so the IPv6 branch never matches `::1` literally.
- **Remediation**: Detect IP literals via net.isIPv4 / net.isIPv6 (already in Electron). Strip URL brackets before the check. Recurse into IPv4-mapped IPv6 inner address. See SAST-1-04 for the consolidated fix; both findings touch the same function.
- **Verifier note**: Genuine but narrow gap. Approval-gated by web auto-approval.
- **Status**: Resolved

### L-6: TemplateCatalog loadUserTheme/saveTheme do not validate themeName for path traversal

- **ID**: SAST-R-09
- **CWE / Category**: CWE-22 Path Traversal
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/core/office/pptx/TemplateCatalog.ts:71-186
- **Risk**: themeName is interpolated directly into `${configDir}/${THEME_BASE_DIR}/${themeName}`. Default-theme allowlist runs only in loadTemplate AFTER loadUserTheme. saveTheme is dead code today (ingest_template removed) but loadUserTheme is reachable via plan_presentation -> loadTemplate(template). Read-only exfiltration channel via formatSlideTypeGuide echo. Project already uses assertSafePathSegment uniformly in InvokeSkillTool, InvokeMcpServerTool, RunSkillScriptTool, pluginApiAdaptive; TemplateCatalog is the outlier.
- **Remediation**: Reuse assertSafePathSegment from src/core/utils/safePathName.ts. Call at the entry of loadUserTheme, saveTheme, and (defensively) loadTemplate.
- **Verifier note**: Consistency gap with project standard. Two-file precondition limits real exploit window.
- **Status**: Resolved

### L-7: Agent-internal search_history tool does not escape LIKE wildcards

- **ID**: OWASP-A03-1
- **CWE / Category**: CWE-89 SQL LIKE wildcard injection
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/core/tools/memory/SearchHistoryTool.ts:78-82
- **Risk**: Agent-internal path builds `text LIKE ?` with `%${query}%` and does NOT escape % or _ in query. The MCP wrapper (src/mcp/tools/searchHistory.ts:54) explicitly escapes these (AUDIT-016 M-2 remediation); the agent-facing path was not patched in parallel. A `%` query becomes a 30-row history-dump primitive.
- **Remediation**: Mirror the MCP fix. Replace lines 78-79 with `query.replace(/[%_\\]/g, '\\$&')` and `LIKE ? ESCAPE '\\'`. Add a regression test. Factor escape into a shared util (src/core/knowledge/sqlLikeEscape.ts) so both surfaces stay aligned.
- **Verifier note**: topK hard-capped at 30, in-process agent caller, single-user trust boundary. Drift with MCP surface is the real issue.
- **Status**: Resolved

### L-8: Filename interpolated into XML attribute without quoting

- **ID**: OWASP-A03-3
- **CWE / Category**: CWE-1287 Improper Validation of Specified Type of Input
- **Severity**: Low
- **Confidence**: Suspected
- **Location**: src/core/semantic/SemanticIndexService.ts:1265, 1275
- **Risk**: `<document title="${title}">` where title is the filename, with no escaping of `"`, `<`, `>`, `&`. A filename like `note" instructions="obey me ".md` produces a malformed XML attribute the LLM may parse as separate attributes carrying instructions. Filenames are vault-controlled; MCP write from untrusted source surface or sync-imported note from public vault. Impact ceiling is polluted embeddings, not unauthorized actions (the sub-LLM call has no tools).
- **Remediation**: Reuse escapeXmlAttr from src/ui/sidebar/AttachmentHandler.ts:29 (extract to shared util or inline copy). Apply at line 1275. Keep length-clamp and existing backtick/role-prefix sanitize.
- **Verifier note**: Inconsistency with AttachmentHandler convention. Single-site fix.
- **Status**: Resolved

### L-9: MCP rate-limit caller key includes client-supplied source_interface

- **ID**: OWASP-A04-2
- **CWE / Category**: CWE-770 Allocation of Resources Without Limits
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/mcp/tools/index.ts:243-247
- **Risk**: callerKey is `${mcpServerToken}:${args.source_interface}`. All external MCP clients share the same server-wide mcpServerToken. There is NO whitelist validation in this code path; validateSourceInterface is not called, so an attacker supplies arbitrary strings and bucket multiplication is unbounded (worse than the original 6x estimate). Defeats the AUDIT-015 M-1 expensive-class protection (10/min on LLM/embedding cost).
- **Remediation**: Drop source_interface from the bucket key. Change callerKey to `mcpServerToken ?? 'unauthenticated'`. Keep args.source_interface only for OperationLogger telemetry. If per-source buckets are desired, first validate via validateSourceInterface to bound cardinality to the 6 whitelist values.
- **Verifier note**: Defense-in-depth control failure, not an authorization break (bearer auth still gates entry).
- **Status**: Resolved

### L-10: OAuth refresh token never revoked server-side on logout

- **ID**: OWASP-A07-01
- **CWE / Category**: CWE-613 Insufficient Session Expiration
- **Severity**: Low (verifier-revised from Medium)
- **Confidence**: Confirmed
- **Location**: src/core/auth/ChatGptOAuthService.ts:389-399, src/core/security/GitHubCopilotAuthService.ts:378-384, src/core/security/KiloAuthService.ts:309-318
- **Risk**: logout() zeros in-memory and persisted token fields but never calls a revocation endpoint. A previously-exfiltrated refresh_token keeps minting access tokens until provider-side rotation catches up. UI label matches actual behavior (clear local credentials); exploitation requires a prior, separate compromise.
- **Remediation**: Best-effort server-side revocation BEFORE clearing local state, only for endpoints with stable public contracts: GitHubCopilotAuthService DELETE https://api.github.com/applications/{client_id}/grant (documented GitHub OAuth Apps API); ChatGptOAuthService best-effort POST to https://auth.openai.com/oauth/revoke (treat 4xx as expected, reverse-engineered endpoint, do not block logout); KiloAuthService defer until a documented endpoint exists. Add a "Sign out everywhere" affordance in ProvidersTab.
- **Verifier note**: Standard practice for OAuth client apps. Linux plaintext-fallback amplification is small.
- **Status**: Partial (deferred to backlog)

### L-11: esbuild-wasm INTEGRITY_HASHES hard-coded with no automated refresh

- **ID**: OWASP-A08-06
- **CWE / Category**: CWE-1395 Dependency on Vulnerable Third-Party Component (operational)
- **Severity**: Low (verifier said Info, kept Low for visibility)
- **Confidence**: Confirmed
- **Location**: src/core/sandbox/EsbuildWasmManager.ts:42-65
- **Risk**: BUG-022 documented an instance where the version was bumped but the hashes were not. The verification still trips on mismatch (failure mode is "sandbox unusable", not "malicious code loaded"). The drift-detection script scripts/update-esbuild-integrity.sh exists but is not wired into CI.
- **Remediation**: Add npm script `"verify:esbuild-hashes": "bash scripts/update-esbuild-integrity.sh"` to package.json. Add a CI step on pull_request paths src/core/sandbox/EsbuildWasmManager.ts that runs `npm run verify:esbuild-hashes`. No code change inside EsbuildWasmManager.ts.
- **Verifier note**: Maintainability/release-quality gap, not exploitable. Trust boundary holds.
- **Status**: Partial (deferred to backlog)

### L-12: Logger swallows write failures silently

- **ID**: OWASP-A09-5
- **CWE / Category**: CWE-779 Logging of Excessive Data (inverse: silent loss)
- **Severity**: Low
- **Confidence**: Confirmed
- **Location**: src/core/governance/OperationLogger.ts:117-120
- **Risk**: log() wraps the write in try/catch and reduces every failure to console.warn. If the vault is read-only, the .vault-operator folder is deleted mid-session, or daily file hits disk-full, the agent keeps executing destructive operations with no audit record and no UI indication. LogTab shows "no entries" indistinguishably from "no operations executed today".
- **Remediation**: Private failedWrites counter and failureNoticeShown flag. On catch, increment counter; at threshold 1-3, fire `new Notice(...)` once per session. Expose getFailedWriteCount so LogTab can render an inline banner. Optionally write a best-effort sentinel `logs/.failures-YYYY-MM-DD`. Keep log() non-throwing.
- **Verifier note**: Governance/observability defect, not exploitable. Vault Operator markets OperationLogger as the auditable record; silent loss breaks that promise.
- **Status**: Partial (deferred to backlog)

### L-13: WebFetchTool DNS guard uses public resolvers, not the OS resolver

- **ID**: OWASP-A10-3
- **CWE / Category**: CWE-918 SSRF
- **Severity**: Low
- **Confidence**: Suspected
- **Location**: src/core/tools/web/WebFetchTool.ts:222-239
- **Risk**: resolveHost hardcodes `['8.8.8.8','1.1.1.1']` while requestUrl uses the OS/Electron resolver. On split-horizon corporate networks, `wiki.internal.example` NXDOMAINs publicly but resolves to a private IP via the company resolver; the empty catch swallows it and requestUrl succeeds against the internal service. Hostname suffix check absent for `.internal`, `.local`, `.corp`, `.lan`, `.intra`.
- **Remediation**: Replace public-resolver pin with `dns.promises.lookup(hostname, { all: true, verbatim: true })`. Add a hostname-suffix denylist before any DNS work. Change the catch at lines 138-142 to fail closed for non-IP-literal hostnames.
- **Verifier note**: Narrow to split-horizon corporate setups. Default-disabled web tools lower baseline.
- **Status**: Resolved

### L-14: WebFetchTool does not constrain Obsidian's requestUrl redirect chain

- **ID**: OWASP-A10-4
- **CWE / Category**: CWE-601 / CWE-918 Open Redirect / SSRF
- **Severity**: Low
- **Confidence**: Hypothesis
- **Location**: src/core/tools/web/WebFetchTool.ts:152-164
- **Risk**: Obsidian's requestUrl follows HTTP redirects internally with no exposed cap. The two-phase SSRF guard runs on the initial URL only. A malicious public endpoint can 302 -> `http://169.254.169.254/latest/...` and requestUrl follows without re-running the guard. AWS IMDSv2 refuses but IMDSv1 deployments and other internal services respond.
- **Remediation**: Replace requestUrl in WebFetchTool.execute with a manual redirect loop using node:https / node:http (port the obsidianFetch pattern). For every hop (cap 3): parse, run Phase 1 + Phase 2 on the new hostname, additionally inspect the connected socket's remoteAddress against isPrivateIP. Integration test using an in-process http server that 302s to private IPs. File an upstream Obsidian issue for `followRedirect: false` as a parallel track.
- **Verifier note**: Real but narrow. IMDSv2 mitigates the canonical target. Residual risk is IMDSv1 plus internal HTTP services responding to plain GET.
- **Status**: Resolved

### L-15: In-plugin tool results not wrapped in a trust-signaling boundary tag

- **ID**: LLM-LLM01-1
- **CWE / Category**: LLM-01 Indirect Prompt Injection
- **Severity**: Medium (verifier-revised; kept Medium per finding consensus)
- **Confidence**: Confirmed
- **Location**: src/core/tools/BaseTool.ts:91-99; ReadFileTool.ts:138-153; SemanticSearchTool.ts:295-310; WebFetchTool.ts:198-205; UseMcpToolTool.ts:97-100
- **Risk**: wrapVaultContentForMcp at McpBridge.ts:866-873 provides a clean `<vault-content path="..." trust="user-data">` boundary primitive. Neither it nor sanitizeVaultContentForLLM is applied when the in-plugin AgentTask reads vault content, fetches a URL, runs hybrid retrieval, or calls an external MCP server. ReadFileTool wraps in neutral `<content>`, SemanticSearchTool concatenates raw excerpts, WebFetchTool uses `<web_fetch>` without trust marker, UseMcpToolTool pushes external responses 1:1. The MCP-side mitigation exists; it just is not reused on the in-plugin side.
- **Remediation**: Extract a shared `wrapUntrustedContent(source, content, metadata)` helper that XML-escapes attribute values AND inner content boundary. Reuse in BaseTool.formatContent (replacing `<content>` with `<untrusted-content trust="user-data" source="...">`), in WebFetchTool's `<web_fetch>` (add trust attribute), in SemanticSearchTool's excerpt block, in UseMcpToolTool around the result string (`<mcp_response server="..." tool="..." trust="user-data">...</mcp_response>`). Fix the unescaped attribute interpolation in BaseTool.ts:94. Extend getSecurityBoundarySection to enumerate the recognized wrappers as USER DATA.
- **Verifier note**: Real, exploitable, partially mitigated by the 4-line prompt boundary. MCP-side mitigation already shows the correct pattern.
- **Status**: Partial (deferred to backlog)

### L-16: getSecurityBoundarySection is a 4-line one-shot lacking tool-result jailbreak language

- **ID**: LLM-LLM01-2
- **CWE / Category**: LLM-01 Indirect Prompt Injection
- **Severity**: Medium (verifier consensus)
- **Confidence**: Confirmed
- **Location**: src/core/prompts/sections/securityBoundary.ts:8-19
- **Risk**: Four-line section names only "vault files or web pages". Omits tool_result blocks, ingested PDFs/DOCX/PPTX/XLSX, semantic_search excerpts, MCP responses, history matches. Omits explicit "ignore previous instructions" / "you are now" jailbreak wording. The robust sanitizer is wired ONLY on the memory-extraction path, NOT into the agent loop.
- **Remediation**: Expand to (1) name every wrapper the model will see (`<untrusted-content>`, `<vault-content>`, `<web_fetch>`, `<attached_document>`, `<mcp_response>`, `<history>`), (2) enumerate the most common injection sentinels and tell the model to treat them as data, (3) explicitly forbid re-targeting tool calls based on instructions inside any of those wrappers, (4) optionally restate the contract at user-message-tail when an external resource was just pulled in. Keep ABOVE the cache breakpoint.
- **Verifier note**: Cheap (prompt-only), stays in cached prefix, aligns with wrappers the code already emits.
- **Status**: Resolved

### L-17: Imported skills are treated as trusted instructions

- **ID**: LLM-LLM01-3
- **CWE / Category**: LLM-01 Prompt Injection via Imported Content
- **Severity**: Low (verifier-revised from Medium per consensus)
- **Confidence**: Confirmed
- **Location**: src/core/tools/agent/InvokeSkillTool.ts:208-224; SkillImportRouter.ts:51-127; SelfAuthoredSkillLoader.ts:298-321
- **Risk**: Skills enter from four paths (bundled, user-imported .md/.zip/folder, agent-written via skill-translator pulling SKILL.md from a GitHub URL via web_fetch, manually-written). All four land in the same SelfAuthoredSkillLoader. composeSubtaskMessage injects skill.body as the first user message of a fresh subtask that inherits the full tool registry or the allowedTools from the SKILL.md frontmatter (which the imported skill itself defines). No provenance check, no execution sandbox between bundled and user skills, no approval-card source badge. Hot-reload watcher auto-loads any SKILL.md dropped into the skills folder.
- **Remediation**: (1) In composeSubtaskMessage, accept the SelfAuthoredSkill. For source !== builtin/bundled, wrap with `<imported-skill source="..." name="...">` and prepend a "cannot override host plugin's tool-approval rules" line. (2) Plumb skill.source into the invoke_skill approval prompt as a Source badge. (3) In checkApproval for group 'skill', when skill.source is not builtin/bundled AND autoApproval.skills is on, fall through to onApprovalRequired for the FIRST invocation; track per-skill approved set in settings. (4) Clamp frontmatter.allowedTools for non-builtin skills: missing -> conservative read-only default; present -> intersect unless user widens. (5) Normalize imported source frontmatter to `user` on write so an imported file cannot claim `builtin`.
- **Verifier note**: Default fail-closed posture (autoApproval.skills=false) and subtask onApprovalRequired inheritance limit live impact. Low calibration matches the audit's default-conservative defaults.
- **Status**: Partial (deferred to backlog)

### Info-1: Fail-closed defaults for MCP server, tunnel, relay, and auto-approval

- **ID**: OWASP-A05-01
- **CWE / Category**: CWE-1188 Insecure Default Initialization
- **Severity**: Info
- **Confidence**: Confirmed
- **Location**: src/types/settings.ts:1664-1740, src/mcp/McpBridge.ts:399
- **Risk**: Recorded for traceability. DEFAULT_SETTINGS ship with enableMcpServer=false, enableRemoteRelay=false, relayUrl='', cloudflareApiToken='', autoApproval.enabled=false. Built-in agent mode does include all tool groups by design, but the approval pipeline gates every write/web/MCP call. Net posture is fail-closed and matches the AUDIT-033 baseline.
- **Remediation**: No change required.
- **Verifier note**: Baseline confirmation. Info severity calibrated correctly.
- **Status**: Confirmed (open) -- baseline marker

### Info-2: Debug mode defaults off; debug-gated testToolExecution bypass is documented

- **ID**: OWASP-A05-03
- **CWE / Category**: CWE-489 Active Debug Code
- **Severity**: Info
- **Confidence**: Confirmed
- **Location**: src/types/settings.ts:1871, src/main.ts:3043, 3052, 3954-3955
- **Risk**: debugMode default false. testToolExecution bypass is explicitly gated on debugMode with both console.warn and a user-visible Notice when blocked. console.debug used per project convention. Risk in shipped builds is low.
- **Remediation**: Optional inline warning under the Debug Mode toggle in Settings noting that enabling it unlocks dev paths that skip approvals.
- **Verifier note**: Hard-coded gating, not a runtime flip. Bypass unreachable without explicit user opt-in.
- **Status**: Confirmed (open) -- baseline marker

### Info-3: Stigmergy daemon is local-socket only and lazily loaded

- **ID**: OWASP-A05-04
- **CWE / Category**: CWE-1357 Reliance on Insufficiently Trustworthy Component
- **Severity**: Info
- **Confidence**: Confirmed
- **Location**: src/core/stigmergy/StigmergyAdapter.ts:218-275
- **Risk**: @agentic-stigmergy/* is lazy-imported, talks only to a unix socket under ~/.stigmergy or STIGMERGY_SOCK, any failure leaves the adapter in no-op mode. Zero network APIs. Trust-blast-radius is limited to local recall ranking. VO selectors hold hard precedence over Stigmergy suggestions.
- **Remediation**: No change required. Re-audit if the package gains a network mode.
- **Verifier note**: Matches arc42 §8.16, ADR-130/131/132/133, AUDIT-035, AUDIT-036.
- **Status**: Confirmed (open) -- baseline marker

### Info-4: HistoryDB lacks daily snapshots; inherits per-write atomic guard

- **ID**: OWASP-A04-4
- **CWE / Category**: CWE-1188 Insecure Default Initialization (durability)
- **Severity**: Info (verifier-revised from Low)
- **Confidence**: Hypothesis
- **Location**: src/core/knowledge/HistoryDB.ts:52-93; src/main.ts:1869-1894
- **Risk**: SnapshotJob targets enumerate only knowledgeDB and memoryDB. historyDB is opened earlier but never appended. The per-write atomic .bak rotation inherited from KnowledgeDB applies, so single-write corruption is still recoverable; the gap is only the 7-day rolling window. Data-durability/availability gap, not a security flaw.
- **Remediation**: In main.ts after the memoryDB push, add: `if (this.historyDB && this.historyDB.getStorageLocation() !== 'obsidian-sync') { targets.push({ name: 'history', sourcePath: this.historyDB.getAbsolutePath() }); }`. No SnapshotJob changes needed. Track as a backlog item under EPIC-019 / FEATURE-0314 follow-up.
- **Verifier note**: Durability fix, not security. CWE-1188 is a poor fit. Info severity appropriate.
- **Status**: Resolved

### Info-5: MD5 used for vault-path identity hash (hygiene)

- **ID**: SAST-C-06
- **CWE / Category**: CWE-327 Use of Broken Cryptographic Algorithm
- **Severity**: Info
- **Confidence**: Confirmed
- **Location**: src/main.ts:798-803, src/ui/settings/VaultTab.ts:988-995
- **Risk**: createHash('md5') derives a 12-char vault-id sub-folder under ~/.vault-operator-migration-backups/. Comment states "not a security-sensitive hash" and the use case (collision-resistant tag for a path bucket) confirms. Actual security impact is zero, but CodeQL and SonarQube re-flag MD5 every audit, so closing it once is cheaper than re-justifying.
- **Remediation**: Replace createHash('md5') with createHash('sha256') in both sites, keeping .slice(0, 12). For backward compatibility, restore path probes both sha256 and md5 directories preferring sha256; write path always uses sha256.
- **Verifier note**: Pure hygiene; zero security impact. SonarQube/CodeQL re-raise without it.
- **Status**: Resolved

### Info-6: deepMergeSettings recurses on __proto__ when key set via JSON.parse

- **ID**: SAST-D-01
- **CWE / Category**: CWE-1321 Prototype Pollution
- **Severity**: Info (verifier-revised from Low)
- **Confidence**: Confirmed
- **Location**: src/main.ts:4044-4066
- **Risk**: JSON.parse stores __proto__ as own enumerable property, so Object.entries surfaces it. The recursive branch enters when defaults.__proto__ is an object (it is, Object.prototype passes the guards). The assignment mutates the local merged object's prototype chain, NOT global Object.prototype. saveData strips __proto__ via JSON.stringify, so any pollution does not survive a round-trip. saved is loaded from data.json on the user's disk; any "attacker" already has FS write access.
- **Remediation**: Skip `__proto__`, `constructor`, `prototype` keys before iterating. Defense-in-depth.
- **Verifier note**: Structural CWE-1321 match but no exploitable issue today. Hardening for future-proofing.
- **Status**: Resolved

### Info-7: @huggingface/transformers 4.2.0 transitive low-severity advisories

- **ID**: OWASP-A06-05
- **CWE / Category**: CWE-1395 Dependency on Vulnerable Component (low-severity transitive)
- **Severity**: Info
- **Confidence**: Suspected
- **Location**: package.json:62 ("@huggingface/transformers": "^4.0.0")
- **Risk**: npm audit lists low-severity transitive advisories through onnxruntime-node (global-agent) and onnxruntime-web (guid-typescript, platform); fixAvailable false. Usage confined to RerankerService with hardcoded MODEL_ID (Xenova/ms-marco-MiniLM-L-6-v2). Runtime WASM blob has SHA-256 integrity gate (RERANKER_WASM_SHA256). CDN auto-downloads disabled.
- **Remediation**: No code change required. Track in SCA dashboard. If MODEL_ID becomes configurable, add an analogous integrity check on the model file.
- **Verifier note**: Real advisories, no exploitation path in current usage.
- **Status**: Confirmed (open)

## Positive findings

- writeBinaryToVault already implements the canonical `..`/absolute-path guard (src/core/tools/vault/writeBinaryToVault.ts:34-39). ADR-031 is honored on the binary path.
- validateVaultRelativePath is a clean shared helper (src/core/tools/vault/pathValidation.ts:22-29) used consistently by IngestTriageTool, MarkNoteAsMemorySource, UnmarkNoteAsMemorySource. The fix for H-1 / M-1 / M-2 is to extend the convention, not invent.
- MCP wire path keeps the validateMcpVaultPath guard at src/mcp/tools/readNotes.ts:24, so external MCP clients cannot exploit ReadFileTool's adapter fallback.
- WebFetchTool's two-phase SSRF guard (literal-IP check then DNS-resolve check) is the right shape; the gaps in M-3 / L-5 are at the edges of an otherwise solid design (src/core/tools/web/WebFetchTool.ts:108-142).
- KnowledgeDB atomic write + integrity_check + auto-recovery + 7-day daily snapshots (FEATURE-0314 / ADR-079) verified intact. HistoryDB inherits the per-write atomic guard correctly via the global storage adapter.
- providerCredentialCrypto.ts walker is well-tested and idempotent. Tests round-trip providerConfigs[].credentials.apiKey, awsApiKey, awsAccessKey, awsSecretKey, awsSessionToken, gatewayHeaderValue, oauthToken correctly. The fix for H-3 is just to call it from the second site.
- McpBridge.writeMcpTokenFile already uses { mode: 0o600 } (line 428) and migrateAgentLayout.ts:185-194 already chmods backups with the right justification comment. The pattern exists; M-6 is mechanical replication.
- providerUrlGuard.ts BLOCKED_HOSTNAMES list (169.254.169.254, metadata.google.internal) plus isPrivateIpHostname is the reusable centerpiece for closing M-14.
- Sandbox process isolation via ProcessSandboxExecutor + ELECTRON_RUN_AS_NODE=1 contains the worker-realm prototype-pollution surface inside a dedicated OS process. ADR-021 trade-off remains sound.
- Stigmergy adapter precedence resolver continues to keep VO selectors authoritative; AUDIT-035 L-1 fix for guidance-text sanitization remains in place (StigmergyAdapter.ts:532-552).
- Reasoning-effort steering across providers does not introduce new credential surfaces; thinkingBudget interacts correctly with resolveOutputBudget across Anthropic, Bedrock, OpenAI-compatible, Codex OAuth.
- esbuild-wasm runtime integrity verification trips on hash mismatch (failure mode "sandbox unusable" not "malicious code loaded"); the trust boundary holds even when BUG-022-class drift recurs.

## Delta from prior audits

| Prior finding | Source | Status |
| --- | --- | --- |
| AUDIT-003 H-1 -- prompt injection by design at permissive | AUDIT-003 | Carryover-open (by design) |
| AUDIT-005 H-2/H-3 -- token redaction at log sinks | AUDIT-005 | Confirmed-resolved on RelayClient; Regressed at McpClient (L-4) |
| AUDIT-006 H-1 -- MCP token file 0o600 | AUDIT-006 | Confirmed-resolved (McpBridge.ts:428) |
| AUDIT-006 H-3 -- update_settings autoApproval carve-out | AUDIT-006 | Confirmed-resolved (the pattern; configure_model is analogous miss, M-11) |
| AUDIT-007 H-1 -- provider credential encryption in data.json | AUDIT-007 | Confirmed-resolved on data.json; Regressed at GlobalSettingsService (H-3 / H-4) |
| AUDIT-007 M-4 -- esbuild package-name regex validation | AUDIT-007 | Confirmed-resolved (EsbuildWasmManager.ts:504) |
| AUDIT-015 M-1 -- MCP rate-limit "expensive" class | AUDIT-015 | Confirmed-resolved; defeated by L-9 source_interface bypass |
| AUDIT-015 M-3 -- strictSourceIsolation on MCP path | AUDIT-015 | Confirmed-resolved |
| AUDIT-016 M-2 -- LIKE-wildcard escape in search_history | AUDIT-016 | Confirmed-resolved on MCP path; Drift on agent path (L-7) |
| AUDIT-024 (full, LOW verdict) | AUDIT-024 | Carryover-clean |
| AUDIT-027 H-1 -- providerCredentialsInPlace walker | AUDIT-027 | Confirmed-resolved on data.json path; missed at GlobalSettingsService (H-3) |
| AUDIT-033 v2.12.6 (targeted, Green) | AUDIT-033 | Carryover-clean (delta accumulated since) |
| AUDIT-035 L-1 -- Stigmergy guidance-text sanitization | AUDIT-035 | Confirmed-resolved (StigmergyAdapter.ts:532-552) |
| AUDIT-035 I-1 -- STIGMERGY_SOCK accepted-as-documented | AUDIT-035 | Carryover (accepted) |
| AUDIT-036 -- EPIC-32 trust-boundary integrity | AUDIT-036 | Confirmed-resolved |
| AUDIT-037 H-3 -- MCP searchHistory / recallMemory trust-tag drift | AUDIT-037 | Confirmed-resolved on MCP wrappers |
| FIX-12 (KnowledgeDB atomic write + snapshots) | FEATURE-0314 / ADR-079 | Confirmed-resolved (verified intact) |
| BUG-022 (esbuild integrity hash drift) | BUG-022 | Carryover-open (operational, L-11) |

## Prioritized remediation plan

P1 (Critical + High):

| # | id | title | severity | effort | path |
| --- | --- | --- | --- | --- | --- |
| 1 | H-2 + H-3 + H-4 | GlobalSettingsService encrypt parity with main.ts | High | M | src/core/storage/GlobalSettingsService.ts:155-197 |
| 2 | H-1 | EditFileTool path-traversal guard | High | S | src/core/tools/vault/EditFileTool.ts:86-101 |
| 3 | H-5 | zod override or openai 5->6 bump | High | S (override) / L (bump) | package.json:74 + overrides |

P2 (Medium):

| # | id | title | severity | effort | path |
| --- | --- | --- | --- | --- | --- |
| 4 | M-1 | WriteFileTool path-traversal guard | Medium | S | src/core/tools/vault/WriteFileTool.ts:141-210 |
| 5 | M-2 | ReadFileTool path-traversal guard | Medium | S | src/core/tools/vault/ReadFileTool.ts:96-122 |
| 6 | M-3 + L-5 | WebFetchTool isPrivateIP hardening (CGNAT, IPv4-mapped IPv6, IPv6 link-local, brackets) | Medium | S | src/core/tools/web/WebFetchTool.ts:20-42 |
| 7 | M-5 + M-15 | SafeStorage plaintext-fallback UI Notice + banner | Medium | M | src/core/security/SafeStorageService.ts + ProvidersTab |
| 8 | M-6 | GlobalFileService 0o600 clamp | Medium | S | src/core/storage/GlobalFileService.ts:117-122 |
| 9 | M-7 | OOXML pre-decompression size cap | Medium | S | src/core/document-parsers/parsers/ooxmlHelpers.ts:42-46 |
| 10 | M-8 | XlsxParser column AND row clamp | Medium | S | src/core/document-parsers/parsers/XlsxParser.ts:88-133 |
| 11 | M-9 | ToolExecutionPipeline mode-gate enforcement | Medium | M | src/core/tool-execution/ToolExecutionPipeline.ts:357-377 |
| 12 | M-10 | IgnoreService validation per-tool key list | Medium | M | src/core/tool-execution/ToolExecutionPipeline.ts:635-651 |
| 13 | M-11 | configure_model approval carve-out | Medium | S | src/core/tool-execution/ToolExecutionPipeline.ts:664-682 |
| 14 | M-12 | OperationLogger sanitizer regex + nested walk + value scrubbing | Medium | S | src/core/governance/OperationLogger.ts:56-85 |
| 15 | M-13 | SemanticIndexService enrichment sanitizer | Medium | S | src/core/semantic/SemanticIndexService.ts:1261-1303 |
| 16 | M-14 | obsidianFetch SSRF guard via providerUrlGuard | Medium | M | src/core/mcp/obsidianFetch.ts + McpClient.ts |
| 17 | M-4 | ToolStepsHtml rehydration sanitize via DOMPurify | Medium | M | src/ui/AgentSidebarView.ts:3355-3373 |
| 18 | L-15 + L-16 | Trust-wrapping helper + extended securityBoundary section | Medium | S | BaseTool.ts + securityBoundary.ts |

P3 (Low + Info):

| # | id | title | severity | effort | path |
| --- | --- | --- | --- | --- | --- |
| 19 | L-1 | SandboxBridge validateVaultPath unify with helper | Low | S | src/core/sandbox/SandboxBridge.ts:329-343 |
| 20 | L-2 | ChatGptOAuthService field-aware error extraction | Low | S | src/core/auth/ChatGptOAuthService.ts:253-256, 375-378 |
| 21 | L-3 | PdfParser MAX_PAGES + MAX_TEXT_BYTES cap | Low | S | src/core/document-parsers/parsers/PdfParser.ts:78-89 |
| 22 | L-4 | McpClient error redaction | Low | S | src/core/mcp/McpClient.ts:111-113 |
| 23 | L-6 | TemplateCatalog assertSafePathSegment | Low | S | src/core/office/pptx/TemplateCatalog.ts:71-186 |
| 24 | L-7 | SearchHistoryTool LIKE-wildcard escape | Low | S | src/core/tools/memory/SearchHistoryTool.ts:78-82 |
| 25 | L-8 | escapeXmlAttr for document title interpolation | Low | S | src/core/semantic/SemanticIndexService.ts:1265, 1275 |
| 26 | L-9 | MCP rate-limit caller key cleanup | Low | S | src/mcp/tools/index.ts:243-247 |
| 27 | L-10 | OAuth revocation on logout (GitHub stable, others best-effort) | Low | M | ChatGptOAuthService + GitHubCopilotAuthService + KiloAuthService |
| 28 | L-11 | esbuild integrity-hash CI wiring | Low | S | package.json + .github/workflows |
| 29 | L-12 | OperationLogger failure-notice + LogTab banner | Low | S | src/core/governance/OperationLogger.ts:117-120 |
| 30 | L-13 | WebFetchTool OS-resolver alignment + suffix denylist | Low | S | src/core/tools/web/WebFetchTool.ts:222-239 |
| 31 | L-14 | WebFetchTool manual redirect-chain SSRF re-check | Low | M | src/core/tools/web/WebFetchTool.ts:152-164 |
| 32 | L-17 | Imported-skill provenance + approval gating + allowedTools clamp | Low | M | InvokeSkillTool + SkillImportRouter + ToolExecutionPipeline |
| 33 | Info-1..3 | Baseline markers, no change required | Info | -- | -- |
| 34 | Info-4 | HistoryDB snapshot target wiring | Info | S | src/main.ts:1869-1894 |
| 35 | Info-5 | MD5 -> sha256 in vault-path bucket id | Info | S | src/main.ts:798-803, src/ui/settings/VaultTab.ts:988-995 |
| 36 | Info-6 | deepMergeSettings __proto__ skip | Info | S | src/main.ts:4044-4066 |
| 37 | Info-7 | @huggingface/transformers SCA tracking | Info | -- | -- |

## Fix-loop result (2026-06-22)

- Date: 2026-06-22
- Resolved: 32
- Partial (deferred to backlog): 8
- Deferred (to backlog): 0
- Blocked: 0
- Build verification: pass (3 TS test-file regressions repaired: `NonNullable<...>` for optional `legacy_active_models_backup`; `unknown[][]` cast for `vi.fn().mock.calls`)
- Test verification: 3162 of 3163 passing (1 expected fail, by design). One behavior fix in `src/core/mcp/McpClient.ts` `redactMcpError` so the userinfo redaction emits a visible `<redacted>` marker instead of silently dropping the userinfo; no test was relaxed.

Findings that did not resolve cleanly:

| ID | Status | Reason |
| --- | --- | --- |
| M-5 | Partial | Persistent banner + one-time Notice shipped via SafeStorageService + ProvidersTab. Plugin onload toast wiring and formal `safeStoragePlaintextFallbackAcknowledged` field on `ObsidianAgentSettings` live in `src/main.ts` and `src/types/settings.ts`, outside the target file allowlist. |
| M-14 | Partial | obsidianFetch + McpClient SSRF guard, default deny, opt-in via `McpClientOptions.allowLocalUrls`. McpTab modal save-time validation and wiring of `plugin.settings.mcp.allowLocalUrls` into the McpClient constructor are out of scope; settings type `McpServerConfig` unchanged. |
| M-15 | Partial | Operational pair of M-5. Same banner + Notice now covers OAuth refresh tokens, GitHub PAT, Bedrock keys, MCP relay/server tokens. The audit's first-save three-option modal is deferred (requires editing encryptSettingsForSave in `src/main.ts`). |
| L-10 | Partial | ChatGPT best-effort revoke and Copilot best-effort grant revoke shipped before clearing local state. Kilo branch explicitly deferred per audit guidance (no documented endpoint). No fetch-mock harness present for unit-test coverage. |
| L-11 | Partial | `verify:esbuild-hashes` npm script and helper wired. CI step on `pull_request` lives in `.github/workflows`, outside the target file allowlist. |
| L-12 | Partial | OperationLogger now exposes failed-write counter, last-failure message, one-time Notice, sentinel file, and clear-failure accessors. LogTab inline banner consumption lives in `src/ui/settings/LogTab.ts`, outside the target file allowlist. |
| L-15 | Partial | `formatUntrustedContent` + exported `escapeXmlAttribute` helper added in `BaseTool.ts` and attribute interpolation fixed in `formatContent`. Adoption in ReadFileTool, SemanticSearchTool, WebFetchTool, UseMcpToolTool was explicitly deferred per scope rules; helper is in place for a one-line swap. |
| L-17 | Partial | Provenance gate, per-session approval, body wrapper, allowedTools intersection, and conservative read-only default shipped inside `InvokeSkillTool.ts` with 14 new tests. Cross-module pieces (ToolExecutionPipeline `skill`-group gating, settings approval-card Source badge, SelfAuthoredSkillLoader source-normalisation) live outside the target file. |

## Appendix: Dropped candidates

| Dropped id | Title | Reason |
| --- | --- | --- |
| SAST-1-05 | SandboxBridge URL allowlist permits subdomain takeover variants of allowed CDNs | Two of three claims factually wrong (WHATWG URL hostname is lowercased; port pin already exists). Residual subdomain-takeover risk on heavily managed CDNs is theoretical and would break esm.sh by design. Layered defenses (rate limit, circuit breaker, prototype-pollution screening, opt-in gate) cover the residual. |
| SAST-1-07 | LibreOffice customPath bypasses binary-content allowlist via basename-only match | Exploitation precondition does not exist in v2.14.15: customLibreOfficePath has zero callers and no Settings UI surface. Basename-only check is latent hygiene, not exploitable today. |
| SAST-1-08 | EsbuildWasmManager evaluates CDN code via indirect Function constructor with TOFU integrity | Finding itself states "No code change required". TOFU model is documented and accepted; esbuild-wasm core has hard-pinned hashes, only transitive resolveInternalImports uses TOFU with package-name allowlist + npm-registry version pin + hash-mismatch hard error. |
| SAST-1-09 | writeBinaryToVault uses substring check on '..' that over-rejects | Finding self-classifies as "Not a vulnerability". Check strictly blocks MORE than the helper; downstream uses Vault.createBinary/modifyBinary which scope to vault root. Code-health refactor, not a security fix. |
| SAST-1-02 (EditorDemo XSS) | v-html with templated icon/input output in EditorDemo.vue | Verified XSS-safe: ICONS is compile-time-static, renderInput HTML-escapes before regex, inputPlain driven by scripted scenes only. Marketing landing demo in docs site, not plugin runtime. |
| SAST-1-03 (MarkdownRenderer trust) | MarkdownRenderer.render trusts LLM markdown end-to-end | No exploit demonstrated. Obsidian MarkdownRenderer is the community-plugin sanitization contract (strips script/on*, intercepts javascript: URIs, sandboxes iframe srcdoc). Defense-in-depth note only. |
| SAST-1-04 (cookies) | Sensitive Cookies scope check | Project-wide grep returns zero cookie surface. Not applicable. |
| SAST-C-07 | JWT claims parsed from access_token without signature verification | Code matches but trust boundary is intact (TLS-only token sources, narrow consumer, SafeStorage at rest). Forward-looking Info, no actionable defect. |
| SAST-D-02 | ModesTab import casts user-supplied JSON to ModeConfig without key allowlist | JSON.parse does not trigger Object.prototype.__proto__ setter (defineProperty semantics). Downstream sinks use spread/typed reads, not Object.assign. No real prototype-pollution defect; AUDIT-033 M-1 size/structure validation already covers bulk concerns. |
| SAST-D-03 | ConversationStore.updateMeta Object.assign on cached meta with caller patch | All six callers use object literals with hard-coded identifier keys; no untrusted-payload forwarding. Speculative future risk. |
| SAST-D-04 | Sandbox worker exposes parent-realm Object/Array constructors inside vm context | Worker runs as separate OS process via child_process.fork; prototype pollution stays inside the sandbox process. Bridge proxies frozen, IPC via process.send. ADR-021 accepted trade-off. |
| SAST-R-05 | HistoryIndexer.backfillAll continues writes after plugin unload | historyDB.close is NEVER called in onunload (grep confirmed). No race exists. Only consequence is wasted CPU after disable. |
| SAST-R-06 | HistoryIndexer.writeChunks uses array index as primary key | Trust-boundary checks fail: local sql.js file, append-only chat UI (no edit affordance), saveCurrentConversation passes fresh snapshot with same indices, same-user-same-machine search consumer. Real correctness bug (MCP append path under-indexes) but not security. Severity revised to Info, captured as backlog FIX. |
| SAST-R-07 | GitCheckpointService.restore reads vault then writes shadow without diff check | Three mitigations already in place (pre-restore snapshot in RestoreCheckpointTool, restoreCheckpointsForward, DiffReviewModal). User-initiated explicit-button-click destructive intent. UX edge case, not CWE-362 in any security sense. |
| SAST-R-10 | SafeStorageService logs plaintext fallback warnings at module load | Generic state-disclosure, no key/token/PII leaked. Console-only. Finding itself recommends only an optional UX Notice. |
| OWASP-A01-4 | Cloudflare relay /{token}/mcp serves CORS * | Wildcard CORS does not amplify token leak: relay's only secret is the token in the URL, no cookie/origin-bound auth, attacker with token uses curl, not browser. CWE-942 does not apply. Documented intentional tradeoff for browser-based MCP clients. |
| OWASP-A01-5 | Plugin API BLOCKED_METHODS does not include prototype-pollution sinks | Default-deny allowlist (Tier-1 curated + Tier-2 dynamic discovery with own-properties-of-prototype scope + VaultDNAScanner constructor block) makes prototype-pollution names unreachable through three existing controls. |
| OWASP-A02-3 | Local MCP token written to ~/.obsidian-agent/mcp-token in plaintext | Accepted-by-design SOTA (parallels gh CLI, ssh, docker tokens), 127.0.0.1-only HTTP server, 0o600 mode already in place, local-account attacker has easier paths. |
| OWASP-A02-4 | KnowledgeDB / HistoryDB store transcripts at rest without encryption | Location claim stale (history.db moved to vault-parent in FEATURE-1508), consent toggle exists (enableChatHistory), severity Low calibrated, finding labels itself "decision-required, not patch-immediately" -- ADR task, not code patch. |
| OWASP-A04-1 | EPIC-32 Stigmergy integration lacks a documented threat model | All three sub-claims already dispositioned in AUDIT-035 + AUDIT-036 (both Green). guidance-text sanitization shipped; STIGMERGY_SOCK accepted as documented config; arc42 §8.16 + ADR-130/131/132/133 cover the trust model. Re-flagging at Medium is overcalibration. |
| OWASP-A04-3 | Retrieval Wave 1 + Memory v2 Phase 6 lack per-feature threat-model documents | REVIEWER_NOTES.md (375 lines) contains the threat model section. AUDIT-035/036/037 cover the deltas with explicit trust-boundary analyses. The drift example actually demonstrates the existing artifacts work. |