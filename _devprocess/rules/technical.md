# Technische Regeln Vault Operator

> Max 150 Zeilen. Stabile Wahrheiten. Aenderungen passieren waehrend
> /architecture und /coding, wenn eine Regel ihre Bedeutung aendert.

## Stack

- Sprache: TypeScript (strict)
- Plattform: Obsidian Plugin (Electron via Obsidian)
- Build: esbuild mit Deploy-Plugin
- Tests: Vitest
- Lint: ESLint + Prettier
- AI SDKs: Anthropic, OpenAI, Bedrock, OpenRouter, GitHub Copilot
- DB: sql.js (WASM, KnowledgeDB) und MemoryDB-Tabellen daneben
- Vector: KnowledgeDB Vektor-Tabelle, qwen3-embedding-8b ueber OpenRouter

## Befehle

```bash
npm run build              # Production Build
npm run dev                # Watch-Mode mit Auto-Deploy
npm run deploy             # Nur Deploy
npm run test               # Vitest
npm run lint               # ESLint
npm run typecheck          # tsc --noEmit
```

Build und Deploy nach JEDEM Implementierungsschritt.

## Konventionen

- Functional components, named exports, PascalCase Klassen
- Conventional Commits: feat:, fix:, chore:, docs:, refactor:, style:
- Co-Authored-By Claude in jedem Commit
- Vault-Pfade ueber `vault.configDir`, niemals hardcoded `.obsidian`
- `instanceof` statt `as TFile` / `as TFolder`
- Floating Promises mit `void` prefix oder `.catch()`
- `unknown` statt `any`, mit Type Guards

## Obsidian Community Plugin Review-Bot Rules (zwingend)

| Verboten                            | Ersatz                                       |
|-------------------------------------|----------------------------------------------|
| `console.log()` / `console.info()`  | `console.debug()`, `.warn()`, `.error()`     |
| `fetch()`                           | `requestUrl` aus obsidian oder SDK-Clients   |
| `require()`                         | ES `import` (Ausnahmen siehe unten)          |
| Hardcoded `.obsidian`               | `vault.configDir`                            |
| `element.style.X = Y`               | CSS-Klassen `.agent-u-*` oder `style.setProperty()` |
| `innerHTML`                         | Obsidian DOM API (`createEl`, `createDiv`)   |
| `any` Types                         | `unknown` plus Type Guards                   |
| Floating Promises                   | `void` Prefix oder `.catch()`                |
| `as TFile` / `as TFolder`           | `instanceof` Checks                          |
| `Vault.delete()` / `Vault.trash()`  | `FileManager.trashFile()`                    |

`require()` Ausnahmen mit `// eslint-disable-line -- reason`:
- `require('electron')` in SafeStorageService
- `require('child_process')` im Sandbox-Bridge
- `require('fs')` fuer Knowledge-DB-Tooling
- `createSandboxExecutor`-Aufruf

Vollstaendige Referenz: `memory/review-bot-compliance.md`.

## Test-Patterns

- Integration-Tests primaer, Unit-Tests sekundaer
- Mocks: nur externe Services (Anthropic API, Obsidian Vault), nie
  interne Module
- Coverage-Schwelle: 30% lines, 35% functions

## Code-Qualitaet

- Fehler an der Wurzel beheben, nicht mit `eslint-disable` stillstellen
- Bei unbekannten APIs: Doku lesen, nicht raten
- Kein impliziter Datenverlust (DROP COLUMN, Schema-Rewrites) ohne
  explizite User-Freigabe
- KnowledgeDB-Migrationen: WriterLock VOR Spalten-Mutation acquiren
  (Lehre aus BUG-012)

## Build Output

- Plugin-Bundle: `main.js` im Repo-Root
- Deploy-Pfad ueber `.env` (`PLUGIN_DIR`), aktuell iCloud:
  `~/Library/Mobile Documents/iCloud~md~obsidian/Documents/NexusOS/.obsidian/plugins/vault-operator/`

## Wegweiser

Fuer "wo lebt X?"-Fragen: `grep "<concept>" src/ARCHITECTURE.map` und
dann den JSDoc-Header der Entry-Point-Datei lesen. Diese Datei hier
listet Regeln, nicht Pfade.

## Markdown-Link-Konvention (`_devprocess/`)

- Cross-Doc-Links zwischen Markdown-Dateien sind **datei-relativ**
  (CommonMark): aus `_devprocess/analysis/X.md` zu einer ADR via
  `[..](../architecture/ADR-NN-...md)`, zu einem Sibling via
  `[..](Y.md)`.
- Quellcode-Verweise sind **datei-relativ** (aus
  `_devprocess/analysis/X.md` zu einer Code-Datei via
  `[ToolRegistry.ts:42](../../src/core/tools/ToolRegistry.ts#L42)`).
  Workspace-relative Schreibweise (`src/core/...` ohne `../../`) ist
  Drift; CommonMark resolved sie nicht und die DIA-Linkpruefung
  meldet sie als Dead-Link. Konvertierung mechanisch moeglich
  (siehe Hygiene-Lauf 2026-05-17).
- Bei `_devprocess/`-internen Links **immer datei-relativ**
  (`../requirements/...`, nicht `_devprocess/requirements/...`).
  Letztere Form sieht aus wie ein absolut-aussehender Pfad, schlaegt
  aber bei jeder Resolver-Strategie ausser workspace-root fehl.

## ADR-ID-Schreibweise

- ADR-Dateinamen verwenden die natuerliche ID-Form ohne fuehrende
  Nullen (`ADR-09-...`, `ADR-90-...`, `ADR-123-...`). Drei-stellige
  Schreibweise mit fuehrender Null (`ADR-090`, `ADR-063`) ist Drift
  und sollte vermieden werden.
- Frontmatter-Felder (`adr-refs: [ADR-90, ADR-63, ADR-123]`),
  BACKLOG-Zeilen und ADR-Cross-Refs verwenden dieselbe Form ohne
  fuehrende Nullen.
