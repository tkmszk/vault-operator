# Epic: Workflow-Builder und Settings-Vereinfachung

> **Epic ID**: EPIC-31
> **Feature Prefix**: FEAT-31-XX
> **Business Alignment**: User-getrieben aus Session-Diskussion 2026-05-20.
> **Scope**: MVP (Skeleton-Epic, volle Specs folgen nach EPIC-29 Welle 3)
> **depends-on**: [EPIC-29]
> **Note**: Skeleton (Phase: Candidates)

## How-Might-We

How might we wiederholbare Aufgaben als Sequenzen von Skills, Prompts und MCP-Aufrufen explizit modellieren, das Rules-Konzept abschaffen und das gesamte Customize-Konzept so vereinfachen, dass User intuitiv komponieren statt durch parallele UI-Bereiche springen muessen?

## Epic Hypothesis Statement

Fuer User die wiederkehrende Aufgaben automatisieren moechten (z.B. taegliche Newsletter-Aggregation, Wochenreport-Erstellung, Meeting-Vorbereitung), die heute zwischen Skills, Rules und Custom Prompts springen und keine klare Komposition-Mechanik haben, verspricht EPIC-31 zwei zusammenhaengende Aenderungen. Erstens: ein neuer Workflow-Builder im Stil von n8n, in dem User eine Sequenz aus Skill-Aufrufen, Prompts und MCP-Calls definieren und unter einem Namen speichern. Zweitens: das Rules-Konzept wird abgeschafft und in das bestehende "Agent > Additional Instructions"-Feld migriert, sowohl im UI als auch in der System-Prompt-Erzeugung. Damit sind die Konzepte auf drei reduziert (Skills, Workflows, Additional Instructions) statt heute fuenf (Skills, Workflows, Rules, Custom Prompts, Modes). Im Gegensatz zum heutigen Status, in dem Modes und Rules ungenutzte Legacy-Konzepte sind (laut Memory: "Mode-System nie genutzt, durch Skills ueberholt"), entsteht ein konsistentes Set von Bauwerkzeugen.

## Business Outcomes (messbar)

1. **Workflow-Adoption**: Mindestens 30% der aktiven User haben mindestens einen Workflow definiert nach 3 Monaten.
2. **UI-Vereinfachung**: Anzahl Customize-Settings-Tabs sinkt von 5 auf 3.
3. **Rules-Migration-Erfolg**: 100% der bestehenden Rules werden ohne User-Aktion in Additional Instructions migriert.

## Feature-Skizze

| Feature ID | Name | Skizze |
|---|---|---|
| FEAT-31-01 | Rules-zu-Additional-Instructions Migration | Bestehende Rules werden in Additional Instructions transferiert, Rules-Tab wird entfernt, System-Prompt-Erzeugung respektiert nur noch Additional Instructions. Migrations-Pass beim ersten Boot nach Update. |
| FEAT-31-02 | Workflow-Builder UI (n8n-aehnlich) | Visueller Editor fuer Workflow-Definition. Drag-and-Drop von Steps (Skill, Prompt, MCP-Call). Save als JSON oder Markdown-Frontmatter. |
| FEAT-31-03 | Workflow-Execution-Runtime | Engine die einen gespeicherten Workflow Schritt-fuer-Schritt ausfuehrt. Step-Inputs aus vorigen Step-Outputs ableitbar. Error-Handling pro Step (skip, retry, abort). |
| FEAT-31-04 | Workflow-Library und Sharing | Workflows als eigene Folder-Struktur in `.vault-operator/workflows/`, ggf. mit Marketplace-Anbindung (EPIC-30). |
| FEAT-31-05 | Modes endgueltig entfernen | Mode-System aus Code und UI entfernen (laut Memory bereits dead code, aber noch im Code). |

Volle FEAT-Specs werden in einer eigenen RE-Session nach Abschluss EPIC-29 Welle 3 geschrieben.

## Explicit Out-of-Scope

- Workflow-Versionierung (kann analog zu FEAT-29-09 spaeter kommen, oder direkt mit drin).
- Cloud-basierte Workflow-Synchronisation.
- Visual-Workflow-Debugging (Step-Through, Inspect).
- Konditionale Verzweigungen mit komplexer Logik (erste Version: nur sequenziell plus simple Conditionals).
- Eigene Skripting-Sprache (n8n-Like: visueller Builder reicht).

## Dependencies

- **EPIC-29 (Skills-Konsolidierung)**: Workflows komponieren Skills, also muss Skill-Foundation stabil sein.
- **FEAT-29-10 Composability**: Workflows nutzen Skill-zu-Skill und Skill-zu-MCP-Aufrufe als Primitives.
- **EPIC-22 Coordinator-Skill (FEAT-22-04)**: Architektur-Vorbild fuer Step-Sequenzen.

## Risks (Skizze)

| Risiko | Wahrscheinlichkeit | Impact | Mitigation |
|---|---|---|---|
| Workflow-Builder-UI wird zu komplex und kein User nutzt es | M | H | Iterativer Build mit User-Feedback nach jedem Sub-Feature, Start simpel und nur dann ausbauen wenn echte Use-Cases es brauchen. |
| Rules-Migration verliert Konfiguration | M | H | Backup vor Migration, klarer User-Hinweis was migriert wurde, Rollback-Option. |
| Workflow-Runtime-Fehler sind schwer debuggebar | M | M | Pro Step ein Audit-Log mit Inputs und Outputs, klare Fehlermeldungen mit Step-Index. |

## Aktueller Status

Skeleton-Epic. Die vollen Specs (Feature-Beschreibungen, Success Criteria, Akzeptanzkriterien, NFRs, ASRs) werden in einer eigenen RE-Session erstellt, sobald EPIC-29 Welle 3 deployed und stabil ist. Vor dem Start sollte ein BA-Pass die n8n-Pattern-Auswahl und die Workflow-Use-Cases sauber durchleuchten.
