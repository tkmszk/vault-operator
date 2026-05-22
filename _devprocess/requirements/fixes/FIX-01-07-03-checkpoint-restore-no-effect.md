---
id: FIX-01-07-03
feature: FEAT-01-07
epic: EPIC-01
adr-refs: []
plan-refs: []
depends-on: []
created: 2026-05-22
---

# FIX-01-07-03: Checkpoint-Restore meldet "Restored", aber Datei behaelt geaenderten Inhalt

## Symptom

Sebastian laesst den Agent das Frontmatter einer Note umbauen ("repariere das frontmatter in dieser note"). Der Pipeline-Hook erzeugt automatisch einen Pre-Write-Checkpoint, die Aenderung wird geschrieben. Sebastian klickt anschliessend in der Sidebar auf "Undo from here". Console-Log zeigt `[Checkpoints] restored via vault.modify` und `[Checkpoints] Restored 1 files for task task-1779486137419`. Trotzdem zeigt die Datei in Obsidian weiter den geaenderten Stand, die Frontmatter-Aenderung ist nicht zurueckgerollt.

Reproduktion: Note `Notes/Backpropagation (Fehlerrückführung).md` im NexusOS-Vault, frontmatter-cleanup Tool-Call, dann Undo-from-here.

## Root cause

Unbekannt -- mehrere Hypothesen, keine im Logging eindeutig belegt:

```
agent edit -> pipeline pre-write snapshot (oid A)
            -> agent write_file (new content)
user clicks Undo from here
            -> service.snapshot(restore-..., files, 'undo_from_here') (oid 5b875779, 28192 chars)
            -> service.restore(commitOid=34dd329d, files=...)
            -> "restored via vault.modify"
            -> file content visible to user unchanged
```

Verdaechtig: snapshot vor Restore liest 28192 chars, restore schreibt 28192 chars von einem ANDEREN Oid. Wenn beide identisch lang sind, ist der Inhalt entweder zufaellig gleich (Frontmatter-Aenderung gleicht zeichen-neutral aus) oder der Pre-Change-Snapshot enthielt schon die Aenderung (z.B. weil pre-write-Snapshot zeitlich nach dem Edit lief).

Hypothesen, in Reihenfolge der Wahrscheinlichkeit:

1. **Falscher Snapshot-Oid:** `commitOid=34dd329d` ist nicht der Pre-Change-Snapshot sondern der erste Snapshot der Datei, der bereits den aktuellen Stand enthielt. Pre-Write-Checkpoint wurde verpasst (z.B. weil das Agent-Tool kein `isWriteOperation` triggert oder weil ein anderes Plugin das Frontmatter geschrieben hat ohne Checkpoint).
2. **vault.modify ueberschrieben:** Restore-Call schreibt korrekt, aber direkt danach feuert ein Sync- oder Frontmatter-Plugin (templater, frontmatter-property, dataview-cache) und schreibt den vorherigen aktuellen Stand zurueck.
3. **Editor-View-Cache:** vault.modify schreibt korrekt auf Disk, aber Obsidians Editor-View ist nicht refreshed und zeigt den alten Pufferinhalt. Sebastian wuerde aber nach Note-Switch + Re-Open den Disk-Stand sehen.
4. **Falsche Datei restored:** Restored taskId ist `task-1779486137419`, aber der Pre-Change-Snapshot gehoert zu einem anderen Task. Cross-Task-Restore zeigt einen anderen Inhalt als erwartet.

Logs liefern Restore-Pfad bis vault.modify, aber kein nachgelagertes Verify (Read-back nach Write). Das macht die Ursachendiagnose ohne Reproduktion mit zusaetzlichem Logging unmoeglich.

## Fix

{Offen -- braucht Reproduktion mit Zusatz-Logging.}

Erster Schritt waere: `GitCheckpointService.restore()` um einen post-write-Read-back ergaenzen, der den geschriebenen Inhalt liest und mit dem erwarteten oid-Inhalt vergleicht. Diff dokumentiert ob vault.modify durchgegangen ist oder ob ein externer Schreibvorgang den Inhalt ueberschrieben hat. Optional Marker fuer "expected pre-write checkpoint missing" wenn der Snapshot vor dem ersten Edit nicht angelegt wurde.

Implementation pointer: TBD.

## Regression test

Offen. Reproduktions-Setup braucht: realer Vault (sehr wahrscheinlich plugin-Interferenz), Note mit verschachtelter Frontmatter, Agent-Tool-Call der Frontmatter aendert, dann Undo. Reproduktion auf synthetischem Test-Vault unklar.

## Status

See the backlog row for FIX-01-07-03 in `_devprocess/context/BACKLOG.md`.
