---
id: FIX-19-28-03
feature: FEAT-19-28
epic: EPIC-19
adr-refs: [ADR-103]
plan-refs: []
depends-on: [FIX-19-28-01]
created: 2026-05-08
---

# FIX-19-28-03: PdfMarkdownMirror schreibt UTF-8-Mojibake

## Symptom

Live-Test 2026-05-08 (`/ingest-deep` auf
`Attachements/enbw-geschaeftsbericht-2025.pdf`, branch
`feature/block-source-citations`). Die geschriebene Mirror-Datei
`Sources/EnBW-Geschaeftsbericht-2025-Mirror.md` enthaelt durchgaengig
doppelt-encodiertes UTF-8.

Beispiele aus dem User-Sample:

- `GeschÃ¤ftsbericht` statt `Geschaeftsbericht`
- `â¬` statt `Euro-Zeichen`
- `1â2` statt `1-2`, `2024â2030` statt `2024-2030`
- `MarktfÃ¼hrer`, `KapitalerhÃ¶hung`, `RÃ¼ckert-Hennen`

User sieht Mojibake im Editor und in Reading-Mode. Frontmatter ist
ebenfalls betroffen (`title: EnBW GeschÃ¤ftsbericht 2025 â Mirror`).
Block-Anchors `^block-N` sind syntaktisch unbeschaedigt.

## Root cause

Klassisches "UTF-8-as-Latin-1" Pattern: das Byte-Paar `0xC3 0xA4`
(UTF-8 fuer ae) wurde irgendwo als zwei einzelne Latin-1-Zeichen
`Ã` plus `(c)` interpretiert und dann erneut UTF-8-encodiert. `write_vault`
selbst nutzt `vault.create` (Obsidian-API), das UTF-8 erwartet, ist
also nicht der Verursacher.

Verdaechtige Stellen, die noch zu pruefen sind (ohne Code-Lese-Pass
festgenagelt):

```
PDF-Tool extrahiert Text als UTF-8 Bytes
  -> Skill-Logic im PdfMarkdownMirror dekodiert als Latin-1
  -> Resultierender String wird vom Provider (Bedrock) als UTF-8
     reserialisiert
  -> write_vault speichert Mojibake
```

Alternative Hypothese: der LLM-Provider (Bedrock event-stream) liefert
die Antwort als Mojibake aus, wenn der eingehende PDF-Text ungewohnte
Bytes enthaelt. Live-Test mit OpenRouter waere ein Differenzialtest.

Zwei Hauptkandidaten im Code:
- `src/core/ingest/PdfMarkdownMirror.ts` (Mirror-Builder)
- `src/core/ingest/parsePdf.ts` oder PDF-Tool-Output-Pipeline

## Fix

Offen. Vorschlag:

1. Reproduce: kleinen Smoke-Test gegen ein PDF mit Umlauten, das
   `PdfMarkdownMirror` direkt mit dem PDF-Text-Output befuettert und
   pruefen, ob Mojibake bereits dort entsteht (Pre-Provider) oder erst
   nach dem Provider-Roundtrip (Post-LLM).
2. Wenn Pre-Provider: Encoding-Konvertierung im Mirror-Builder
   reparieren (`Buffer.from(text, 'utf8').toString('utf8')` statt
   impliziter Latin-1-Pfad).
3. Wenn Post-Provider: AgentTask Tool-Argument-Deserialization
   pruefen, insb. wenn Bedrock event-stream chunks zusammengefuegt
   werden ohne UTF-8-aware-Concat.

Implementation pointer: noch offen, abhaengig vom Reproducer.

## Regression test

Smoke-Test, der einen synthetischen Mini-PDF-Text (`"Geschaeftsbericht"`
mit echtem ae) durch den Mirror-Builder schickt und assertiert, dass
der Output exakt `"Geschaeftsbericht"` ist. Plus Re-Run gegen
`Attachements/enbw-geschaeftsbericht-2025.pdf` mit Assertion: kein
`Ã¤`/`Ã¶`/`Ã¼`/`â¬` in der Mirror-Datei.

## Status

See the backlog row for FIX-19-28-03 in `_devprocess/context/BACKLOG.md`
(status, phase, claim, commit SHA).
