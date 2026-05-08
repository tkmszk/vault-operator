---
id: FIX-19-28-04
feature: FEAT-19-28
epic: EPIC-19
adr-refs: [ADR-103]
plan-refs: []
depends-on: []
created: 2026-05-08
---

# FIX-19-28-04: PdfMarkdownMirror deckt nur Teil der PDF ab (1-135 von 410)

## Symptom

Live-Test 2026-05-08 mit
`Attachements/enbw-geschaeftsbericht-2025.pdf` (410 Seiten). Der
generierte Mirror `Sources/EnBW-Geschaeftsbericht-2025-Mirror.md`
endet mit `Mirror erstellt: 2026-05-07 | Seiten abgedeckt: 1-135
(strategisch relevante Abschnitte)` und enthaelt 43 Block-Anchors
(`^block-1` bis `^block-43`).

User-Erwartung (bestaetigt 2026-05-08): vollstaendiger Mirror aller
410 Seiten, damit jede spaetere Sense-Making-Frage einen Anker findet.
Aktuelles Verhalten ist nicht dokumentiert (kein Setting, kein
Hinweis, kein Skill-Step), wirkt fuer den User wie ein willkuerlicher
Cut-off.

## Root cause

Noch offen. Drei Hypothesen:

1. **Skill-Logic-Filter**: Der `/ingest-deep`-Skill triggert eine
   Cut-off-Heuristik fuer "strategisch relevante Abschnitte" und stoppt
   nach Lagebericht / Wirtschaftsbericht. Der LLM-Agent kompiliert
   den Mirror selbst (statt deterministischem Tool-Pfad) und entscheidet
   was relevant ist. Footer-Text "(strategisch relevante Abschnitte)"
   stuetzt diese Hypothese -- der Wortlaut riecht nach LLM-Output.

2. **Token-/Char-Cap im Mirror-Builder**: `PdfMarkdownMirror.ts`
   hat einen Hard-Cap (z.B. 100k Zeichen oder 200 Seiten), der bei
   grossen PDFs frueh abbricht. Der Footer wird trotzdem geschrieben.

3. **read_document-Pagination ohne Auto-Continue**: Skill ruft
   `read_document` mit `start_page`/`end_page` und stoppt nach 2-3
   Aufrufen, weil der Skill-Workflow den naechsten Block nicht
   automatisch nachlaedt. Der User-Test-Transcript zeigt nur Reads bis
   `start_page=9, end_page=20`, danach Triage und Mirror-Erstellung.

Die User-Beobachtung 43 Anchors fuer 135 Seiten passt zu Hypothese 1
(LLM destilliert) oder 3 (Pagination-Stop nach manuellem read).
Hypothese 2 (deterministischer Cap) waere weniger wahrscheinlich,
weil der Footer dann eher "abgebrochen bei N Zeichen" lauten wuerde.

## Fix

Offen. Vorschlag:

- Pruefen ob der Skill-Workflow eine deterministische "alle Seiten
  lesen"-Schleife hat oder LLM-getrieben ist.
- Falls LLM-getrieben: Skill-Description anpassen mit klarer Anweisung
  "Mirror MUSS jede Seite enthalten, kein selektiver Filter".
- Falls deterministisch mit Cap: Cap konfigurierbar machen oder
  entfernen, mindestens aber Footer mit ehrlichem Cap-Hinweis schreiben
  ("abgebrochen bei N von M Seiten wegen Token-Cap, naechste Seiten
  separat ingestieren").
- Ggf. Setting `vaultIngest.mirrorMode`: `full` (Default) | `selective`
  (LLM-Filter) | `paginated` (manueller Re-Run pro Block).

Implementation pointer: erst nach Diagnose-Pass durch
`src/core/ingest/PdfMarkdownMirror.ts` und Skill-Spec
`_devprocess/architecture/skills/ingest-deep.skill.md`.

## Regression test

Smoke-Test mit dem EnBW-Bericht: `/ingest-deep` muss Mirror mit
mindestens 90% der PDF-Seitenzahl als Block-Anchors produzieren, oder
explizit dokumentierten Cut-off mit Hinweis im Footer.

## Status

See the backlog row for FIX-19-28-04 in `_devprocess/context/BACKLOG.md`
(status, phase, claim, commit SHA).
