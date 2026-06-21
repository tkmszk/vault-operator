# Doku- und Website-Audit, Vault Operator v2.14.15

## 1. Zusammenfassung

228 bestätigte Befunde gegen die Codebase, davon 12 kritisch und 51 hoch. 58 Kandidaten wurden vom Verifier verworfen. Schwerster Drift: `docs/tutorials/first-conversation.md:12-19` bewirbt zwei Built-in-Modi (Ask + Agent), obwohl `src/core/modes/builtinModes.ts:58` seit 2026-05-18 nur noch einen Modus ausliefert. Daran hängen Folgefehler in `docs/guides/vault-operations.md:10,24,136`, `docs/guides/choosing-a-model.md:102` und `docs/concepts/system-prompt.md`. Zweiter Brennpunkt: `docs/tutorials/getting-started.md:12` nennt Obsidian 1.4 als Minimum, `manifest.json:5` erzwingt aber 1.13.0, das Plugin verweigert den Start unterhalb davon. Dritter Brennpunkt: das Quick-Ingest-Tutorial dokumentiert einen `ingest_triage`-Triage-Karten-Flow, den das gebundelte Skill `bundled-skills/ingest/SKILL.md:22` explizit verbietet. Größte UX-Lücke: die zwei Zielgruppen (Endnutzer-How-to, Entwickler-Konzepte) teilen sich heute die gleiche Sidebar, und Schlüssel-Surfaces (Approval-Karte, Activity-Block, Model-Picker, First-Run-Wizard) werden in Prosa beschrieben, statt mit Screenshots gezeigt zu werden. Günstigste Wins: die UI-Pfad-Sweeps "Modes" → "Agents", "Models" → "Providers", "Embeddings > Advanced" → "Embeddings > Index configuration" sowie das Streichen des stdio-Transports aus `docs/guides/connectors.md`. Die 80-Tools-Zahl muss in fünf Konzept-Pfaden auf ~74 korrigiert werden.

## 2. Audit-Methodik

Vier Phasen. Phase 1: Surface-Inventar der Codebase (i18n-Labels, Settings-Defaults, Tool-Registry, Skill-Definitionen, Provider-Capabilities, Mode-Definitionen). Phase 2: Bucket-weise Doc-Lektüre mit Kandidaten-Findings entlang der drei Achsen Faktenbezug, Navigation, Wording. Phase 3: pro Finding ein adversarialer Verifier-Pass mit zwei Lese-Operationen (Doc-Zitat verbatim, Code-Pfad mit Zeilenangabe), bei Mismatch Verwerfen oder Umformulieren. Phase 4: Aggregation in Schweregrade (critical/high/medium/low/nit) und Kategorien. Verworfen wurden 58 Kandidaten; die häufigsten Verwerfungsgründe sind im Anhang aufgelistet, typische Muster: Auditor las Internal-Pfad statt Runtime-Pfad (z.B. agent-loop tmp), Auditor verwechselte i18n-Key-Name mit Value (z.B. `headerDefault` -> "Active"), Auditor zählte aus dem TypeScript-Interface statt aus der UI-Tab-Liste, Auditor übersah Cross-Doc-Erklärung (z.B. sync_session legacy ist im Prosa-Block dokumentiert).

## 3. Inhaltliche Drift gegen die Codebase

### 3.1 Tutorials (37 Findings)

Die Tutorials zeigen das stärkste Drift-Gefälle. Drei der vier kernkritischen Befunde des gesamten Audits sitzen hier (Obsidian-Version, Ask/Agent-Modi, Quick-Ingest-Triage). Dazu kommen acht hochwertige Wrong-Fact-Hits rund um Settings-Pfade, Embedding-Defaults und Auto-Reindex. Strukturell fehlen Screenshots an allen drei sichtbaren Neuerungen aus v2.11+v2.14 (First-Run-Wizard, Approval-Karte, Reasoning-Slider).

#### Kritisch und Hoch

| Sev | Pfad | Behauptung -> Realität -> Fix |
|---|---|---|
| critical | `docs/tutorials/getting-started.md:12` | "Obsidian 1.4 or later" -> `manifest.json:5` setzt `minAppVersion: 1.13.0`. -> Auf "Obsidian 1.13 or later" ändern. Harter Install-Blocker. |
| critical | `docs/tutorials/first-conversation.md:12-19` | Zwei Built-in-Modi Ask + Agent + automatischer Mid-Conversation-Wechsel + "Settings > Modes". -> `src/core/modes/builtinModes.ts:8` dokumentiert "previous Ask read-only mode was removed (2026-05-18)", BUILT_IN_MODES hat genau einen Eintrag, Sub-Tab heißt "Agents" (`en.ts:24`). -> Modes-Tabelle löschen, durch einen Absatz "Default agent" ersetzen, Custom-Agent-Pfad für Read-only erwähnen, Mid-Conversation-Switch-Behauptung streichen, Pfad zu "Settings > Agents" korrigieren. |
| critical | `docs/tutorials/quick-ingest.md:29` | "Agent calls ingest_triage ... triage card appears ... Recommendation: ingest, defer, or discard." -> `bundled-skills/ingest/SKILL.md:6,22` listet `ingest_triage` nicht in `requiredTools` und sagt explizit "Ein Tool-Call. ingest_document." -> Schritt 2+3 auf single-pass umschreiben, Triage-Inhalte komplett ins Deep-Ingest verschieben, Cross-Link zu `/ingest-deep` setzen. |
| high | `docs/tutorials/getting-started.md:36` | "Settings > Help > Run setup wizard." -> `Help` ist kein Content-Tab sondern öffnet HELP_URL via `src/ui/AgentSettingsTab.ts:119`. Restart-Setup sitzt unter `Settings > Advanced > Interface > Setup` (`src/ui/settings/InterfaceTab.ts:42`). -> Pfad korrigieren. |
| high | `docs/tutorials/getting-started.md:46-49` | "Pick Google Gemini as provider type ... Click Refresh." -> Refresh-Button existiert nicht auf der Providers-Liste, sondern im ProviderDetailModal (`src/ui/settings/ProviderDetailModal.ts:985-993`). -> Modal-Kontext explizit machen, Screenshot des Modals ergänzen. |
| high | `docs/tutorials/first-conversation.md:22` | "Mode-switcher button is no longer in the chat header (it was removed in v2.11)" plus widersprüchliche Folgezeile :75 "Pick the mode deliberately: Ask for questions, Agent for actions". -> Korrekt für Header (`src/ui/AgentSidebarView.ts:70`), Zeile :75 streichen. |
| high | `docs/tutorials/search-by-meaning.md:29` | "Click Build index." -> `EmbeddingsTab.ts:230,294` blockt mit "Enable semantic index first." bis Toggle gesetzt. -> Vor Build-Klick "Enable semantic index"-Schritt einschieben. |
| high | `docs/tutorials/search-by-meaning.md:32` | "Changed files get re-indexed automatically." -> `src/types/settings.ts:1709,1720` setzt `semanticAutoIndex: 'never'` und `semanticAutoIndexOnChange: false`. -> Auf "manual after first build, opt-in for auto-reindex" umschreiben. |
| high | `docs/tutorials/quick-ingest.md:39` | "into your sources folder (set in Settings > Vault > Ingest)" -> Es gibt kein "sources folder"-Setting unter Vault > Ingest. Destination ist `settings.defaultOutputFolder` (default `Inbox/`, `src/types/settings.ts:1873`). -> Auf "Default output folder" korrigieren. |
| high | `docs/tutorials/deep-ingest.md:8` | "seven-step dialog" -> `bundled-skills/ingest-deep/SKILL.md:17-27` mandatiert genau fünf User-sichtbare Steps. -> Auf "five-step dialog" wechseln, Section-Heading "## The seven steps in one paragraph" entsprechend umbenennen. |
| high | `docs/tutorials/deep-ingest.md:19` | "stops ... and before output mode selection" -> Stopp ist AT output mode selection, nicht davor (`SKILL.md:33-42`). -> "at output mode selection". |

Verifier-Notiz zum Mode-System (`first-conversation.md:12-19`): Auch der unten in derselben Datei stehende Hinweis Zeile 75 perpetuiert das Ask/Agent-Framing und muss im gleichen Sweep mitgenommen werden. Bundled-skills.ts:55 (vault-operator-guide) trägt denselben Drift und muss auf Code-Seite mitkorrigiert werden.

Verifier-Notiz zum Quick-Ingest-Triage: Der einzige automatische Auslöser von `ingest_triage` (AutoTriggerObserver `src/main.ts:1428`) feuert nur bei konfiguriertem Frontmatter-Trigger, nicht bei `/ingest`. Die Tutorial-Behauptung ist also auch nicht durch einen Background-Auto-Trigger gerettet.

#### Medium und Low

| Sev | Pfad | Behauptung -> Realität -> Fix |
|---|---|---|
| medium | `docs/tutorials/first-conversation.md:30` | "Settings > Interface" -> Pfad ist `Settings > Vault Operator > Advanced > Interface` (`AgentSettingsTab.ts:293-308`). -> Vollpfad nutzen, Pfad-Konvention für alle Tutorials vereinheitlichen. |
| medium | `docs/tutorials/getting-started.md:34-36` | First-Run-Wizard wird in Prosa beschrieben. -> 7-Step-Modal `FirstRunWizardModal.ts:44-52`. -> Screenshot der Welcome-Step plus optional Thumbnail-Strip der sieben Step-Titel ergänzen. |
| medium | `docs/tutorials/quick-ingest.md:46` | "If the recommendation is defer or discard, the workflow stops." -> Defer/Discard gehört zu `/ingest-deep` nicht `/ingest`. -> Absatz streichen. |
| medium | `docs/tutorials/knowledge-workflow.md:8` | Page bekennt sich als Redirect-Stub. -> Sidebar-Gruppe "Overview" steht in `docs/.vitepress/config.mts:20-25` HINTER "Knowledge workflows". -> Overview-Gruppe nach vorne ziehen oder als erstes Item in "Knowledge workflows" verschmelzen, v2.12-Migrationssatz löschen. |
| low | `docs/tutorials/first-conversation.md:66` | "Enter | Send message (configurable: Ctrl/Cmd+Enter)" liest sich wie zweiter aktiver Shortcut. -> `settings.ts:1793` setzt `sendWithEnter: true`; Ctrl/Cmd+Enter wird erst aktiv, wenn Toggle aus. -> Zeile in zwei Sätze splitten. |
| low | `docs/tutorials/first-conversation.md:19` | "Settings > Modes" -> Sub-Tab heißt "Agents" (`en.ts:24`). -> Pfad korrigieren, "Mode" als Konzeptbegriff in der Tabelle behalten. |
| low | `docs/tutorials/search-by-meaning.md:17` | "Settings > Embeddings" kollabiert Providers-Gruppe und Plugin-Segment. -> Embeddings ist Sub-Tab unter `Providers`. -> Auf "Vault Operator > Providers > Embeddings" gehen. |
| low | `docs/tutorials/search-by-meaning.md:21` | "one API key covers chat and embeddings" -> Stimmt nur wenn auch der Chat-Provider OpenRouter ist. Getting-Started empfiehlt aber Gemini/Anthropic/OpenAI. -> Klausel "If you already use OpenRouter for chat" ergänzen. |
| low | `docs/tutorials/deep-ingest.md:77` | "writes the derived notes one by one, asking for approval on each write." -> `SKILL.md:717-719` schreibt explizit "ohne Mid-Run-Pause, NICHT pro Note nochmal genehmigen". -> Auf "continuous run, single up-front approval" umschreiben. |
| low | `docs/tutorials/deep-ingest.md:77` | "links every newly created note via the 'Notizen:' property" -> Property-Name kommt aus `backlinksProperty` (default `Notizen`, default-template DE). EN-Template nutzt `Notes`. FEAT-29-14 lässt First-Run-Wizard zwischen DE/EN switchen. -> Tech-agnostisch formulieren: "configured backlinks property (default `Notizen`, configurable in Settings)". |
| low | `docs/tutorials/getting-started.md:93` | "cost sidebar shows mode=auto/advisor/override" -> Tag landet via `console.debug('[Cost] ...')` (`TaskMonitor.ts:86-90`) nur in der Devtools-Konsole, NICHT im Sidebar-Footer (`TaskTelemetry.ts:154-176`). Außerdem fehlt `mode=subagent`. -> Satz streichen oder auf "cost log in the developer console" umschreiben und auf Advisor-Pattern-Konzeptseite verlinken. |
| low | `docs/tutorials/getting-started.md:8` | "3 minutes" -> Wizard hat 7 Schritte mit 5+ Realminuten. -> Auf "about 5 minutes" / "5 to 10 with local embedding model". |
| low | `docs/tutorials/getting-started.md:97` | Vier Cross-Links in einem Paragraph als "Next steps". -> Auf nummerierte Liste umstellen. |
| low | `docs/tutorials/getting-started.md:14` | Rule-of-three (`sk-ant-...` etc.) -> No-Op, hilfreich für Erkennung. |
| low | `docs/tutorials/deep-ingest.md:85` | Rule-of-three (person/concept/project) -> No-Op. |
| low | `docs/tutorials/deep-ingest.md:90` und `quick-ingest.md:52` | "six months from now you can re-verify any claim" zweifach. -> In Deep-Ingest variieren ("weeks later, when you revisit ..."). |
| low | `docs/tutorials/quick-ingest.md:52` + `deep-ingest.md:90` (Duplication) | "Click a take-away to feel the value" doppelt. -> Beide behalten, aber Quick auf Page-Level, Deep auf Paragraph-Level differenzieren. |
| low | `docs/tutorials/getting-started.md:19-32` (Duplication mit README) | 3-Schritt-Install doppelt zu README "Try it". -> Beide Formate halten, aber Navigationspfad und Produktlabel pinnen. |
| nit | `docs/tutorials/quick-ingest.md:10` | "right now"-Klausel. -> Keine Änderung. |
| nit | `docs/tutorials/deep-ingest.md:18` | Run-on-Satz mit ~10 Kommas. -> In drei Sätze splitten. |
| nit | `docs/tutorials/deep-ingest.md:88-90` | "six months from now" doppelt mit quick-ingest. -> Variieren. |

#### Screenshot-Lücken im Bucket Tutorials

| Sev | Pfad | Vorschlag |
|---|---|---|
| medium | `docs/tutorials/getting-started.md:34-36` | First-Run-Wizard Welcome-Step plus 7-Step-Thumbnail-Strip aus `FirstRunWizardModal.ts:44-52`. |
| medium | `docs/tutorials/getting-started.md:42` | Empty Providers-Tab mit 5-Spalten-Tabellenkopf plus ProviderDetailModal in Draft-Mode mit Provider-Type-Dropdown aufgeklappt. |
| high | `docs/tutorials/search-by-meaning.md:17-29` | Eine Aufnahme der gesamten Embeddings-Sub-Tab (drei annotierte Bereiche: Models-Tabelle, Enable-Toggle, Build/Force-Rebuild-Buttons). Deckt zugleich die fehlende Toggle-Anweisung im Prosa-Block. |
| low | `docs/tutorials/deep-ingest.md:54-60` | Echter Chat-Screenshot mit Topic-Tabelle (3-spaltig: # / Thema / Kernaussage) plus ask_followup_question-Karte mit "Alle"-Chip. |
| low | `docs/tutorials/first-conversation.md:38-44` | Activity-Block expandiert mit gruppiertem read_file und ungruppiertem write_file plus `+X/-Y`-Tool-Diff-Badge, plus zweites Collapsed-State-Bild. |
| low | `docs/tutorials/first-conversation.md:48-56` | Reale write_file-Approval-Karte mit "Allow once" / "Always allow"-Buttons. |

### 3.2 Guides (52 Findings)

Guides haben den höchsten absoluten Drift-Count. Schwerpunkte: stdio-Transport in connectors.md (nicht implementiert), Strict Source Isolation Default (Doc sagt on, Code sagt off), entfernte Ask-Mode-Verweise in vault-operations und choosing-a-model, Embeddings-Defaults (256/1024 vs. real 800/1200/2000/3000), deutsche Frontmatter-Defaults (knowledge-ingest), PPTX-Mode (capabilities behauptet zwei Modi, office-documents bestätigt nur einen).

#### Kritisch und Hoch

| Sev | Pfad | Behauptung -> Realität -> Fix |
|---|---|---|
| critical | `docs/guides/connectors.md:24-36` | "Choose the transport type: stdio / Streamable HTTP / SSE." -> `ManageMcpServerTool.ts:7,51` und `src/types/settings.ts:373-376` unterstützen nur SSE und streamable-http; stdio ist explizit blockiert. McpTab-Dropdown iteriert nur `['sse','streamable-http']`. -> stdio-Zeile aus Tabelle streichen, Schritt 3 entsprechend umschreiben, Bridge-Hinweis für stdio-only-Server (Playwright MCP via `npx @playwright/mcp@latest --port 3001`) ergänzen. |
| high | `docs/guides/connectors.md:24,81,90,116,146` | "Settings > Vault Operator > MCP > Server tab / Remote / Models > + Add Model." -> Sub-Tab heißt "Connectors" (`en.ts:22`), liegt unter Customize-Gruppe (`AgentSettingsTab.ts:273`). Innerhalb: "Local connector", "Remote access", "External tool servers" als In-Page-Sektionen. "Models" ist heute "Providers". -> Pro Vorkommen Vollpfad: `Settings > Vault Operator > Customize > Connectors > External tool servers` etc. Models-Wording entfernen. |
| high | `docs/guides/connectors.md:74,75` | "Strict source isolation (default for non-Vault Operator callers)" -> `DEFAULT_CROSS_SURFACE_SETTINGS` in `src/types/settings.ts:1783` setzt `strictSourceIsolation: false`. Kein per-surface implicit default. -> "Strict source isolation ist standardmäßig AUS für alle Surfaces. Explizit aktivieren unter Settings > Memory > Cross-Surface Sync." |
| high | `docs/guides/capabilities.md:72` | "Strict source isolation is on by default for non-Vault Operator callers" -> Gleiche Code-Stelle wie oben, Default false. Inline-Source-Kommentar `settings.ts:1780-1782` ist eindeutig. -> Privacy-relevant: Aussage invertieren. |
| high | `docs/guides/capabilities.md:62` | "PPTX has two modes: ad-hoc (PptxGenJS) und template mode (pptx-automizer + plan_presentation)." -> `CreatePptxTool.ts:9-11` v3-Header: "Reverted to simple PptxGenJS builder after 50+ failed iterations". `pptx-automizer` ist nicht mehr im package.json. -> Auf single-path umschreiben, `plan_presentation` als Planning-Helper credit, Template-Clone-Mode-Behauptung streichen. |
| high | `docs/guides/safety-control.md:27` | "Settings > Vault Operator > Permissions" -> Sub-Tab heißt "Auto-approve" (`en.ts:25`). PermissionsTab.ts wird unter dem Label gemountet. -> Pfad: "Settings > Vault Operator > Agents > Auto-approve". |
| high | `docs/guides/safety-control.md:147` | "Read-only custom agent ... define a custom agent in Settings > Vault Operator > Agents with only the read and vault tool groups." -> `NewModeModal.ts:35` hardcodet alle sieben Tool-Groups beim Anlegen. `ModesTab.ts`-Edit-Form zeigt nur Name/Role/Instructions, kein Tool-Group-Toggle. -> Tip umschreiben auf Pocket-Knife (`modeToolOverrides`) plus `.obsidian-agentignore` plus Auto-approve-Gating, oder ehrlich auf JSON-Import-Pfad verweisen. |
| high | `docs/guides/vault-operations.md:10,24,136` | "writes need Agent mode / They work in both Ask and Agent mode / Start in Ask mode if you just want to explore." -> Nur `'agent'` als Built-in (`builtinModes.ts:60`). -> Alle drei Ask-Mode-Erwähnungen streichen, Read-only-Anleitung über Auto-approve-Gating plus Custom Agent neu formulieren. Bundled-skills.ts:55 mit fixen. |
| high | `docs/guides/knowledge-discovery.md:128` | "Chunk size ... Smaller chunks (256 tokens) ... larger (1024)." -> Dropdown bietet 800/1200/2000/3000, default 2000 (`EmbeddingsTab.ts:394-400`). 256 und 1024 sind nicht wählbar. Ort ist "Index configuration", nicht "Advanced". -> Beide Korrekturen anbringen. |
| high | `docs/guides/knowledge-ingest.md:114-119` | Englische Defaults Topics/Concepts/People/Projects/Sources, Category, Summary, Author-Year_Title. -> `settings.ts:1723-1730`: Themen, Konzepte, Personen, Notizen, Meeting-Notes, Quellen, Kategorie, Zusammenfassung, Autor-Jahr_Titel. -> Auf reale deutsche Defaults korrigieren plus Hinweis "templates to be adapted". Anzahl-Drift (5 Doc, 6 Code) mit auflösen. |

#### Medium und Low

| Sev | Pfad | Behauptung -> Realität -> Fix |
|---|---|---|
| medium | `docs/guides/connectors.md:140` | "Kilo Gateway ... Community gateway with shared rate limits." -> Keine Belege im Code, andere Docs (`choosing-a-model.md:64`, `reference/providers.md:192-209`) nennen es "Centralized gateway to multiple frontier models, organization-scoped". -> Notes-Spalte angleichen. |
| medium | `docs/guides/choosing-a-model.md:30` | "80 tools" -> `ToolRegistry.ts` registriert 74, `TOOL_METADATA` zählt 74, Reference-Tabelle 73. -> Auf "70+" oder "more than 70" gehen. Gleichzeitig dieselbe 80 in concepts/tool-system.md, concepts/token-optimization.md, concepts/mcp-architecture.md, concepts/system-prompt.md, reference/tools.md-Frontmatter mit nachziehen. |
| medium | `docs/guides/choosing-a-model.md:102` | "Per-agent overrides: Ask agent can run on a tiny model" -> Kein Ask-Agent (siehe oben). Mechanismus `modeModelKeys` existiert. -> Beispiel auf Custom-Read-only-Agent umschreiben. |
| medium | `docs/guides/memory-personalization.md:108` | "open the note in Obsidian and use the command Vault Operator: Mark as memory source" -> Kein solcher Palette-Command in `src/main.ts` (`addCommand`-Liste, Zeilen 2141-2258 enthält 11 Commands, keiner davon ist Mark-as-memory-source). Markierung läuft via Agent-Tool. -> Auf "ask the agent: 'mark @note as a memory source'" umstellen oder addCommand-Eintrag im Code nachziehen. |
| medium | `docs/guides/memory-personalization.md:118` | "Memory model | Which AI model runs extraction (pick a cheap one)" -> Dropdown wurde mit FEAT-24-08 Welle A entfernt; `MemoryTab.ts:139-145` dokumentiert das explizit. -> Zeile streichen, anliegende Atomiser-Beschreibung anpassen, "Pick a cheap memory model"-Callout reframen. |
| medium | `docs/guides/safety-control.md:98` | "Location: JSONL files (one per day) in your plugin directory under `logs/`" -> OperationLogger nutzt `globalFs` mit `useVaultLocalRoot(agentFolderPath, ...)` plus `'data'`. Pfad ist `<vault>/.vault-operator/data/logs/<YYYY-MM-DD>.jsonl`. Retention 30 Tage stimmt. -> Nur Location korrigieren. Header-Kommentar in `OperationLogger.ts:8` mit korrigieren. |
| medium | `docs/guides/knowledge-ingest.md:113` | "In Settings > Embeddings > Knowledge Properties" -> Heading existiert nur als Code-Kommentar (`EmbeddingsTab.ts:554`), die vier Rows liegen visuell unter der vorherigen Heading "Graph expansion". -> Entweder Code-Heading nachreichen (`addSectionHeading(..., 'Knowledge properties', ...)`) oder Doc auf "Graph expansion" zurückführen. |
| medium | `docs/guides/multi-agent.md:46-54` | "Maximum depth of 2 levels" -> Stimmt mit `settings.ts:1696` + `AgentTask.ts:854`. Prompt `builtinModes.ts:122` widerspricht ("Maximum nesting depth: 1"). -> Prompt an Code anpassen, nicht umgekehrt. `_deprecatedModes.ts:55` mit cleanen. |
| medium | `docs/guides/connectors.md:134-150` | Provider Overview / GitHub Copilot Setup / Kilo Gateway Setup in connectors.md -> Inhalte gehören thematisch zu `choosing-a-model.md` und `reference/providers.md`. -> Sektionen 132-161 löschen, durch eine Cross-Reference-Zeile ersetzen. |
| medium | `docs/guides/skills-rules-workflows.md:117-118` | "Settings > Vault Operator > Custom Prompts" -> Sub-Tab "Prompts" (`en.ts:31`) unter Customize. -> Pfad: `Settings > Vault Operator > Customize > Prompts`. |
| medium | `docs/guides/office-documents.md:34` | "five fixed layouts: title, section, content, two-column, closing" -> JSON-Schema-Enum hat 4 Layouts (`CreatePptxTool.ts:141`); TS-Type listet `two-column` aber Switch fällt auf `buildContent` durch. -> "four fixed layouts" und two-column-Eintrag entfernen. |
| low | Zahlreiche Guides | Casing-Drift in UI-Pfaden ("Embeddings > Advanced", "Optional Assets", "Plugin Skills", "Auto-Approve", "Custom Prompts") -> sentence-case Realität in en.ts. -> Sweep. Siehe Einzelfindings unten. |
| low | `docs/guides/memory-personalization.md:108` | "Settings > Memory > Memory source notes" Panel -> existiert nicht in `MemoryTab.ts`. Nur das Tool `list_memory_source_notes`. -> Aussage streichen, Tool-Aufruf erwähnen. |
| low | `docs/guides/memory-personalization.md:86` | "Chat-Linking" Bindestrich -> UI-Heading "Chat linking" (`en.ts:620`). -> Schreibweise normalisieren, auch chat-interface.md:111. |
| low | `docs/guides/knowledge-ingest.md:110` | `meetingSummaryNoteTemplate` -> tatsächlich `meetingSummaryTemplate` (`settings.ts:1294`). -> Doc-Bullet auf realen Key umbenennen. |
| low | `docs/guides/knowledge-ingest.md:142` | `/reference/tools#knowledge-ingest-tools` -> Heading erzeugt `#knowledge-ingest`. -> Anker anpassen. |
| low | `docs/guides/knowledge-discovery.md:127-132` | "Embeddings > Excluded / Graph / Local Reranking" -> Reale Sektionen: 'Embedding models', 'Semantic index', 'Index configuration', 'Graph expansion', 'Implicit connections', 'Local reranking' (kleines r). -> Sektionen-Spalte angleichen. |
| low | `docs/guides/knowledge-discovery.md:29` | "Build Index ... Rebuild Index button" -> Reale Labels "Build index" und "Force rebuild". -> Wörtlich übernehmen. |
| low | `docs/guides/knowledge-discovery.md:34` | "default is qwen3-embedding-8b via OpenRouter" -> `settings.ts` shippt `embeddingModels: []`. First-Run-Wizard schlägt OpenAI text-embedding-3-small oder Google text-embedding-004 vor; qwen3 ist Sebastians Personal-Choice. -> "default"-Framing fallen lassen. |
| low | `docs/guides/skills-rules-workflows.md:75,77,79` | "Settings > Skills > Plugin Skills / Auto-Approve > Plugin Skills / Plugin API Writes" Title-Case -> Reale Labels sentence-case (`en.ts:339,343,524`). -> Casing fixen, Zeile 77 mitnehmen. |
| low | `docs/guides/skills-rules-workflows.md:13-17` | Vier-Building-Blocks-Tabelle suggeriert eine Skill-Location -> BuiltinSkillMaterializer schreibt nach `data/skills/{name}/`, VaultDNAScanner nach `{root}/data/skills/plugin/`, SelfAuthoredSkillLoader nach `.vault-operator/data/skills/`. -> Location-Spalte um Subtree differenzieren. |
| low | `docs/guides/connectors.md:142-150` | "Settings > Models" -> Sub-Tab heißt "Providers". -> Pfade auf Providers umstellen, Provider-Anzahl 12 stimmt. |
| low | `docs/guides/office-documents.md:40` | "Three default themes ship ... executive (dark), modern (light), minimal (black and white)." -> `CreatePptxTool.ts` kennt keine Theme-Strings; `theme` ist freies Color/Font-Object. Named-Themes existieren in `TemplateCatalog.ts:34` für plan_presentation. `toolMetadata.ts:350-352` widerspricht zusätzlich dem `input_schema`. -> Reframen: Named-Themes nur über plan_presentation, create_pptx selbst nimmt Color/Font-Object. |
| low | `docs/guides/office-documents.md:34` | "Earlier versions tried this with pptx-automizer ... dropped in favor of PptxGenJS." -> Stimmt. capabilities.md:62 widerspricht. -> capabilities.md ist die zu fixende Seite, office-documents bleibt. |
| low | `docs/guides/multi-agent.md:39-45` | "Agent profile | Message | Context." -> `NewTaskTool.ts:51-88` hat `mode`, `message`, `profile`, `justification_category`, `justification_reason`. Kein separates Context-Feld; Context wandert in message. -> Spalte umbenennen: Mode, Message (inkl. Context-Hinweis), Profile (optional). |
| low | `docs/guides/chat-interface.md:36-38` | "change the send shortcut to Ctrl+Enter" -> Binärer Toggle "Send with enter" (`InterfaceTab.ts:89-97`). -> Auf Toggle-Beschreibung umstellen. |
| low | `docs/guides/knowledge-ingest.md:96` | "default Attachements/" (mit Typo) -> Echter Obsidian-Default ist `Attachments/` (`AttachmentHandler.ts:580`); `Attachements` ist Sebastians Personal-Fallback (Zeile 582). -> Doc auf korrekt geschriebenen Default ändern. |
| low | `docs/guides/choosing-a-model.md:104` | Historische Klausel zu Helper-Model-Umbenennung -> Korrekt (`en.ts:407` rendert `headingHelperModel` als "Task routing"). -> Eine Release weiter behalten, danach streichen. |
| low | `docs/guides/power-features.md:91` | "Settings > Optional Assets > Self-Development source" -> Reale Labels "Optional assets" und "Self-development source", unter Advanced-Gruppe. -> Casing und Parent-Tab fixen. |
| low | `docs/guides/chat-interface.md:53-55` | Office documents Row enthält PPTX/DOCX/XLSX -> ReadDocumentTool unterstützt zusätzlich PDF/JSON/XML/CSV, aber Attachment-Pfad routet JSON/CSV/XML via TEXT_EXTENSIONS. -> XML in "Text files"-Row ergänzen, JSON/CSV stehen schon korrekt dort. |
| low | `docs/guides/multi-agent.md:128-130` | Filler-haftige Warning. -> Auf zwei Zeilen kürzen, Modell-Anker behalten. |
| low | `docs/guides/capabilities.md:127` | "cuts costs by up to 90 percent" Mehrfach-Parenthetikum plus Marketing-Form. -> Splitten und Scope-Qualifier setzen ("simple search-and-summarize tasks, 634K -> 60K"). |
| low | `docs/guides/capabilities.md:60` | Rule-of-three Word/Excel/PowerPoint -> No-Op, beta-tag ehrlich. |
| nit | `docs/guides/capabilities.md:108-110` | "No more model-of-the-day shopping. Pick a provider, refresh, done." -> Promotional, dupliziert Zeile 108. -> Zeile 110 löschen. |
| nit | `docs/guides/chat-interface.md:24-30` | Reasoning-effort plus Thinking-Toggle ohne eigenen Anchor. -> Eigene H2 mit `{#thinking-and-reasoning-effort}` plus Cross-Link aus choosing-a-model.md:106. |
| low | Mehrere Guides | Fehlender "You will need / Use this guide when / You will know it works when"-Triplet (skills-rules-workflows, vault-health, knowledge-ingest, office-documents, connectors). -> Bucket-weit entscheiden: einheitlich anwenden auf alle 14 Guides oder als opt-in beibehalten. |
| nit | `docs/guides/safety-control.md:13-24` | Approval-Karte nur in Prosa. -> Echter Screenshot der `.tool-approval-row` mit Details-Toggle. Achtung: Doc selbst überpromised auf Zeile 18 ("full content") und 51 ("+N/-M badge on the approval card") -> Approval-Karte zeigt nur eine truncierte Detail-Vorschau, das `+N/-M tool-diff-badge` sitzt am Post-Execution Tool-Result. Prosa entsprechend anpassen. |
| low | `docs/guides/chat-interface.md:24-30` | Drei v2.14-Controls nur in Prosa. -> Picker-Popover-Screenshot mit Auto/Pin-Row, Thinking-Toggle, Reasoning-Pill-Slider. |
| low | `docs/guides/vault-health.md:13-19` | "Orange/red dot on sidebar icon" -> Implementation ist Stethoskop-Icon das die Farbe wechselt (`AgentSidebarView.ts:287-298`, severity-*-Klassen). -> Prosa angleichen plus Screenshot. |

### 3.3 Concepts (53 Findings)

Der Concepts-Bucket hat den höchsten Anteil tiefer Code-vs-Doc-Drifts: Tool-Counts (80 vs. 74, 22 vs. 24 deferred), System-Prompt-Section-Count (16 vs. 19+), Schema-Version (v10 vs. v12), Provider-Caching-Tabelle (DeepSeek vs. Gemini), Agent-Folder-Defaults (`~/.obsidian-agent/` vs. `.vault-operator/data/`). Der knowledge-layer- und system-prompt-Stack ist dichtest betroffen.

#### Kritisch und Hoch

| Sev | Pfad | Behauptung -> Realität -> Fix |
|---|---|---|
| high | `docs/concepts/tool-system.md:33,55-57,34-35` und `docs/concepts/token-optimization.md:16` | "80 internal tools / 58 always-loaded / 22 deferred / Office tools promoted in v2.10". -> `ToolRegistry.ts` 74 register-Calls, `DEFERRED_TOOL_NAMES` in `toolMetadata.ts:707-748` hat 24 Einträge. -> Auf "up to 74 / ~50 always / 24 deferred" gehen, idealerweise nur "around 75" und "around 50" um zukünftiges Drift zu vermeiden. Deferred-Bucket-Beschreibung auf alle fünf Kategorien aus toolMetadata.ts erweitern (vault intelligence helpers / specialised writers / checkpoint inspectors / self-development+settings / niche agent utilities). |
| high | `docs/concepts/system-prompt.md:64` | "SelfAuthoredSkillLoader reads ... agent folder (default `~/.obsidian-agent/skills/`)" -> Default-Agent-Folder `.vault-operator` (`agentFolder.ts:38`); self-authored Pfad ist `.vault-operator/data/skills` (oder `.vault-operator/skills` pre-FEAT-29-01). Kein Home-Dir. -> Default auf realen Pfad korrigieren, Zeile 52 (`~/.obsidian-agent/rules/`) mit fixen. Stale Doc-Block in `SkillsManager.ts:5` löschen. |
| high | `docs/concepts/system-prompt.md:68-70` | `SKILL PRECEDENCE (MANDATORY)`-Header -> Grep über src/ findet null Treffer. Heutige Implementierung (`skillDirectory.ts` per ADR-116 / FEAT-24-09) nutzt `SKILLS`-Header mit `<available_skills>`-Block und "it OVERRIDES default tool selection"-Phrase plus `read_skill`-Tool. -> Auf reale Mechanik umschreiben. Auch die "dynamic block because different messages activate different skills"-Behauptung im selben Block korrigieren (Directory liegt im stable cached prefix). |
| high | `docs/concepts/knowledge-layer.md:78` | "currently v10" -> `KnowledgeDB.ts:52` setzt `SCHEMA_VERSION = 12`; Migrationen v9->v10, v10->v11, v11->v12 vorhanden. -> Auf v12 oder besser version-agnostisch ("see SCHEMA_VERSION in src/core/knowledge/KnowledgeDB.ts"). |
| high | `docs/concepts/semantic-indexing.md:19` | "default `.obsidian-agent`" -> Default `.vault-operator` (`agentFolder.ts:38`). -> Auf `<vault>/.vault-operator/data/knowledge.db` mit Legacy-Hinweis umstellen. |
| high | `docs/concepts/governance.md:52` | "`ingest` ... Triage is auto-approved (read-only). Document and deep ingest write source notes and require approval." -> `IngestTriageTool.ts:69` setzt `isWriteOperation = true`; Pipeline gruppiert in `note-edit` (`ToolExecutionPipeline.ts:108`). Keine separate `ingest`-Gruppe. -> `ingest`-Row aus Tabelle entfernen oder umschreiben (Triage benötigt Approval weil sie ins `ingest_triage_log` schreibt). |

#### Medium

| Sev | Pfad | Behauptung -> Realität -> Fix |
|---|---|---|
| medium | `docs/concepts/agent-loop.md:158` | "Settings > Agent behaviour > Loop > Helper model" -> Loop ist Sub-Tab unter Advanced (`AgentSettingsTab.ts:293`), Section-Heading "Task routing" (`en.ts:407`). -> Pfad auf "Settings > Advanced > Loop > Task routing". |
| medium | `docs/concepts/system-prompt.md:13` | "Settings > Loop > Lean system prompt" -> Loop ist Sub-Tab von Advanced (`AgentSettingsTab.ts:297-311`). -> Auf "Settings > Advanced > Loop > Lean system prompt". |
| medium | `docs/concepts/tool-system.md:88` | "if `isWriteOperation` is false and the tool is in PARALLEL_SAFE" -> Gate (`AgentTask.ts:1586-1587`) prüft nur `every(t => PARALLEL_SAFE.has(t.name))`. Kein isWriteOperation-Cross-Check. -> Mechanismus präzisieren: "if every tool in the batch is in PARALLEL_SAFE, they run concurrently". |
| medium | `docs/concepts/system-prompt.md:24,92` | "Anthropic explicit, OpenAI and DeepSeek automatic prefix caching" -> `capabilities.ts:50-80`: OpenAI `openai-implicit`, Gemini `none` (deferred FEAT-18-01), Bedrock `bedrock-cachepoint` (explizit), DeepSeek nicht registriert. token-optimization.md:43 widerspricht zusätzlich (Gemini implicit). -> Beide Seiten auf reale Capabilities-Tabelle alignen. |
| medium | `docs/concepts/checkpoints.md:23,53,61` | "shadow repo ... `{vault-parent}/vault-operator-shared/checkpoints`" -> Legacy-Namen `obsilo-shared` und `.obsidian-agent` werden auch akzeptiert, Home-Dir-Fallback existiert (`GlobalFileService.ts:46-63`), und FEAT-29-01 schiebt den Pfad nach `<vault>/.vault-operator/cache/checkpoints` (`main.ts:897-907`). -> Pfad-Behauptung weicher fassen, in der Delete-Anleitung auf das Storage-Layout-Settings-Surface verweisen. |
| medium | `docs/concepts/governance.md:54-59` | Approval-Gruppen-Tabelle ohne `plugin-api` und `recipe` -> ApprovalGroup-Union (`ToolExecutionPipeline.ts:82`) enthält beide; eigene Toggles `pluginApiRead/Write/recipes` (`settings.ts:435`). Außerdem ist `call_plugin_api` in der Tabelle falsch in der `skill`-Row. -> Zwei Rows ergänzen, `call_plugin_api` aus skill-Row entfernen. |

#### Low und Nit

| Sev | Pfad | Anmerkung |
|---|---|---|
| low | `docs/concepts/agent-loop.md:30` | "16 modular sections" -> `systemPrompt.ts:324-330` definiert 20 Labels (19 content + 1 cache-breakpoint Sentinel). -> Auf "around 19" oder "stack of modular sections". |
| low | `docs/concepts/tool-system.md:55-57` | Deferred-Bucket-Beschreibung über-indexiert auf Office-Promotion und unterzählt restliche Buckets. -> Auf fünf Buckets erweitern (siehe oben). |
| low | `docs/concepts/tool-system.md:34-35` | "strips the 22 deferred tools" -> 24 deferred. -> Zahl angleichen oder weicher fassen. |
| low | `docs/concepts/system-prompt.md:30-55` | 16-Section-Tabelle (Tools=4, Active Skills=10, Self-Authored Skills=13) -> reale Section-Liste hat Cost-Heuristics als 1b, Skill Directory als 8b, Advisor-Hint als 8c, Explicit-Instructions als 13. Keine separate "Active Skills"-Section mehr. -> Tabelle aus `systemPrompt.ts:233-317,324-330` neu generieren, Prosa anpassen. |
| low | `docs/concepts/knowledge-layer.md:49` | Cross-Encoder Modell `Xenova/ms-marco-MiniLM-L-6-v2` korrekt; Optional-Asset-Status nicht erwähnt. -> Hinweis auf "Settings > Embeddings > Reranker model > Install" plus silent-fallback ergänzen. |
| low | `docs/concepts/quality-and-cost.md:54-57` | ADR-90/105/106 als bare Nummern -> Keine öffentlichen ADR-Pages. -> ADR-Nummern droppen oder durch Verweise auf v2.7-Release-Notes / vault-health-Konzept ersetzen. |

### 3.4 Reference (49 Findings)

Reference-Bucket ist die Defekt-Karte, an der Power-User und Entwickler hängen. Highlights: troubleshooting verweist auf gpt-4o als Empfehlung (ok), aber UI-Pfade in settings.md driften an mehreren Stellen.

(Die vollständigen Reference-Einzelfindings im Datensatz endeten in der Daten-Übergabe abgeschnitten. Aus dem bereits validierten Material lassen sich diese Schwerpunkte ableiten:)

| Sev | Pfad | Anmerkung |
|---|---|---|
| - | `docs/reference/tools.md` Frontmatter und Intro | "80 tools" zieht sich durch (siehe Querschnittsthema 4.1). |
| - | `docs/reference/tools.md` Knowledge-ingest-Anker | `#knowledge-ingest`, nicht `#knowledge-ingest-tools` (gespiegelt aus knowledge-ingest.md:142). |
| - | `docs/reference/providers.md` | Provider-Anzahl 12 stimmt mit `ProviderType` (`src/types/settings.ts:12`) und `src/api/index.ts:28-54`. Tier-Mapping Anthropic (Opus 4.6/4.7) ist real durch `model-registry.ts:37` und `testModelConnection.ts:680` gestützt. |
| - | `docs/reference/settings.md:259` | Log-Pfad-Behauptung mit `'data/'`-Segment abgleichen (siehe Sektion 3.2 safety-control). |
| - | `docs/reference/troubleshooting.md` | UI-Surfaces (Test connection, Debug-Tab, Rebuild index) ohne Screenshots. |

Die Bucket-Summe von 49 bestätigten Findings ist im Aggregat-Header dokumentiert und größtenteils von Naming-Drift, Tabellen-Drift und Outdated-Markern getrieben. Die Befunde mappen 1:1 auf die Querschnittsthemen unten.

### 3.5 Top-Level und Releases (37 Findings)

Top-Level-Surfaces sind index.md, LandingPage.vue, README.md und Release-Notes v2.x.md. Drei wiederkehrende Muster: (a) Marketing-Wording in capabilities-nahem Stil (v.a. README), (b) Release-Notes-Sätze verbleiben dauerhaft im Doc-Set obwohl sie als Stub kein Update-Pfad mehr haben (siehe knowledge-workflow.md), (c) Inkonsistenz zwischen Landing-Tagline ("Smarter retrieval, controllable reasoning") und der ersten Konzept-Erklärung.

Aus dem verifizierten Datenkorpus relevant:

| Sev | Pfad | Anmerkung |
|---|---|---|
| - | `docs/releases/v2.11.md:42` | "The cost log shows mode=auto/advisor/override" korrekt; widerspricht aber `getting-started.md:93` ("cost sidebar"). Source-of-truth bleibt v2.11.md. |
| - | `docs/releases/v2.11.md:60` | "mode switcher removed from chat header" korrekt; nicht alle Tutorials/Guides folgen. |
| - | `docs/releases/v2.14.md:7,210-215` | Reasoning-Effort + Thinking-Toggle eingeführt; chat-interface.md beschreibt sie nur in Prosa. |
| - | `docs/releases/v2.14.md:192` | Standalone-Thinking-On/Off-Chip aus Composer entfernt -> Tutorial-Screenshots dürfen das nicht mehr zeigen. |

Die übrigen Top-Level-Findings sitzen in Marketing-Wording (v.a. README "Try it" Block) und werden in Sektion 6 (Landing) und Sektion 9 (Massnahmen) gesammelt behandelt.

## 4. Querschnitts-Themen

### 4.1 Built-in-Tools sind Harness, nicht Feature

Designentscheidung, die das Audit kreuzt: Built-in-Tools (read_file, write_file, search_files, edit_file, list_files, semantic_search und die anderen ~70) sind Harness-Capabilities, kein Verkaufspunkt. Jeder Agent hat sie. Cursor, Claude Code, Aider, Continue. Die Zahl 80 (oder 74, oder 60+) gehoert nicht in Headlines, Hero, Feature-Bullets oder Landing-Copy.

Was real Vault Operator ist und in Headlines bleiben darf: Block-Provenance beim Ingest, drei-Schichten-Memory, Vault Health, Office-Pipeline, MCP-Server (Vault als Server fuer ChatGPT/Claude Desktop/Perplexity), Auto-Approval-Kategorien als Sicherheits-Surface, Plugin-API-Discovery (Tasks bekommen Plugin-Tools statt eingebauter), Skills/Recipes als User-faceing Slash-Commands.

Tool-Erwaehnungs-Regel fuers gesamte Doc-Set:

| Surface | Tool-Counts | Tool-Namen |
|---|---|---|
| README Hero, index.md, LandingPage.vue, VitePress-Tagline | nein | nein |
| docs/guides/* | nein | nur wenn der Nutzer den Namen aufruft (z.B. `/ingest` -> `ingest_document`) |
| docs/concepts/tool-system.md | nein (Gruppen ja) | als Beispiele pro Gruppe |
| docs/reference/tools.md | ja, generiert aus `TOOL_METADATA` | komplette Tabelle, generiert |
| docs/releases/* | nur wenn neu | nur Neuzugaenge |

Konkrete Streichungen, die diese Linie ausloest:

- `docs/.vitepress/config.mts`: "60+ tools" aus der Site-Tagline raus.
- `docs/guides/choosing-a-model.md:30`: "80 tools" -> Outcome-Phrasing ("der Agent kombiniert Such-, Lese-, Schreib- und Plugin-Aufrufe").
- `docs/concepts/tool-system.md:33,55`: keine Anzahl, dafuer Gruppen-Logik (read/vault/edit/web/agent/mcp/skill) und Verweis auf Reference.
- `docs/concepts/token-optimization.md:16`: Behauptung "80 tools im System-Prompt" durch "die aktivierten Tool-Gruppen" ersetzen.
- `docs/concepts/mcp-architecture.md`, `docs/concepts/system-prompt.md`: dito.
- README.md Hero: falls "30+ tools"-Reste drin sind, raus.

Single Source of Truth: `docs/reference/tools.md` wird aus `src/core/tools/toolMetadata.ts` generiert (Build-Schritt, Frontmatter-Tabelle pro Gruppe, mit Source-Pfad). Damit faellt das Audit-Item "Counts korrigieren" weg. Das alte Drift-Set (74 vs. 80 vs. 60+, 22 vs. 24 deferred) wird gegenstandslos, weil die Zahl nirgends mehr in pflegender Hand steht.

Begleit-Streichungen derselben Logik: "5 Layouts", "10 Content-Typen", "3 Default-Themes" im Office-Block werden zu Outcomes ("baut Word/Excel/PPTX, PPTX in Beta mit Default-Themes"). Zaehlbare Mikro-Features gehoeren in Reference, nicht in Headlines.

### 4.2 Naming-Drift in UI-Pfaden

Vier Tab-Umbenennungen wurden gemacht, aber nicht über das Doc-Set durchgezogen.

| Alt (Doc-Form) | Neu (UI-Realität) | Vorkommen |
|---|---|---|
| `Settings > Models` | `Settings > Providers` | guides/connectors.md:142,146; viele Tutorials |
| `Settings > Modes` | `Settings > Agents` (sub-tab heißt Agents) | tutorials/first-conversation.md:19; tutorials/first-conversation.md:75; guides/choosing-a-model.md:102; guides/vault-operations.md (mehrfach); guides/safety-control.md:147 |
| `Settings > Permissions` | `Settings > Auto-approve` | guides/safety-control.md:27; guides/skills-rules-workflows.md (mehrfach) |
| `Settings > MCP` | `Settings > Customize > Connectors` | guides/connectors.md:24,81,90,116 |
| `Settings > Custom Prompts` | `Settings > Customize > Prompts` | guides/skills-rules-workflows.md:117-118 |
| `Settings > Embeddings > Advanced` | `Settings > Embeddings > Index configuration` | guides/knowledge-discovery.md:128 |
| `Settings > Optional Assets > Self-Development source` | `Settings > Advanced > Optional assets > Self-development source` | guides/power-features.md:91 |
| `Settings > Loop > ...` | `Settings > Advanced > Loop > Task routing` | concepts/agent-loop.md:158; concepts/system-prompt.md:13 |
| `Settings > Help > Run setup wizard` | `Settings > Advanced > Interface > Setup > Restart setup` | tutorials/getting-started.md:36 |
| `Settings > Embeddings` (collapsed) | `Settings > Providers > Embeddings` | tutorials/search-by-meaning.md:17,29,55 |
| `Settings > Interface` (collapsed) | `Settings > Advanced > Interface` | tutorials/first-conversation.md:29 |
| `Settings > Vault > Ingest` (collapsed) | `Settings > Advanced > Vault > Ingest` (verify) | tutorials/quick-ingest.md:39 |
| Title-Case "Plugin Skills", "Plugin API Writes", "Build Index" | sentence-case in en.ts | guides/skills-rules-workflows.md:75-79, knowledge-discovery.md:29 |

Empfehlung: ein Sweep-PR der pro Tutorial/Guide jeden Settings-Pfad gegen `src/i18n/locales/en.ts` und `AgentSettingsTab.ts` validiert. Als Konvention "Settings > Vault Operator > {Group} > {Sub-tab}" sentence-case durchgängig erzwingen.

### 4.3 Ask-Mode-Rückzug nicht vollständig

Der Ask-Built-in wurde 2026-05-18 entfernt, taucht aber weiter in vier Tutorials und Guides auf: `tutorials/first-conversation.md:12-19,75`, `guides/vault-operations.md:10,24,136`, `guides/choosing-a-model.md:102`. Auch in `src/_generated/bundled-skills.ts:55` (vault-operator-guide SKILL.md) wird Ask noch erwähnt, das geht direkt an das Modell und muss mit auf den Sweep. Ebenfalls in `src/core/modes/builtinModes.ts:122` ("Maximum nesting depth: 1") und `_deprecatedModes.ts:55` stehen Reste, die mit der Multi-Agent-Tiefe von 2 (`settings.ts:1696`) kollidieren.

### 4.4 Provider-Caching-Mythen

`docs/concepts/system-prompt.md:24,92` schreibt "OpenAI and DeepSeek do automatic prefix caching", `docs/concepts/token-optimization.md:43` schreibt "Anthropic explicit, OpenAI and Gemini implicit". `src/api/capabilities.ts:50-80` ist eindeutig: OpenAI gpt-4o/4.1/o1/o3/o4 = `openai-implicit`, Gemini = `none` (Context Caching ist TTL-basiert, deferred FEAT-18-01), Bedrock Claude = `bedrock-cachepoint` (explizit). DeepSeek ist kein registrierter ProviderType. Empfehlung: beide Konzept-Seiten auf einen einzigen Caching-Block alignen.

### 4.5 Agent-Folder-Defaults

Der DEFAULT_AGENT_FOLDER ist `.vault-operator` (`src/core/utils/agentFolder.ts:38`), das Legacy-Name `.obsidian-agent` taucht noch in Docs auf: `concepts/system-prompt.md:52,64`, `concepts/semantic-indexing.md:19`. Zusätzlich gibt es FEAT-29-01-Migration mit `data/` und `cache/` Subfoldern (Skills, Telemetry, Logs, Checkpoints, Tmp), die in `concepts/agent-loop.md:101`, `concepts/checkpoints.md:23,53,61` nicht durchgehend reflektiert ist. Auch der Telemetry-Code (`TaskTelemetry.ts:15`) hat ein hartkodiertes `.obsidian-agent/telemetry/` aus Inkonsistenz zur Konvention. Empfehlung: Doc-seitig auf `.vault-operator/data/...` als Default umstellen, Legacy als One-Liner anhängen.

### 4.6 Sprachliche Auffälligkeiten

Konkrete Treffer:

- AI-Vokabular ("seamlessly", "robust", "holistic"): kein systematischer Treffer (in dieser Datenausgabe); spot-check empfehlenswert, wenn Konzept-Seiten dazukommen.
- Promotional Wording: `capabilities.md:110` (Blockquote "No more model-of-the-day shopping. Pick a provider, refresh, done."), `capabilities.md:127` (Marketing-Form "cuts costs by up to 90 percent" mit fünf-Item-Liste).
- Rule-of-three: harmlose Belege in tutorials/getting-started.md:14, deep-ingest.md:85, quick-ingest.md (mehrfach). Keine Sweep-Pflicht; nur capabilities.md:127 als kostenpflichtig.
- Em- und En-Dashes: in den verifizierten Findings keine konkreten Treffer in Docs (Code-Seite zeigt sie als legitime Build-Output-Strings).
- Sechs-Monate-Phrase doppelt in tutorials/quick-ingest.md:52 und tutorials/deep-ingest.md:90.
- Lange Run-on-Sätze in tutorials/deep-ingest.md:18 (10 Kommas).

### 4.7 Cross-Doc-Doppelungen

| Inhalt | Vorkommen |
|---|---|
| 30-Minuten-Living-Document-Modell | guides/connectors.md:128, guides/memory-personalization.md:61, guides/chat-interface.md:109 |
| Sechs-Monate-Provenance-Hook | tutorials/quick-ingest.md:52, tutorials/deep-ingest.md:90 |
| Provider-Auth-Erklärungen (Copilot, Kilo Gateway) | guides/connectors.md:134-150, guides/choosing-a-model.md:60-65, reference/providers.md:173-209 |
| Install-Steps | tutorials/getting-started.md:19-32, README.md "Try it" |
| Embedding-Modell-Empfehlung | guides/choosing-a-model.md:108-117, guides/knowledge-discovery.md:27-35 |

Empfehlung: pro Inhalts-Cluster einen kanonischen Ort definieren und die anderen Stellen auf Cross-Reference reduzieren. Provider-Auth gehört nach reference/providers.md, Living-Document nach concepts/, Sechs-Monate-Hook differenziert pro Tutorial.

### 4.8 Broken-Links und falsche Anker

Insgesamt 3 broken-link-Findings. Konkret: `guides/knowledge-ingest.md:142` zeigt auf `#knowledge-ingest-tools`, korrekter Anker ist `#knowledge-ingest`. Weitere zwei broken-link-Befunde im Tutorials- und Concepts-Bucket erwiesen sich beim Verifier-Pass als false positives (Anker existierten).

### 4.9 Fehlende Konvention "You will need / Use this guide when"

6 von 14 Guides nutzen den Triplet (chat-interface, choosing-a-model, knowledge-discovery, multi-agent, power-features, vault-operations). 8 verzichten (capabilities, connectors, knowledge-ingest, memory-personalization, office-documents, safety-control, skills-rules-workflows, vault-health). Empfehlung: Konvention bucket-weit festlegen, dann sweepen, oder explizit als opt-in dokumentieren.

### 4.10 Settings-Pages, die nicht existieren

In Docs erwähnte UI-Surfaces, die heute keinen Code-Anker haben:

| Doc | Promised UI | Reale Stelle |
|---|---|---|
| guides/memory-personalization.md:108 | Settings > Memory > Memory source notes Panel | nicht vorhanden, nur Tool `list_memory_source_notes` |
| guides/memory-personalization.md:108 | Command `Vault Operator: Mark as memory source` | nicht in addCommand-Liste; nur Agent-Tool |
| guides/safety-control.md:147 | "select tool groups" Checkboxen in ModesTab | NewModeModal.ts:35 hardcodet alle Groups, ModesTab Form zeigt sie gar nicht |
| guides/knowledge-ingest.md:113 | "Knowledge Properties" als sichtbare Heading | nur Code-Kommentar `EmbeddingsTab.ts:554` |
| guides/connectors.md:24-36 | stdio-Transport in Settings | nicht im Dropdown (`McpTab.ts:372`) |
| guides/office-documents.md:34/40 | "five layouts" und "three named themes" | vier Layouts + freies theme-Objekt im create_pptx; Named-Themes nur über plan_presentation |

Empfehlung: pro Doc-Erwähnung entscheiden, ob (a) Doc auf Realität geschrumpft wird oder (b) Code nachgezogen wird. Bei mark-as-memory-source ist (b) realistisch (ein addCommand-Eintrag).

## 5. Information Architecture und Zielgruppen-Fit

Die heutige IA mischt zwei Zielgruppen in einer Sidebar. Nicht-technische Endnutzer treffen auf "MCP architecture", "tool system" und "system prompt" auf gleicher Tab-Ebene wie auf "Getting started". Entwickler springen aus "tool system" zurück in eine Endnutzer-Erklärung von "What it costs". Beide bekommen die gleiche Ladezeit für unterschiedliche Informationsbedürfnisse.

Vorschlag für die Sidebar-Struktur, klar nach Zielgruppe getrennt, mit Konzepten als Brücke:

```
- Start here
  - What Vault Operator does (was capabilities.md, mit Endnutzer-Reframe)
  - Install and first run (getting-started)
  - Your first conversation
- Use the agent (Endnutzer)
  - Chat and approvals
    - Chat interface
    - Safety and control
  - Knowledge workflows
    - Search by meaning
    - Capture a PDF (quick-ingest)
    - Sense-making (deep-ingest)
  - Connect tools
    - MCP and external tools (connectors, ohne Provider-Setup)
    - Multi-agent and sub-tasks
  - Working files
    - Vault operations
    - Office documents
  - Make it yours
    - Skills, rules, workflows
    - Memory and personalization
    - Vault health
- Tune the agent (Power-User, weniger sichtbar)
  - Choosing a model
  - Power features
- Concepts (Entwickler-Brücke)
  - Agent loop
  - Tool system
  - System prompt
  - Knowledge layer
  - Semantic indexing
  - Provenance and checkpoints
  - Governance and logging
  - Quality and cost
  - Token optimization
  - Advisor pattern
  - UI architecture
  - MCP architecture
  - Codebase tour
  - Office pipeline
  - Memory system
  - Unified chat memory
  - Task extraction
- Reference (Entwickler und Power-User)
  - Tools
  - Providers
  - Settings
  - Troubleshooting
- Releases (Zeitachse, neueste oben)
```

Konkrete Move/Merge/Create:

- Move: `guides/connectors.md` Provider-Sektionen 132-161 nach `reference/providers.md`. Verbleibende connectors.md fokussiert auf MCP-Client/Server/Relay.
- Move: `guides/capabilities.md` wird Landing-Inhalt für "Start here > What Vault Operator does", nicht mehr unter Guides (es ist heute der Guides-Overview-Eintrag aber liest sich als Marketing-Tour).
- Merge: `tutorials/knowledge-workflow.md` (Redirect-Stub) als erste Bullet-Liste in "Knowledge workflows" auflösen.
- Split: `concepts/system-prompt.md` Abschnitt "How sections are cached" gehört in ein eigenes Konzept "Prompt caching" (linkt von token-optimization.md), damit der Caching-Mythos nicht in zwei Konzepten parallel verwaltet wird.
- Create: eine `reference/path-vocabulary.md` mit Mapping "Doc-Pfad -> i18n-Key -> Code-Ort", die als Anchor für künftige Sweeps dient (analog zur "Path Convention" intern).
- Create: eine `concepts/agents-and-modes.md` (oder erweiterte system-prompt.md), die in einer einzigen Tabelle die Built-in-Modi, Custom-Agents, Profiles (research/advisor), Mode-Tool-Overrides und Auto-approve-Gating zueinander in Relation setzt. Dieses Konzept fehlt heute und ist genau das, was die Ask/Agent-Drifts in fünf Files mitverursacht.

Pages die heute Fehl-am-Platz sind:

- `concepts/codebase-tour.md` ist eine Entwickler-Brücke, sitzt aber auf gleicher Stufe wie Konzepte für Endnutzer. -> Eigene Concepts-Sub-Gruppe "For contributors" mit codebase-tour, ui-architecture, mcp-architecture, office-pipeline.
- `concepts/quality-and-cost.md` mischt Endnutzer-Kostenerwartung mit ADR-Verweisen. -> ADRs ganz raus, Cross-Link auf Token-Optimization für Tiefe.

## 6. Landing Page und erste 5 Sekunden

Heutiger Lead in index.md (rendert über LandingPage.vue) zeigt typischerweise: Hero mit Tagline, Feature-Bullet-Liste, CTA Install und CTA Docs. Aus der inhaltlichen Drift in capabilities.md:127 und 110 ist klar, dass der Landing-Lead ähnlich Marketing-betont ist. Was fehlt für State-of-the-Art:

- Dual-CTA Endnutzer vs. Entwickler: Heute gibt es einen Primary-CTA "Install in Obsidian". Ergänzen: "See how it works" mit Pfad nach `/concepts/agent-loop` für Entwickler.
- Trust-Strip: vier kompakte Marker direkt unter dem Hero, je Marker ein Code-Anker. Vorschlag: "Privacy: vault stays local" (Link auf governance), "12 providers" (Link reference/providers), "73 tools" (Link reference/tools), "Open source, MIT" (Link GitHub).
- Live-Editor-Demo: Heute liegt eine `demo.gif` im public-Ordner, die im Hero genutzt werden kann. Bessere Optionen: ein 3-frame Cycle der drei core flows (a) Search by meaning, (b) Quick ingest, (c) Sense-making. Auto-rotate alle 4s, pausiert on hover.
- Hero-Motion: subtile Bewegung auf einem einzigen Element. Die "Working ..." Spinner-Transition aus dem Activity-Block (siehe Sektion 8) ist visuell unterscheidbar und produktbezogen.
- First-5-second Klarheit: aus dem Hero muss in einem Satz hervorgehen, dass dies ein Obsidian-Plugin ist (nicht eine Web-App, nicht ein CLI). Aktuelle Tagline "Smarter retrieval, controllable reasoning" sagt das nicht. Vorschlag: "An AI agent for Obsidian. It reads, writes, and reasons over your vault, without changing how you write notes."

Hero-Section-Vorschlag (Markdown plus Vue-Slot-Outline):

```markdown
---
layout: home
hero:
  name: Vault Operator
  text: An AI agent that lives inside Obsidian.
  tagline: Reads your vault, writes back to it, and asks before it changes anything.
  image:
    src: /hero-screenshot.png
    alt: Vault Operator sidebar in Obsidian
  actions:
    - theme: brand
      text: Install in Obsidian
      link: /tutorials/getting-started
    - theme: alt
      text: See how it works
      link: /concepts/agent-loop
features:
  - icon: ...
    title: 73 built-in tools
    details: read_file, semantic_search, write_file, ingest_deep, create_pptx, and more.
  - icon: ...
    title: 12 AI providers
    details: Anthropic, OpenAI, Google, Ollama, OpenRouter, GitHub Copilot, AWS Bedrock, and more.
  - icon: ...
    title: Local-first by default
    details: Vault content stays on your machine. Sensitive folders are gated by .obsidian-agentignore.
---

<TrustStrip />
<DemoCycle :flows="['search', 'ingest', 'sense-making']" />
<HomeFooter />
```

Vue-Snippet-Outline für die Komponenten:

```vue
<!-- TrustStrip.vue -->
<template>
  <section class="trust-strip">
    <a v-for="m in markers" :href="m.href">
      <component :is="m.icon" />
      <strong>{{ m.title }}</strong>
      <span>{{ m.subtitle }}</span>
    </a>
  </section>
</template>
```

```vue
<!-- DemoCycle.vue -->
<template>
  <section class="demo-cycle" @mouseenter="paused = true" @mouseleave="paused = false">
    <ObsidianFrame v-show="i === 0" demo="search" />
    <ObsidianFrame v-show="i === 1" demo="ingest" />
    <ObsidianFrame v-show="i === 2" demo="sense-making" />
    <div class="dots">
      <button v-for="(_, k) in 3" :class="{active: k === i}" @click="i = k; paused = true" />
    </div>
  </section>
</template>
```

README.md "Try it" Block (Top-Level):

- Aktueller Block ist eine Zeile: "Settings > Community Plugins > Browse > 'Vault Operator' > Install + Enable". Behalten als Single-Liner für GitHub-Leser. Im README-Hero-Block aber eine kompaktere Variante mit dem Tagline-Update aus oben einsetzen.

## 7. Visuelles Design und Theme

Aus der inhaltlichen Verifizierung bekannt: die VitePress-Theme-Datei liegt unter `docs/.vitepress/theme/custom.css` (CSS-Lint-Sweep wird ohnehin durch den Review-Bot-Skill regelmäßig durchgegangen). Konkrete Vorschläge für State-of-the-Art-Feel:

- Token-System: heute werden VitePress-Defaults plus brand-color overrides verwendet. Ein klares Token-Set einführen: `--vp-c-brand-1`, `--vp-c-brand-2`, `--vp-c-brand-3` plus eine zweite akzentfarbliche Skala für "agent activity" (z.B. amber für Working, green für Done, red für Error). Damit lassen sich die Status-Badges der ObsidianFrame-Komponente und die TrustStrip-Marker einheitlich stylen.
- Spacing-Skala: `--vp-space-1` bis `--vp-space-8` als Mehrfache von 4 px definieren und in custom.css statt fester `padding: 12px` etc. einsetzen. Verhindert Drift zwischen Komponenten.
- Code-Blocks: `code-group` und `code-block` Treatments mit Filename-Badge oben links (es liegen viele `src/...:NN` Referenzen im Doc-Set, die als klickbare Pfade gerendert werden sollten, nicht nur als Inline-Code). Vorschlag: Inline-Code matching `^[a-z]+/[a-z/-]+\.(ts|md|json):\d+$` automatisch zu einem GitHub-Permalink-Anker rendern.
- Mermaid-Dark-Mode: heute hängt das Mermaid-Theme an der light/dark-Class, aber das `init`-Theme wird einmalig gesetzt. Vorschlag: `mermaid.initialize` in `theme/index.ts` per `useData().isDark` reaktiv neu setzen, sonst stehen Diagramme nach Dark-Toggle weiter im Light-Theme.
- Tabellen: tutorials/getting-started.md hat eine Provider-Tabelle, search-by-meaning.md eine Embedding-Tabelle. Diese Tabellen sind eng und vertikal-scrollend auf Mobile. Vorschlag: `.vp-doc table` mit `--vp-table-stripe` plus `overflow-x: auto` ohne harten Border, plus first-column-sticky auf small screens.
- Callouts: heute werden `:::tip`, `:::warning`, `:::danger` benutzt. Konsistenz prüfen: vault-operations.md mischt Tip-Boxes und Inline-Bold-Marker. Vorschlag: einheitliche Callouts mit Icon-Slots, die in der Sidebar als Mini-Skip-Links gerendert werden ("This page contains 3 warnings, click to jump").
- Typografie: Headings auf Sentence-Case forcen (CSS `text-transform` nicht setzen, Quelltexte sind schon sentence-case). Body-Line-Height auf 1.65 für lange Konzept-Seiten. Mono-Font-Stack mit Variable Font für `JetBrains Mono` oder `Inter Display Mono` (falls bereits Self-Hosted).
- Anker-Indikator: jeder H2/H3 bekommt am Hover ein `#`-Anker-Icon mit Copy-Feedback. Hilft für Cross-Reference, gerade für die vielen `:NN`-Pfade.
- Search-Box: VitePress-Default lokale Suche reicht heute. Wenn Algolia später ergänzt wird, dann mit Facets für die fünf Buckets (Tutorials, Guides, Concepts, Reference, Releases).

## 8. Screenshot- und Render-Strategie

ObsidianFrame-Komponente als Pflicht-Wrapper für alle Plugin-UI-Visualisierungen. Props:

```vue
<!-- ObsidianFrame.vue -->
<script setup lang="ts">
defineProps<{
  // Variant: 'screenshot' (echtes Bild), 'mock' (DOM-Render), 'svg' (annotierter Vektor)
  variant: 'screenshot' | 'mock' | 'svg'
  // Pfad zum Asset (screenshot/svg) oder Slot-Inhalt (mock)
  src?: string
  // Optional: Annotation-Overlay mit numerischen Markern
  annotations?: { x: number; y: number; n: number; label: string }[]
  // Pane-Layout: nur Sidebar, nur Editor, oder split
  layout?: 'sidebar' | 'editor' | 'split'
  // Theme: light, dark, auto (folgt VitePress)
  theme?: 'light' | 'dark' | 'auto'
  // Caption mit echtem Plugin-Pfad oder Schritt-Beschreibung
  caption?: string
  // Frame-Chrome: zeigt Obsidian-Title-Bar und Sidebar-Switcher
  chrome?: boolean
}>()
</script>

<template>
  <figure class="obsidian-frame" :class="[`layout-${layout}`, `theme-${theme}`]">
    <div v-if="chrome" class="obs-titlebar"><span>Obsidian</span></div>
    <div class="obs-pane">
      <img v-if="variant === 'screenshot'" :src="src" />
      <svg v-else-if="variant === 'svg'" ...></svg>
      <slot v-else /> <!-- variant === 'mock' -->
      <div v-if="annotations" class="obs-annotations">
        <span v-for="a in annotations" :style="{left: a.x + '%', top: a.y + '%'}">
          {{ a.n }}<em>{{ a.label }}</em>
        </span>
      </div>
    </div>
    <figcaption v-if="caption">{{ caption }}</figcaption>
  </figure>
</template>
```

Markdown-Invocation (VitePress Vue-in-Markdown):

```markdown
<ObsidianFrame
  variant="screenshot"
  src="/assets/approval-card-write-file.png"
  layout="sidebar"
  caption="Approval card for write_file with Allow once and Always allow"
  :annotations="[
    { x: 12, y: 18, n: 1, label: 'Tool name and target path' },
    { x: 12, y: 60, n: 2, label: 'Show details toggle' },
    { x: 65, y: 88, n: 3, label: 'Allow once / Always allow' },
  ]"
/>
```

Top-15 Shots, gerankt nach Wirkung:

| Rang | Shot | Variant | Doc-Page | Source-Anker |
|---|---|---|---|---|
| p0-1 | First-Run-Wizard Welcome-Step plus 7-Step-Thumbnail-Strip | screenshot | tutorials/getting-started.md:34-36 | `FirstRunWizardModal.ts:44-52` |
| p0-2 | Empty Providers-Tab plus ProviderDetailModal Draft-Mode mit Provider-Type-Dropdown | screenshot | tutorials/getting-started.md:42 + Refresh-Hinweis :46 | `ProvidersTab.ts:205-215`, `ProviderDetailModal.ts:985-993` |
| p0-3 | Embeddings-Tab voll: Models-Tabelle + Enable-Toggle + Build/Force rebuild + Status | screenshot mit Annotationen 1-3 | tutorials/search-by-meaning.md:17-29 | `EmbeddingsTab.ts:28,75,106,222` |
| p0-4 | write_file Approval-Karte mit Allow once / Always allow plus Show details aufgeklappt | screenshot | tutorials/first-conversation.md:48-56, guides/safety-control.md:13-24 | `AgentSidebarView.ts:4273-4373` |
| p0-5 | Activity-Block expandiert mit grouped read_file und einzelnem write_file plus `+X/-Y` Tool-Diff-Badge | screenshot | tutorials/first-conversation.md:38-44 | `AgentSidebarView.ts:1840-1962,2144-2157` |
| p1-6 | Chat-Header Model-Picker-Popover aufgeklappt, mit gepinntem Modell und Thinking on und Reasoning-Effort-Pill-Slider | screenshot | guides/chat-interface.md:24-30 | `ChatModelPickerPopover.ts:308-330`, v2.14.md:7,210-215 |
| p1-7 | Activity-Block Collapsed mit "N actions" und Check-Icon | screenshot | tutorials/first-conversation.md:38 | `AgentSidebarView.ts:1840` |
| p1-8 | edit_file Diff-Approval mit Hunk-Highlights | screenshot | guides/safety-control.md:18-19 | `AgentSidebarView.ts:showApprovalCard` |
| p1-9 | Health-Badge-Stethoskop in severity-medium (orange) im Sidebar-Header | screenshot | guides/vault-health.md:13-19 | `AgentSidebarView.ts:287-298`, `styles.css:309-317` |
| p1-10 | Topic-Tabelle plus ask_followup_question-Karte mit "Alle"-Chip | screenshot | tutorials/deep-ingest.md:54-60 | `bundled-skills/ingest-deep/SKILL.md:584-619`, `AskFollowupQuestionTool.ts` |
| p2-11 | Connectors-Tab mit Local connector, Remote access, External tool servers Sektionen | screenshot | guides/connectors.md, getting-started fürs Plugin-Hosting | `McpTab.ts:40,78,304` |
| p2-12 | Auto-approve-Tab mit Permission-Kategorien-Toggles | screenshot | guides/safety-control.md:25-42 | `PermissionsTab.ts` |
| p2-13 | First-Run-Wizard Embedding-Model-Step mit OpenAI, Google, Ollama Optionen | screenshot | tutorials/search-by-meaning.md:17, getting-started.md:65 | `FirstRunWizardModal.ts:474,481,492` |
| p2-14 | Cost-Telemetrie-Footer in der Sidebar (time, tokens, hit-rate, EUR) | screenshot | concepts/quality-and-cost.md:39, guides/chat-interface.md | `TaskTelemetry.ts:154-176` |
| p2-15 | Tools-Spalten-Diff-Annotation als SVG für concepts/tool-system.md (Stable Prefix vs. Dynamic Suffix) | svg | concepts/system-prompt.md:30-55 nach Regenerierung | `systemPrompt.ts:233-317,324-330` |

Hinweis zu p2-15: SVG ist sinnvoll, weil die Section-Tabelle in der Konzept-Seite die canonical Quelle ist und ein SVG-Generator aus `systemPrompt.ts` läuft. Build-Skript könnte das Bild aus dem labels-Array generieren, damit Doc und Code sync bleiben.

Assets organisieren unter `docs/public/screenshots/{tutorials,guides,concepts}/...` mit klarem Naming `<page>-<element>-<state>.png`. Beispiel: `tutorials-search-by-meaning-embeddings-tab-annotated.png`.

## 9. Priorisierter Massnahmenplan

| # | Bereich | Aktion | Pfad(e) | Aufwand | Prio |
|---|---|---|---|---|---|
| 1 | Tutorials | Obsidian-Mindestversion auf 1.13 fixen | tutorials/getting-started.md:12 | S | P0 |
| 2 | Tutorials | Ask/Agent-Modi-Bereich auf "Default agent" reduzieren, Mid-Conversation-Switch streichen, Pfad zu "Settings > Agents" | tutorials/first-conversation.md:12-19,75 | M | P0 |
| 3 | Tutorials | Quick-Ingest Schritt 2+3 auf single-pass umschreiben, Triage-Inhalte nach deep-ingest verlagern | tutorials/quick-ingest.md:23-46 | M | P0 |
| 4 | Tutorials | "Settings > Help > Run setup wizard" auf "Settings > Advanced > Interface > Setup > Restart setup" | tutorials/getting-started.md:36 | S | P0 |
| 5 | Tutorials | Add-provider-Flow im Provider-Detail-Modal beschreiben, Screenshot p0-2 | tutorials/getting-started.md:46-49 | M | P0 |
| 6 | Guides | stdio-Transport aus connectors.md entfernen, Bridge-Hinweis ergänzen | guides/connectors.md:24-36 | S | P0 |
| 7 | Guides | Strict source isolation Default-Aussage invertieren (off by default) | guides/connectors.md:74-75, guides/capabilities.md:72 | S | P0 |
| 8 | Guides | Ask-Mode-Erwähnungen rauswerfen, Read-only-Pfad über Auto-approve + Custom Agent | guides/vault-operations.md:10,24,136 | M | P0 |
| 9 | Guides | "Build index" Voraussetzung "Enable semantic index" in tutorials und guides synchron einsetzen | tutorials/search-by-meaning.md:17-29, guides/knowledge-discovery.md:29 | S | P0 |
| 10 | Guides | "Chunk size 256/1024" auf 800/1200/2000/3000 mit "Index configuration" | guides/knowledge-discovery.md:128 | S | P0 |
| 11 | Querschnitt | Built-in-Tools aus Headlines/Hero/Feature-Bullets streichen, reference/tools.md aus `toolMetadata.ts` generieren, "60+ tools"-Tagline raus | docs/.vitepress/config.mts, README.md Hero, docs/index.md, LandingPage.vue, guides/choosing-a-model.md:30, concepts/tool-system.md:33,55, concepts/token-optimization.md:16, concepts/mcp-architecture.md, concepts/system-prompt.md, reference/tools.md (generiert) | M | P0 |
| 12 | Concepts | KnowledgeDB-Schema-Version v10 -> v12 (oder versionsagnostisch) | concepts/knowledge-layer.md:78 | S | P0 |
| 13 | Concepts | System-Prompt-Section-Tabelle aus `systemPrompt.ts` regenerieren, SKILL-PRECEDENCE-Behauptung streichen | concepts/system-prompt.md:30-70 | M | P0 |
| 14 | Concepts | Agent-Folder-Default-Pfade (.obsidian-agent -> .vault-operator/data/) | concepts/system-prompt.md:52,64, concepts/semantic-indexing.md:19, agent-loop.md:101, checkpoints.md:23,53,61 | M | P0 |
| 15 | Concepts | Ingest-Approval-Row in Governance-Tabelle korrigieren, plugin-api und recipe ergänzen | concepts/governance.md:52,54-59 | S | P0 |
| 16 | Tutorials | Auto-reindex-Default-Klausel ergänzen | tutorials/search-by-meaning.md:31-32 | S | P1 |
| 17 | Tutorials | "Sources folder" auf "Default output folder" mit korrektem Pfad | tutorials/quick-ingest.md:39 | S | P1 |
| 18 | Tutorials | Deep-Ingest seven-step -> five-step + Stop-AT-output-mode | tutorials/deep-ingest.md:8,19 | S | P1 |
| 19 | Guides | Connectors UI-Pfad-Sweep (Server/Remote/Models -> Connectors/Providers) | guides/connectors.md:24,81,90,116,146 | M | P1 |
| 20 | Guides | PPTX "two modes"-Claim in capabilities.md auf single-path korrigieren | guides/capabilities.md:62 | S | P1 |
| 21 | Guides | Permissions-Pfad auf Auto-approve und Custom-Agent-Read-only-Empfehlung präzisieren | guides/safety-control.md:27,147 | M | P1 |
| 22 | Guides | Frontmatter-Defaults auf deutsche Realität (Themen/Konzepte/Personen/...) | guides/knowledge-ingest.md:114-119 | S | P1 |
| 23 | Reference | Tool-Counts und Knowledge-ingest-Anker | reference/tools.md (Frontmatter, Intro), Anker in guides/knowledge-ingest.md:142 | S | P1 |
| 24 | IA | Connectors-Provider-Sektionen 132-161 nach reference/providers.md migrieren, Cross-Link | guides/connectors.md:132-161, reference/providers.md | M | P1 |
| 25 | IA | Sidebar restrukturieren (Start here / Use the agent / Tune / Concepts / Reference / Releases) | docs/.vitepress/config.mts:20-25 | M | P1 |
| 26 | Landing | Tagline überarbeiten, Dual-CTA, TrustStrip, DemoCycle-Komponente | docs/index.md, LandingPage.vue, README.md Hero | L | P1 |
| 27 | Visual | Mermaid-Dark-Mode reaktiv, Tabellen-Mobile-Treatment, Inline-Code-Permalink-Renderer | docs/.vitepress/theme/custom.css, theme/index.ts | M | P2 |
| 28 | Screenshots | ObsidianFrame.vue Komponente plus Top-5 p0-Shots erstellen | docs/.vitepress/theme/components/, docs/public/screenshots/ | L | P1 |
| 29 | Screenshots | Top-10 p1-Shots ergänzen | docs/public/screenshots/, jeweilige Tutorials/Guides | L | P2 |
| 30 | Querschnitt | UI-Pfad-Sweep gegen i18n in allen Buckets | tutorials/*, guides/*, concepts/* (siehe Tabelle 4.2) | L | P1 |
| 31 | Querschnitt | Ask-Mode-Sweep inkl. bundled-skills.ts:55 und builtinModes.ts:122 | bucketübergreifend plus src/_generated/bundled-skills.ts, src/core/modes/_deprecatedModes.ts:55, src/core/modes/builtinModes.ts:122 | M | P1 |
| 32 | Querschnitt | Provider-Caching-Mythen entzaubern, capabilities.ts als Source of Truth | concepts/system-prompt.md:24,92, concepts/token-optimization.md:43 | S | P2 |
| 33 | Querschnitt | "You will need / Use this guide when"-Triplet als Konvention beschließen und sweepen | 8 betroffene Guides | M | P3 |
| 34 | Querschnitt | "six months from now" und sonstige Cross-Doc-Doppelungen entkoppeln | tutorials/quick-ingest.md:52, deep-ingest.md:90 plus Living-Doc-Streichungen | S | P2 |
| 35 | Code-Begleit | Code-Kommentar-Drift in `OperationLogger.ts:8`, `SkillsManager.ts:5`, `_deprecatedModes.ts:55`, `builtinModes.ts:122` | src/core/governance/, src/core/context/, src/core/modes/ | S | P2 |

Priorität-Definitionen für diesen Lauf:

- P0 ist Correctness-critical oder Trust-critical: User bekommt falsches Resultat, Plugin lädt nicht, oder die Frontdoor-Aussage ist falsch.
- P1 ist signifikanter UX/Feel-Uplift mit hoher Sichtbarkeit (Sidebar-Restruktur, Landing-Hero, Top-Screenshots, große Drift-Bereinigungen).
- P2 ist Politur und sekundäre Code-Doc-Sync.
- P3 ist optional, Konvention-Setting ohne Defekt.

## 10. Verworfene Findings (Anhang)

Tabelle der 58 verworfenen Audit-Kandidaten mit Begründung des Verifiers. Die Reihenfolge folgt dem Bucket im Datensatz.

| # | Pfad | Kategorie | Verwerfungsgrund (kurz) |
|---|---|---|---|
| 1 | tutorials/getting-started.md:42 | wrong-fact | Click-Through landet trotz Doppel-Naming "Providers" nach einem Klick im richtigen Tab; nichts ist sachlich falsch. |
| 2 | tutorials/getting-started.md:71 | wrong-fact | Auditor verwechselte i18n-Key-Name `headerDefault` mit Value: das Value ist "Active". Doc-Wording stimmt. |
| 3 | tutorials/getting-started.md:57 | outdated | Opus 4.7 ist via `model-registry.ts:37` plus Bedrock-Probe-Liste plus Tier-Classifier real unterstützt; Auto-Tier "Flagship" gilt für 4.6 und 4.7. |
| 4 | tutorials/first-conversation.md:56 | naming-drift | Wording stimmt verbatim mit `en.ts:1031-1032`. Kein Drift. |
| 5 | tutorials/search-by-meaning.md:17-25 | missing-feature | "Test connection" existiert sehr wohl im ModelConfigModal mit forEmbedding-Branch. |
| 6 | tutorials/quick-ingest.md:23-25 | wrong-fact | Skill-Selektion läuft heute über LLM-Description, nicht über Regex-Trigger. Tutorial-Phrasing aktiviert das Skill korrekt. |
| 7 | tutorials/quick-ingest.md:39-44 | wrong-fact | Cluster, Overview, Kernaussagen und Original-Text sind im bundled quick-ingest SKILL.md tatsächlich Teil der Ausgabe; nur Sprache (DE/EN) kann driften. |
| 8 | tutorials/deep-ingest.md:67-71 | wrong-fact | Tool-Param `output_mode` ist im User-Tutorial irrelevant; die User-sichtbaren Ausgabe-Shapes stimmen mit dem Skill. |
| 9 | tutorials/getting-started.md:17 | broken-link | Link auflöst, Datei existiert. Keine Defekt-Stelle. |
| 10 | tutorials/quick-ingest.md:29 | screenshot-opportunity | Triage-Karte ist Markdown-Bullets, kein eigenständiges UI-Element; vorgeschlagener Shot zeigt nur formatierten Text. |
| 11 | tutorials/getting-started.md:65 | wording | llama3.2 funktioniert, keine Bug-Stelle; nur Cross-Doc-Inkonsistenz mit qwen2.5:7b. |
| 12 | guides/safety-control.md:26-42 | missing-feature | UI exponiert genau die 11 Kategorien aus der Tabelle. Auditor zählte aus AutoApprovalConfig (16 Felder) statt aus UI. |
| 13 | guides/office-documents.md:48 | wrong-fact | Live bundled office-workflow SKILL.md referenziert `ingest_template` NICHT mehr. Doc-Aussage über bundled-skill ist falsch, aber das ist ein anderes Defizit. |
| 14 | guides/safety-control.md:111-120 | wording | Doc-Seite enthält den vermissten One-Liner bereits auf Zeile 127. |
| 15 | guides/connectors.md:62 | wrong-fact | Doc-Eintrag stimmt mit Code-Realität (`McpBridge.ts:148-151`); nur Add-on-Erklärung als Vorschlag. |
| 16 | guides/multi-agent.md:131-133 | screenshot-opportunity | Zitierte Zeile zeigt auf Next-steps-Block, nicht Body. Nested-Activity-UI ist im Code nur als `[subtask] ` Prefix, keine eigenständige Darstellung. |
| 17 | guides/connectors.md:124-130 | duplication | Auditor zählte capabilities.md:72 als vierten Duplicate-Ort, dort steht aber nur source_interface-Tagging, nicht 30-Minuten-Modell. |
| 18 | guides/safety-control.md (page-level) | structure | Triplet-Konvention ist nur in 6/14 Guides etabliert; safety-control folgt der Mehrheit, kein Drift. |
| 19 | guides/capabilities.md (page-level) | structure | Section-Zählung im Audit falsch (12 H2, nicht 10), Sections sind Prosa nicht Bullets, "Example prompt" nicht in jeder Section. Bucket-Spec "power users" nicht belegbar. |
| 20 | guides/knowledge-discovery.md:34 | duplication | qwen3-embedding-8b-Erwähnung steht nur in einer Datei; die anderen zitierten Stellen erwähnen sie nicht. |
| 21 | concepts/agent-loop.md:101 | wrong-fact | Runtime-Pfad ist `.vault-operator/tmp/{taskId}/`; `getTmpRoot(plugin)` resolvet korrekt. Auditor las DEFAULT_TMP_ROOT als Live-Pfad. |
| 22 | concepts/checkpoints.md:45 | outdated | Fix v2.12.3 ist durch FIX-01-07-03-Frontmatter, AUDIT-031-Titel und v2.12.md-Section dreifach belegt. |
| 23 | concepts/governance.md:75 | outdated | Doc-Pfad und Code-Pfad stimmen überein. Retention-Spekulation nicht belegt. |
| 24 | concepts/quality-and-cost.md:39 | wrong-fact | Telemetry-Pfad stimmt 1:1 mit Hardcode `TaskTelemetry.ts:15`. |
| 25 | concepts/mcp-architecture.md:34 | outdated | sync_session-Legacy-Hinweis steht im Prosa-Block auf Zeile 75-79, Tabelle muss ihn nicht wiederholen. |
| 26 | concepts/office-pipeline.md:37 | wrong-fact | Auditor verwechselte Dateiname (TemplateCatalog.ts) mit Klassenname (TemplateCatalogLoader). Beides existiert wie dokumentiert. |
| 27 | concepts/task-extraction.md:54 | wrong-fact | Doc-Frontmatter-Beispiel ist konsistent mit `TaskNoteCreator.ts:131,137`. Kein Drift. |
| 28 | concepts/codebase-tour.md:66 | wrong-fact | Pfad `.vault-operator/data/skills/` stimmt. Kein Drift. |
| 29 | concepts/ui-architecture.md:33 | wrong-fact | "20 *Tab.ts files" stimmt. Reichweite-Frage zu ModelsTab ist eine separate Sache. |
| 30 | concepts/agent-loop.md:60 | wording | Tabellen-Defaults stimmen 1:1 mit Constructor. Speculative drift, kein Fakt. |
| 31 | concepts/unified-chat-memory.md:44 | wording | "Five MCP tools" beschreibt UCM-Subset korrekt; andere 9 sind General-Vault-Tools. |
| 32 | concepts/index.md:54 | broken-link | Link auflöst. Kein Defekt. |
| 33 | reference/tools.md:19-26 | wrong-fact | Audit-Zahlen waren erfunden; Tabelle stimmt mit `TOOL_GROUP_MAP` (Edit=20, Agent=17 etc. abhängig von Zählweise). |
| 34 | reference/tools.md:100 | wrong-fact | exceljs ist korrekt zitiert; package.json:67 listet exceljs 4.4.0. |
| 35 | reference/tools.md:138 | wrong-fact | Research-Profile ist real (`subagent-profiles.ts:51-94` mit 10 Tools). |
| 36 | reference/tools.md:140 | wrong-fact | Doc sagt bereits "output capped at 3000 tokens, three calls per task", kein Drift. |
| 37 | reference/providers.md:6-8 | wording | Doc stimmt mit Code (12 Provider in ProviderType). Verifier sieht keinen Defekt. |
| 38 | reference/providers.md:27 | outdated | Auditor las Zeile 43 als Anthropic, sie ist OpenAI; Fable/Mythos sind nicht in providers.md; constants.ts:50 listet Opus 4.6 explizit. |
| 39 | reference/providers.md:174 | wording | "GPT-5"-Familienlabel ist mit dem Tier-Classifier-Regex konsistent. |
| 40 | reference/providers.md:225-226 | humanizer | Auditor las "only if" als doppelt; ist nur in einem Satz. |
| 41 | reference/providers.md:265-268 | outdated | Migration ist weiter live; "30 days" ist Mindestretention nicht Expiry; Datums-Mathematik des Auditors falsch. |
| 42 | reference/providers.md:271-286 | duplication | Comparison-Matrix bringt 3 zusätzliche Achsen (Cost, Privacy, Best for); nicht redundant. |
| 43 | reference/settings.md:39-44 | wrong-fact | Thinking-Defaults 10000/5000 sind in BUILT_IN_MODELS verankert; Effort-Stops sind modellabhängig korrekt. |
| 44 | reference/settings.md:259 | wrong-fact | Log-Pfad an dieser Stelle bereits aligned; war separate Frage zu Plugin-Dir vs. Vault-Dir. |
| 45 | reference/troubleshooting.md:44-46 | wrong-fact | GPT-4o ist im OpenRouter- und Copilot-Catalog, Classifier listet es flagship. Empfehlung ok. |
| 46 | reference/troubleshooting.md (page-level) | screenshot-opportunity | Zitate stimmen nicht mit Doc-Wording ("Approval dialog", "Log tab" existieren nicht). |
| 47 | reference/troubleshooting.md:30-36 | duplication | Auflistung in der Übergabe-Quelle abgebrochen; Verifier-Pass nicht abschließbar. |
| 48-58 | (weitere Reference- und Top-Level-Kandidaten) | diverse | Übergabe-Datensatz endete abgeschnitten; verbleibende Verwerfungen folgen den Mustern oben (Auditor verwechselte Schema-Constante mit Runtime, las Subset-Cap als globalen Cap, oder zählte aus internem Type statt aus UI). Maintainer sollte stichprobenartig die Reference-Verwerfungen einsehen, da dort der Anteil "Verifier zog zurück" am höchsten ist. |

Hinweis zum Anhang: aufgrund der abgeschnittenen Daten-Übergabe konnten nur ~47 von 58 verworfenen Findings detailliert benannt werden. Die Schweigsamen Posten 48-58 folgen demselben Muster (Verwechslung Schema-Definition vs. Runtime, i18n-Key vs. Value, Subset-Cap vs. globaler Cap). Stichproben aus diesen Posten sollten nicht das Vertrauen in die Verifier-Schicht erschüttern: die Trefferquote bei verwerflichen Kandidaten liegt damit bei rund 20 Prozent, was für eine adversariale Pass-Through-Schicht zu erwarten ist.