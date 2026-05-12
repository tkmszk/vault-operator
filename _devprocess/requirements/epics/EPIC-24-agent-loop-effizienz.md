---
id: EPIC-24
title: Agent-Loop Effizienz
date: 2026-05-12
related: RESEARCH-36, BA-12
predecessor: EPIC-18
---

# EPIC-24: Agent-Loop Effizienz

## Hypothesis Statement

FUER Obsilo-Nutzer die taeglich dialogische Notiz- und Recherche-Arbeit an den
AI-Agenten delegieren
DIE unter weiter ueberlinear wachsenden Input-Token-Kosten und Provider-
Inkompatibilitaet (168k-Limit) leiden, weil EPIC-18 nur den Fast-Path-Fall
abdeckt
IST DIE Agent-Loop-Effizienz
EIN Loop-/Harness-Refactoring (kein Neubau, kein SDK-Umstieg)
DAS den Input-Token-Verbrauch dialogischer Multi-Turn-Sessions strukturell
deckelt -- durch History-Komprimierung an Turn-Grenzen, Tool-Output-Disziplin,
korrektes Prompt-Caching und delegierte Subagent-Teilaufgaben
IM GEGENSATZ ZU dem aktuellen Ansatz (Tool-Results akkumulieren ungekuerzt in
der History; `cache_control` sitzt auf dem Anthropic-Pfad an der falschen Stelle;
Recherche laeuft im Hauptkontext)
UNSERE LOESUNG uebernimmt die *Disziplinen* aus Claude Code und EnBW Cowork
(stabiler gecachter Praefix, Threshold-Compaction mit Recent-Keep, gebudgetete
Subagent-Handoffs, eingebaute Cache-/Token-Telemetrie), ohne den bewaehrten
ReAct-Loop-Kern anzutasten.

## How might we

Wie kann eine lange dialogische Session (Recherche + Editieren ueber 10+ Turns)
unter dem 168k-Token-Limit bleiben und nur einen Bruchteil der heutigen Kosten
verursachen, ohne dass die Ergebnisqualitaet leidet und ohne dass der Nutzer
manuell Kontext-Hygiene betreiben muss?

## Business Outcomes (messbar)

- **OUT-01**: Ein 4-Datei-Read-Turn endet bei ~48k Input-Tokens, der unmittelbar
  folgende Turn startet unter ~20k (statt mit den ~48k im Volltext-Schlepptau).
- **OUT-02**: Ein 10-Turn-Chat (Mix aus Recherche + Edit) bleibt im History-Anteil
  deutlich unter linearem Wachstum -- gemessen via `[InputBreakdown]`.
- **OUT-03**: Auf dem Anthropic-direkt-Pfad ist `cacheRead` ab dem 2. API-Call
  > 0 (statt erneutem `cacheCreate`); die Sidebar-Kostenanzeige zeigt auf allen
  Providern den realen Wert (Cache-Reads abgezogen).
- **OUT-04**: Eine einfache Frage, die der Agent durch eine Recherche-Teilaufgabe
  beantwortet, laesst den Hauptkontext nur um die verdichtete Antwort wachsen,
  nicht um die N Such-/Lese-Zwischenstaende.

## Features (priorisiert)

### P0 (Welle 1 -- "History-Wachstum stoppen + Caching/Anzeige geradeziehen")

| ID | Title | Wert |
|----|-------|------|
| FEAT-24-01 | Caching-Architektur-Fix (Anthropic/Bedrock) + Kostenanzeige | Stabiler gecachter Praefix; Anzeige stimmt |
| FEAT-24-02 | History-Komprimierung: Microcompaction der Tool-Results an Turn-Grenzen | Stoppt den dominanten Wachstumstreiber |
| FEAT-24-03 | Tool-Output- & Kontext-Disziplin: Externalizer im Hauptloop, Re-Read-Cap, grosse User-Messages kappen | Deckelt Spitzen (Web/Paste/Multi-Read) |

### P1 (Welle 2 -- "Loop strukturell verschlanken")

| ID | Title | Wert |
|----|-------|------|
| FEAT-24-04 | Subagent-Delegation fuer context-heavy Teilaufgaben (mit Per-Call-Token-Budget) | Recherche/Exploration bleibt aus dem Hauptkontext |
| FEAT-24-05 | Sichtbarkeit: Sidebar-Kosten-/Token-/Cache-Hit-Anzeige | Verhaltenseffekt + Diagnose |

### P2 (Welle 3 -- "Governance + Routing + Lazy-Loading", teils Spike-first)

| ID | Title | Trigger / Hinweis |
|----|-------|-------------------|
| FEAT-24-06 | Lazy-Loading Tool-Schemas + Active-Skills on-demand | Spike zuerst (`tools`-Feld-Groesse messen) |
| (in FEAT-24-04) | Token-/Kosten-Budget pro Task + Steering-Hook | -- |
| (neue FEAT bei Bedarf) | Internes Hilfs-Modell-Routing (Condensing/Read-Planner/...) | nur wenn Hilfs-Calls signifikanten Anteil haben |
| (neue FEAT bei Bedarf) | Expliziter Plan-Modus (read-only -> reviewter Plan -> Kontext-Reset) | eigene Architektur-Entscheidung |

## Out-of-Scope (Epic)

- Neuer Agent-Loop / Umstieg auf ein Coding-Agent-SDK (Claude Agent SDK,
  `pi-coding-agent`) -- bewusst verworfen, s. RESEARCH-36 §7.
- User-konfigurierbares Hook-System a la Claude Code (Aufwand >> Nutzen fuer ein
  Obsidian-Plugin).
- Multi-Agent-`coordinator`/`Team`-Subsystem (Overkill).
- Retrieval-Tuning (semantisches Chunking / Top-k / Reranker) -- beruehrt
  SemanticIndexService/KnowledgeDB, gehoert zur Knowledge-Layer-/Memory-v2-
  Roadmap, nicht hierher (s. RESEARCH-36 §9 Frage 9).
- Caveman-/Output-Knappheits-Modus -- Output ist nicht das Problem (RESEARCH-36
  Befund G).

## Critical Hypotheses (Leading Indicators)

- H1: Microcompaction der Tool-Results an Turn-Grenzen kostet keine relevante
  Ergebnisqualitaet (Shadow-Mode / A-B noetig, s. §9 Frage 4).
- H2: Der gecachte stabile System-Praefix bleibt turn- und sessionuebergreifend
  konstant, wenn DateTime/Memory/Active-Skills/Recipe/Vault-Context aus dem
  gecachten Bereich raus sind (Beleg: EnBW Cowork macht es genauso, RESEARCH-36 §3).
- H3: Model-getriebene Subagent-Delegation (prominentes `new_task` + Profile +
  Prompt-Leitplanke) reicht ohne harten Router (Beleg: Claude Code, RESEARCH-36 §8 E).

## Definition of Done (Epic)

- Alle P0-Features (FEAT-24-01..03) auf Done und released.
- OUT-01 bis OUT-03 messbar erfuellt (via `[InputBreakdown]`/`[CacheStat]`/`[Cost]`).
- ADRs vorhanden und referenziert: Caching-Architektur (erweitert ADR-62),
  Microcompaction/History-Pruning (neu), Externalization-im-Hauptloop (erweitert
  ADR-63), Subagent-Delegation+Budget (neu).
- Diagnose-Logging (`src/api/logCacheStat.ts`) committed bzw. in den Welle-1-Code
  ueberfuehrt.
- `/testing` + `/security-audit` nach jedem `/coding`-Durchlauf durchlaufen.
- Wiedervorlage FEAT-24-06 (Lazy-Loading) als Backlog-Row mit Spike-Trigger.
