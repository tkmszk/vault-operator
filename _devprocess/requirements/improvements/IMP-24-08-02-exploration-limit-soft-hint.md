---
id: IMP-24-08-02
feature: FEAT-24-08
epic: EPIC-24
adr-refs: [ADR-114, ADR-113, ADR-06]
plan-refs: []
audit-refs: []
depends-on: []
created: 2026-05-17
---

# IMP-24-08-02: Weiches Exploration-Limit -- Hinweis nach N read/search-Calls ohne produktiven Schritt

## Motivation

Aus FEAT-24-08 / ADR-114 zurueckgestellt. Mechanik C der drei
Autonomie-Bausteine; in v2.11.5-beta.2 wurde nur Mechanik B
(Steering-Hook) implementiert.

Ein Agent mit vagem Auftrag ("recherchiere X") kann beliebig viele
read/search-Tool-Aufrufe machen, bevor er etwas tut. Jede dieser
explorativen Iterationen ist ein voller Kontext-Durchlauf -- der
gesamte bisherige Verlauf wird ans LLM mitgeschickt. Typisches
Symptom: 15-20x `read_file` vor dem ersten `write_file`, Kontext
blaeht sich auf, Folge-Kosten steigen ueberproportional.

ADR-06 (Wiederholungs-Detektor) ist enger (gleiches Tool mit gleichen
Args wiederholt). Das Exploration-Limit ist die breitere Bremse:
verschiedene read-Tools ueber mehrere Iterationen ohne produktiven
Schritt.

## Vorschlag

1. **Tool-Klassifikation**: zwei Gruppen im Loop-Zaehler:

   | Counter +1 (Read/Search) | Counter = 0 (Produktiv) |
   |---|---|
   | read_file, list_files, search_files | write_file, edit_file, append_to_file |
   | semantic_search, anti_echo_search | attempt_completion |
   | read_document, read_mcp_tool | new_task (Subtask-Spawn) |
   | web/browser-Tools | create_pptx/docx/xlsx etc. |

2. **Counter im Loop**: Zaehler hochzaehlen bei read/search-Tool-Use,
   zuruecksetzen bei jedem produktiven Tool oder bei der finalen
   Antwort.

3. **Schwellwert-Hinweis**: bei `explorationLimit` (Default 8) eine
   zusaetzliche User-Message vor der naechsten Iteration einspeisen:
   "Du hast 8 reine Lese-/Suchschritte hintereinander gemacht ohne
   produktives Ergebnis. Entweder fokussieren (was ist die konkrete
   naechste Aktion?) oder die Recherche in einen Subtask delegieren
   (`new_task` mit `profile: 'research'`)."
4. **Kein Abbruch** -- nur Steuerimpuls. Der Agent kann selbst
   entscheiden, was er tut.

## Settings

- `explorationLimit: number` (Default 8)
- Pfad in WRITABLE_PATHS.

## Verwandt

- ADR-113 / FEAT-24-04 `research`-Profile: der Hinweis empfiehlt die
  Subtask-Delegation als Ausweg. Token-Spareffekt durch isolierten
  Subtask-Kontext.
- ADR-06 Wiederholungs-Detektor: enger, automatisch. Exploration-Limit
  ist breiter, "weicher" (kein Abbruch, nur Hinweis).
- IMP-24-08-01 Task-Budget: harte Bremse darueber.

## Akzeptanz

- Counter zaehlt korrekt hoch bei read/search.
- Reset bei Edit/Subtask/Completion.
- Bei Schwellwert-Erreichen erscheint die User-Message vor der
  naechsten Iteration.
- Einfache Tasks (3-5 produktive Schritte) beruehren das Limit nie.

## Status

Zurueckgestellt 2026-05-17 zusammen mit IMP-24-08-01. Siehe
BACKLOG-Row IMP-24-08-02.
