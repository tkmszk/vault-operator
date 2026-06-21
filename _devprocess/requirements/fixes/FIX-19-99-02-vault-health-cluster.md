---
id: FIX-19-99-02
epic: EPIC-19
feature: FEAT-19-19
adr-refs: [ADR-95]
plan-refs: []
depends-on: [FIX-19-19-01]
audit-refs: [STABILITY-AUDIT-v2.14.0-2026-06-21]
created: 2026-06-21
---

# FIX-19-99-02: Vault Health Cluster (Stufe-3 gating, strongSignal, FreshnessFrontmatterPatcher, VaultHealthCheckTool, Cross-Property)

## Symptom

Stabilitaets-Audit 2026-06-21 fand fuenf eng verwandte Vault-Health-
Defekte:

1. Stufe-3 Periodic-Job lief am generischen `autoTrigger.enabled`-Flag
   mit (statt eigenem Setting). Plus: lastRun wurde nur ueber den
   In-Memory-`rolloverIfNewWeek`-Mechanismus gehalten und beim
   Plugin-Reboot neu berechnet, sodass theoretisch zwei Runs in
   einer Woche moeglich waren.
2. Stufe-3 strongSignal-Klassifikation zaehlte raw URLs (`length >= 2`).
   Zwei Links auf dieselbe Domain wurden als "strong" gewertet --
   Audit-Empfehlung war Domain-Grouping (eTLD+1) mit Threshold 3.
3. `FreshnessFrontmatterPatcher` ist seit IMP-20-06-01 W4-T1
   implementiert, aber nirgends instanziiert. Das Setting
   `freshness.writeFrontmatter` hatte keine Wirkung.
4. `VaultHealthCheckTool` umging Approval und Checkpoint: mass-edits
   ueber `fix_backlinks`, `cleanup`, `fix_categories` haben Hunderte
   von Notes ohne Restore-Punkt veraendert.
5. `checkMissingBacklinks` erwartete identische Property auf beiden
   Seiten. Live-Vault zeigte das eine high-Finding
   `Notes/Zettelkasten-Workflow.md`: Quelle.Notizen -> Konzept,
   Konzept.Quellen -> Quelle. Beide Properties sind semantisch
   reziprok, das Predicate aber nicht.

## Root Cause + Fix

### FIX-19-19-02 (Stufe-3 Gating + lastRunIso)

- `VaultIngestSettings.stufe3PeriodicJob: { enabled: boolean, lastRunIso: string }` neu in `src/types/settings.ts`.
- `src/main.ts` Stufe-3-Wrapper: Gate auf `stufe3PeriodicJob.enabled`, prueft `lastRunIso` (6-Tage-Margin damit der Wochen-Rhythmus nicht verrutscht), persistiert nach erfolgreichem run.

### FIX-19-19-02 (strongSignal Domain-Grouping)

- Neuer `countIndependentDomains(urls)`-Helper in `src/main.ts`: zerlegt jede URL via `new URL(...)`, zaehlt eTLD+1-Surrogat (`last-2-parts`).
- strongSignal-Branch von `>= 2 URLs` auf `>= 3 distinct domains` umgestellt.

### FIX-19-99-03 (FreshnessFrontmatterPatcher wiring)

- `FreshnessOrchestratorDeps` um drei optionale Felder erweitert: `frontmatterPatcher`, `writeFrontmatterEnabled`, `getFileByPath`. Test-Setups bleiben minimal.
- Neue Methode `maybeMirrorToFrontmatter(verdict)` schreibt den verdict-Label via Patcher's allowlist-Setze (`freshness:`) in die Note-FM. Gated durch das Setting.
- `src/main.ts` instanziiert `FreshnessFrontmatterPatcher` mit `new FrontmatterWriter(this.app, { storageMode: 'global' })` und wired die drei Felder.

### FIX-19-99-04 (VaultHealthCheckTool isWriteOperation=true + multi-file snapshot)

- `isWriteOperation` getter returnt jetzt `true`.
- Neue private Methode `takeRepairSnapshot(action)`: ruft `checkpointService.snapshot(taskId, allMarkdownFiles, 'vault_health_check:<action>')` mit allen Vault-MD-Files vor jeder Repair-Branch.
- Genutzt von `fix_backlinks`, `cleanup`, `fix_categories`. `cleanup_edges` braucht keinen Snapshot weil nur die DB veraendert wird.

### FIX-19-99-02 (Cross-Property-Reciprocity)

- `VaultHealthSettings.reciprocalProperties: Array<[string, string]>` neu (default `[['Notizen', 'Quellen']]`).
- `VaultHealthService.runChecks(options)` und `checkMissingBacklinks` nehmen `reciprocalProperties`.
- SQL wird zur Laufzeit erweitert: wenn `backlinksProperty='Notizen'` UND `[['Notizen','Quellen']]` konfiguriert, wird `AND e2.property_name = 'Notizen'` zu `AND e2.property_name IN ('Notizen','Quellen')`.
- Alle 4 `runChecks`-Caller (main.ts, AgentSidebarView, VaultHealthRepairModal, VaultHealthCheckTool) reichen das Setting durch.

## Tests

- `src/types/__tests__/vaultHealthSettings.test.ts` um zwei Tests erweitert: `reciprocalProperties` defaults + die Shape-Liste der fuenf Toggles.
- 2963 passing + 1 expected fail.

## Out of Scope (Deferred)

- **IMP-19-19-01: Stufe-2-Klick triggert keine Pipeline.** Der Hint-Click zeigt aktuell nur eine Tipp-Notice. Audit-Empfehlung: `AntiEchoSearchTool` ueber `ToolExecutionPipeline` ausloesen. Braucht einen Sidebar-public-API fuer Programmatic-Prompt-Inject und eine sub-task-Spawn-Logik. Architektur-Welle, kein Quick-Fix.
