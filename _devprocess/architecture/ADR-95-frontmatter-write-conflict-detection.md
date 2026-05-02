---
id: ADR-95
title: Frontmatter-Write Conflict-Detection
status: Proposed
deciders: Architecture
date: 2026-05-03
related:
  - BA-25
  - FEAT-19-09
  - FEAT-19-10
---

# ADR-95: Frontmatter-Write Conflict-Detection

## Context

Auto-Summary-Generierung (FEAT-19-09) und Backfill-Job (FEAT-19-10) schreiben Frontmatter-Properties in Vault-Notes. Wenn der User waehrend des Writes dieselbe Note bearbeitet, droht Daten-Verlust oder YAML-Korruption. iCloud-Sync verschaerft das Problem (Cross-Device-Edit). Bindende User-Entscheidung: bestehende Properties NIE ueberschreiben.

## Decision Drivers

- Daten-Sicherheit (User-Trust)
- iCloud-Sync-Kompatibilitaet
- Performance (Write darf nicht serialisieren)
- Wiederholbarkeit nach Conflict

## Considered Options

### Option A: mtime-Pre-Check, optimistic concurrency

Vor Write: mtime der Note lesen. Vor finalem Write: mtime erneut lesen. Wenn unveraendert, Write. Wenn geaendert, Skip plus Notification.

Pros:
- Einfach implementierbar.
- Performance: kein Lock noetig.

Cons:
- Race-Window zwischen mtime-Check und Write.
- iCloud-Sync verschiebt mtime non-deterministisch.

### Option B: Vault.process(file, transformer) (Obsidian-API atomic edit)

Pros:
- Obsidian-Native, atomic auf Single-Device.
- Verhindert Race auf demselben Device.

Cons:
- Cross-Device-Race bleibt (zwei Plugins editieren parallel).
- WriterLock-Pattern aus ADR-79 fuer iCloud-Mode noetig.

### Option C: Hybrid: Vault.process auf Single-Device, WriterLock im obsidian-sync-Mode

Pros:
- Robust gegen alle bekannten Race-Modi.
- Reuse existierender WriterLock-Mechanik (BUG-029 Fix).
- Performance: Lock nur im Sync-Mode aktiv.

Cons:
- Doppelte Code-Pfade je nach Storage-Mode.

## Decision

**Option C**: Hybrid. Vault.process(file, fn) wird Standard-Write-Pfad. Im obsidian-sync-Mode zusaetzlich WriterLock vor dem Vault.process-Aufruf.

Begruendung:
- Vault.process ist die idiomatische Obsidian-Loesung fuer atomic Frontmatter-Updates.
- WriterLock-Pattern existiert bereits (ADR-79) und ist battle-tested fuer Cross-Device-Race.
- Performance-Penalty nur fuer User in Sync-Mode (typischerweise Power-User, akzeptabel).
- Bei Conflict: Skip plus Log-Eintrag plus optional Notification an User.

## Consequences

### Positive
- Zero Frontmatter-Corruption Toleranz erreicht (BA-25 H-06).
- iCloud-Sync-User bekommen sicheren Write-Pfad.
- Wiederholbar: Backfill kann Skipped Notes in naechster Iteration retry'en.

### Negative
- Code-Pfad pro Storage-Mode unterschiedlich.
- Bei Conflict-Storm (User editiert gerade aktiv 50 Notes) wird Backfill langsam.

### Risks
- Vault.process-API-Aenderungen in zukuenftigen Obsidian-Versionen koennten Pattern brechen. Mitigation: Pattern in einem Helper kapseln, Single-Source-of-Truth.

## Implementation Notes

Frontmatter-Helper-Funktion:
- Input: TFile, transformerFn (current Frontmatter -> new Frontmatter).
- Im Helper: detectStorageMode() -> wenn obsidian-sync, acquire WriterLock. Dann vault.process(file, oldContent => writeFrontmatter(oldContent, transformerFn)). Release Lock.
- Bei Conflict (Vault.process throws oder Lock-Acquire failt): skip plus log.warn.
- Konflikt-Counter pro Backfill-Run, am Ende Notification "X notes skipped due to conflicts".
