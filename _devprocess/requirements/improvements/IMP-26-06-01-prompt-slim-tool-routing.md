---
id: IMP-26-06-01
feature: FEAT-26-06
epic: EPIC-26
adr-refs: [ADR-08, ADR-62]
plan-refs: []
audit-refs: []
depends-on: []
created: 2026-05-17
---

# IMP-26-06-01: Prompt-Slim Welle 2 -- tool-routing-Section konditional rendern

## Motivation

EPIC-26 Welle 3 (FEAT-26-06) hat cost-heuristics und plugin-skills in
schlanke Lean-Varianten gebracht (Trigger ueber
`pluginSkillsLean` / `recentPluginSkillUsage`). Die `tool-routing`-
Sektion im Modular-System-Prompt (ADR-08) bleibt unangetastet voll
gerendert.

In Auto-Sessions mit wenig genutzten Tool-Gruppen koennte das
Section schlanker rendern: nur die wirklich relevanten
Group-Beschreibungen, statt aller Gruppen + ihrer Anleitungen.
Erwartete zusaetzliche Prompt-Reduktion: ~3-5%.

## Vorschlag

- Heuristik: aktive Tool-Use im letzten N Turn-Fenster pro Gruppe
  zaehlen.
- Bei <Threshold: nur Headline rendern, ohne Tool-Liste / Routing-
  Anleitung.
- Bei aktiver Nutzung: volle Sektion wie heute.
- Switch via Setting + Default an in Auto-Mode.

## Implementation pointer

- `src/core/prompts/sections/tools.ts` und `tool-routing.ts`
  (falls separat).
- `src/core/AgentTask.ts` muss Recent-Group-Usage tracken (analog zu
  `recentPluginSkillUsage`).

## Akzeptanz

- Prompt-Diff messbar (Tokens vor/nach bei einer Standard-Auto-Session).
- Keine Regression auf Tool-Discovery (find_tool funktioniert
  weiterhin).
- Test: einen Lauf mit nur read-only Vault-Tools und einen mit
  vollem Edit + MCP nebeneinander vergleichen.

## Prioritaet

P3. cost-heuristics und plugin-skills sind die groesseren Hebel;
tool-routing-Slim ist Polish.

## Status

Siehe BACKLOG-Row IMP-26-06-01.
