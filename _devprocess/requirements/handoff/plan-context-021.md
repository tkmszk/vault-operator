# Plan Context: ChatGPT OAuth Provider (EPIC-021)

> **Zweck:** Technische Zusammenfassung fuer Claude Code
> **Erstellt durch:** Architect-Skill
> **Datum:** 2026-04-28

---

## Technical Stack

**Backend (Plugin):**

- Sprache: TypeScript (strict)
- Framework: Obsidian Plugin API
- Build: esbuild
- Runtime: Electron (via Obsidian)

**AI-Endpoints:**

- Auth: `https://auth.openai.com/oauth/authorize` und `/oauth/token` (PKCE-Flow)
- API: `https://chatgpt.com/backend-api/codex/responses` (Responses-API-Schema)
- Kein OpenAI-SDK (SDK adressiert `api.openai.com`, hier nicht passend)

**Security:**

- Token-Verschluesselung: SafeStorageService (bestehend, OS-Keychain)
- PKCE: `crypto.randomBytes` (Node) plus Web-Crypto SHA-256 fuer Code-Challenge

**HTTP:**

- OAuth-Token-Endpoints: `requestUrl` (URL-encoded POST)
- Codex-API-Streaming: Node-`https`-Modul mit `IncomingMessage`-Stream (Pattern aus `src/api/providers/openai.ts:75`)
- Loopback-Callback: Node-`http`-Modul, gebunden an 127.0.0.1, Port-Range 1455 bis 1460

## Architecture Style

- Pattern: Adapter (ADR-011), erweitert um vierten Provider neben Anthropic, OpenAI BYOK, Copilot.
- Quality Goals:
  1. Correctness: Schema-Mapping zwischen Codex-Responses und internem `ApiStream`.
  2. Security: Verschluesselte Token-Speicherung, kein Klartext-Logging, Loopback-Bind ausschliesslich auf 127.0.0.1.
  3. Reliability: Auto-Refresh 60 Sekunden vor Ablauf, Promise-Lock gegen parallele Refreshs, Schema-Drift-Erkennung.

## Key Architecture Decisions

| ADR | Titel | Vorgeschlagene Entscheidung | Impact |
|-----|-------|------------------------------|--------|
| ADR-088 | ChatGPT OAuth Provider Architecture | Eigener Provider plus `ChatGptOAuthService`-Singleton, `CodexResponseMapper`, Settings verschachtelt, JWT-Mini-Decoder, Type-Guards statt zod, Hardcode-Modell-Liste, Node-`https`-Streaming | High |
| ADR-089 | PKCE Loopback OAuth Flow | `PkceLoopbackServer` im Renderer mit `require('http')`, Bind 127.0.0.1, Port-Range 1455 bis 1460, 5-Minuten-Timeout, AbortController | High |

**Detail pro ADR:**

1. **ADR-088 Provider-Architektur:** Drei neue Dateien (`src/core/auth/ChatGptOAuthService.ts`, `src/api/providers/chatgpt-oauth.ts`, `src/api/providers/chatgpt-codex-mapper.ts`) plus Mini-Decoder `src/core/auth/jwt-decode.ts`. Schema-Annahmen pro Funktion mit Datums-Kommentar, Drift-Resilienz auf Mapper-Datei begrenzt.

2. **ADR-089 PKCE-Loopback:** Server lebt nur fuer die Dauer des Flows. State-Param schuetzt vor CSRF, Port-Range gegen Belegung. Eslint-disable mit klarer Begruendung.

## Data Model

**Settings-Erweiterung (`src/types/settings.ts`):**

Reale `ProviderType`-Union (verifiziert 2026-04-28 in `src/types/settings.ts:10`):

```typescript
type ProviderType = 'anthropic' | 'openai' | 'gemini' | 'ollama' | 'lmstudio' | 'openrouter' | 'azure' | 'custom' | 'github-copilot' | 'kilo-gateway' | 'bedrock' | 'chatgpt-oauth';
```

Neue Werte: `'chatgpt-oauth'`. Hinweis: `'lmstudio'` ohne Bindestrich, `'bedrock'` existiert bereits.

Codebase-Konvention ist flach (siehe `githubCopilot*` und `kilo*` in `src/types/settings.ts:734-754`). Wir folgen demselben Stil:

```typescript
// ChatGPT OAuth (EPIC-021)
chatgptOAuthAccessToken: string;        // safeStorage.encrypt()
chatgptOAuthRefreshToken: string;       // safeStorage.encrypt()
chatgptOAuthIdToken: string;            // safeStorage.encrypt()
chatgptOAuthAccountId: string;          // public, fuer chatgpt-account-id-Header
chatgptOAuthEmail: string;              // public, UI
chatgptOAuthPlanTier: 'plus' | 'pro' | 'unknown' | '';
chatgptOAuthExpiresAt: number;          // Unix-ms
chatgptOAuthModel: string;              // default 'gpt-5-codex'
chatgptOAuthDisclaimerAcknowledgedAt: number; // 0 wenn nicht bestaetigt
```

**SafeStorageService (real, verifiziert 2026-04-28 in `src/core/security/SafeStorageService.ts:66-95`):**

Reine String-API ohne Envelope-Objekt:

- `encrypt(plain: string): string` -> liefert `enc:v1:<base64>` oder Plaintext-Fallback (wenn `isAvailable()` false ist).
- `decrypt(value: string): string` -> erkennt `enc:v1:`-Prefix, ansonsten Passthrough.
- `isEncrypted(value: string): boolean`.
- `isAvailable(): boolean`.

## External Integrations

| Integration | Endpoint | Methode | Auth | Streaming |
|-------------|----------|---------|------|-----------|
| OAuth Authorize | `auth.openai.com/oauth/authorize` | Browser-Redirect | PKCE | nein |
| OAuth Token | `auth.openai.com/oauth/token` | POST (form-encoded) | PKCE-Verifier | nein |
| Codex Responses | `chatgpt.com/backend-api/codex/responses` | POST (JSON) | Bearer + chatgpt-account-id | ja (SSE) |
| Codex Models (optional) | `chatgpt.com/backend-api/codex/models` | GET | Bearer | nein |

**Required Headers fuer Codex-Calls:**

```
Authorization: Bearer <access_token>
chatgpt-account-id: <accountId aus id_token>
OpenAI-Beta: responses=experimental
User-Agent: Obsilo/<plugin-version>
Content-Type: application/json
```

## Performance & Security

### Performance-Targets

| Metrik | Wert | Quelle |
|--------|------|--------|
| Token-Refresh-Latenz | <500 ms | FEATURE-021-001 |
| Loopback-Server-Startup | <200 ms | FEATURE-021-001 |
| Time-to-First-Token | <2 s p95 | FEATURE-021-002 |
| Streaming-Chunk-Verarbeitung | <50 ms | FEATURE-021-002 |
| Modal-Render | <300 ms | FEATURE-021-003 |
| Memory-Footprint Streaming-Buffer | <16 KB pro Tool-Call-Akkumulator | FEATURE-021-002 |

### Security-Targets

| Aspekt | Mechanismus |
|--------|-------------|
| Token-Storage | SafeStorageService, OS-Keychain |
| PKCE-Verifier | 64 Byte aus `crypto.randomBytes`, SHA-256-Challenge |
| State-Param | 32 Byte Zufall, vor Code-Tausch verifiziert |
| Loopback-Bind | 127.0.0.1, Port-Range 1455 bis 1460 |
| Token-Scope | `openid profile email offline_access` |
| Logging-Diet | Tokens nie in `console.*`, nicht in Audit-Logs, nicht in Telemetrie |
| Per-Request-Auth | Token pro Request vom Service, kein Caching im Handler |
| Disclaimer-Pflicht | Erstmaliger Login mit Bestaetigung, persistiert in `disclaimerAcknowledgedAt` |

## Implementation Sequence

1. **FEATURE-021-001 (OAuth Lifecycle):**
   - `src/core/auth/ChatGptOAuthService.ts` (Singleton, Refresh-Lock, Generation-Counter)
   - `src/core/auth/PkceLoopbackServer.ts` (siehe ADR-089)
   - `src/core/auth/jwt-decode.ts` (id_token Claim-Decoder)
   - Settings-Erweiterung (`src/types/settings.ts`)
   - Unit-Tests fuer PKCE-Generation, State-Verifikation, Refresh-Lock

2. **FEATURE-021-002 (Codex API Handler):**
   - `src/api/providers/chatgpt-oauth.ts` (`ApiHandler`-Implementierung)
   - `src/api/providers/chatgpt-codex-mapper.ts` (Request- und Response-Mapping, Type-Guards, Modell-Liste)
   - `src/api/index.ts` Switch-Erweiterung
   - Unit-Tests fuer Schema-Mapping mit synthetischen Codex-Streams
   - Manueller E2E-Test mit echtem ChatGPT-Plus-Account

3. **FEATURE-021-003 (Settings UI):**
   - `src/ui/settings/ChatGptOAuthBlock.ts` (Provider-Block-Renderer)
   - `src/ui/settings/ModelConfigModal.ts` Erweiterung
   - `src/ui/settings/constants.ts` (PROVIDER_LABELS, PROVIDER_COLORS)
   - i18n-Strings DE und EN
   - Confirm-Modal fuer Disconnect (bestehende Convention `feedback_delete_confirmation`)

## Risks (during coding)

| Risiko | Wahrscheinlichkeit | Mitigation |
|--------|--------------------|------------|
| Codex-Schema laesst sich nicht 1:1 auf `ApiStream` mappen | Mittel | Mapper isoliert, klarer Fehler bei unbekanntem Event-Typ, Datums-Kommentar im Code |
| Plugin-Review-Bot beanstandet `require('http')` | Mittel | Praezedenzfaelle (`require('https')` in `openai.ts`, `require('electron')` in SafeStorageService), klare eslint-disable-Begruendung, PR-Hinweis vorbereiten |
| Port 1455 belegt | Niedrig | Port-Range 1455 bis 1460, klare Fehlermeldung |
| `auth.openai.com` lehnt mehrere Redirect-URIs in Codex-Client-ID ab | Niedrig | Fallback auf Port 1455, Hinweis im Doc |
| Modell-Liste veraltet bei OpenAI-Updates | Hoch | Plugin-Update-Pfad, optional Probe-Request als Folge-Feature |
| ChatGPT-Plus-Subscriber haben keine Codex-Quota | Niedrig | Vor Release mit Test-Account verifizieren, Quota-Fehler in UI verstaendlich anzeigen |
| Endpoint-Drift bei `chatgpt.com/backend-api/codex/responses` | Hoch (intrinsisch) | Schema-Validation, klarer Fehler statt stiller Fehlinterpretation, Disclaimer im UI, Endpoint-Drift-Indikator |

## Open Items (deferred to coding)

- **JWT-Claim-Name fuer `chatgpt-account-id`:** Entweder `https://api.openai.com/auth.chatgpt_account_id` oder `chatgpt_account_id` direkt im Top-Level. Muss beim ersten echten Login-Test geklaert werden.
- **Plan-Tier-Claim:** Genauer Claim-Name (`plan`, `subscription_plan`, `tier`) ist erst beim Login-Test sichtbar.
- **Codex-Response-Event-Typen:** Vollstaendige Liste aller `event:`-Werte ist nur ueber empirische Tests bestimmbar. Mapper bekommt `default`-Branch mit `console.warn` plus Drift-Indikator.
- **Probe-Request fuer Modelle:** Zuerst Hardcode-Liste, sobald `/models`-Endpoint-Verhalten klar ist, ggf. Probe nachschieben.

## Compliance

- Review-Bot: kein `fetch()`, kein `innerHTML`, keine `any`, keine floating Promises, keine inline-Styles im Plugin-DOM.
- Eslint-Disable nur mit `-- reason`-Kommentar.
- Mobile-Inkompatibilitaet im UI klar markiert.

## Consistency Check

| Decision | ADR | plan-context-021.md | Status |
|----------|-----|---------------------|--------|
| Eigener Provider plus Singleton-Service | ADR-088 | Architecture Style | OK |
| Node-`https`-Streaming | ADR-088 | HTTP-Tabelle | OK |
| Verschachteltes Settings-Schema | ADR-088 | Data Model | OK |
| Type-Guards statt zod | ADR-088 | Implementation Sequence | OK |
| JWT-Mini-Decoder | ADR-088 | Implementation Sequence | OK |
| Hardcode-Modell-Liste | ADR-088 | Open Items | OK |
| PKCE Loopback Server im Renderer | ADR-089 | HTTP-Tabelle | OK |
| Port-Range 1455 bis 1460 | ADR-089 | Security-Targets | OK |
| 5-Minuten-Timeout fuer Loopback | ADR-089 | implizit | OK |

Alle Entscheidungen aus den ADRs sind im plan-context konsistent abgebildet.
