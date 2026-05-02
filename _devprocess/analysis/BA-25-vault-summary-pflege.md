---
status: Draft
scope: MVP
created-by: /business-analysis on 2026-05-02
parent-ba: BA-19-knowledge-maintenance.md
related-epics: EPIC-15, EPIC-19, EPIC-03
---

# Business Analysis: Zentrale Vault-Summary- und Frontmatter-Pflege

> **Scope:** MVP
> **Erstellt:** 2026-05-02
> **Status:** Draft
> **Inspiration:** [Karpathy LLM-Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), Vertiefung von [BA-19](BA-19-knowledge-maintenance.md)

---

## 1. Executive Summary

### 1.1 Problem Statement

Obsilo hat mit EPIC-15 (Knowledge Layer), EPIC-19 (Knowledge Maintenance) und EPIC-20 (Graph Intelligence) die volle Karpathy-Wiki-Foundation bereits gebaut: SQLite-Storage mit Vektoren, Edges, Implicit-Connections, Ontologie, Freshness-Klassifikation, 4-Stufen-Retrieval-Pipeline mit Reranker, Vault-Health-Check, Knowledge-Ingest-Skill. **Was fehlt ist die Note-Level-Summary als gemeinsamer Anker fuer alle weiteren Operationen.** Der User pflegt heute manuell pro Note via Skill-Aufruf eine Frontmatter-Property "Zusammenfassung" plus Keywords plus Themen plus Konzepte. Das skaliert nicht: bei 1.500 Notes faellt entweder die Pflege weg, oder sie kostet 30-60 Sekunden pro Note plus mehrere LLM-Calls fuer Taxonomie-Suche.

Die zweite Luecke: Obsilos Wissen ueber den Vault liegt in der knowledge.db. Der Agent hat keinen verdichteten Vault-Ueberblick im stabilen Prompt-Prefix, sondern muss bei jeder Recherche-Frage erst search_vault aufrufen. Das ist KV-Cache-ineffizient und verlangt Tool-Roundtrips selbst dann, wenn der Agent nur "weiss, was es ueberhaupt gibt" muesste.

### 1.2 Proposed Solution

Vier zusammenhaengende Massnahmen:

1. **Zentrale Note-Summaries:** SemanticIndexService liest beim Indexing pro Note das Frontmatter. Existierende "Zusammenfassung" wird in eine neue Tabelle `note_summaries` uebernommen. Fehlt sie, wird sie via konfigurierbarem Standard-Prompt (Sebastians vorgegebener Wortlaut) generiert und in der DB gespeichert.

2. **SQL-beschleunigte Taxonomie-Pflege:** "Themen", "Konzepte" und "tags" werden beim Indexing aus dem Frontmatter in eine Tabelle `frontmatter_properties` gespiegelt. Bei Generierung neuer Notes liest der Agent die existierende Taxonomie aus SQL (1ms) statt LLM-Volltext-Suche (mehrere Sekunden plus Tokens).

3. **Setting-gated Vault-Frontmatter-Write:** Standardmaessig schreibt das System nur in die DB. Wer Karpathys Pattern voll will, aktiviert einen Toggle "Frontmatter-Pflege im Vault aktivieren". Das loest einen einmaligen Backfill-Lauf aus, der fehlende Properties ergaenzt und nichts ueberschreibt.

4. **Selektiver Top-Hub-Block im KV-Cache:** Ein Token-budgetierter (~3k) Block aus den Top-30 Hub-Notes der Ontologie wird optional in den stabilen System-Prompt-Prefix gehaengt. Standardmaessig aus, bis Telemetrie zeigt dass es Retrieval messbar verbessert.

### 1.3 Expected Outcomes

- Note-Summary-Pflege kostet User null aktive Zeit: Indexing erledigt es im Hintergrund.
- Existierende manuell gepflegte Summaries bleiben unangetastet, werden aber genauso konsumiert wie generierte.
- Frontmatter im Vault wird optional zentral gepflegt, ohne User-Edits zu zerstoeren.
- Taxonomie-Konsistenz (gleiche Schreibweise von Themen/Konzepten) steigt, weil Vorschlaege aus existierender Liste kommen statt jedes Mal LLM-frei erfunden.
- Agent gewinnt optional Vault-Awareness im KV-Cache, ohne dass User ihn aktiv triggern muss.

---

## 2. Business Context

### 2.1 Background

BA-19 hat das Karpathy-Wiki-Pattern als Leitstern fuer EPIC-19 etabliert und die drei Operationen (Ingest, Lint, Synthese) in Obsilo verankert. Was BA-19 nicht ausgearbeitet hat: **die Note-Level-Summary als verbindendes Element**. Karpathys index.md hat pro Page einen 1-Zeiler. Obsilo hat heute Per-Chunk-Text in `vectors.text`, aber keine Note-Level-Beschreibung. Diese Luecke macht drei Dinge teurer als noetig:

- Retrieval-Output: search_vault liefert Chunk-Snippets, kein Note-Level-Kontext.
- Taxonomie-Pflege: jeder neue Themen-Vorschlag braucht LLM-Suche statt SQL-Lookup.
- Vault-Awareness des Agents: ohne Note-Level-Index keine kompakte Karte fuer Cold-Start oder KV-Cache.

Sebastian pflegt heute manuell ein Frontmatter-Schema mit "Zusammenfassung" (1 Satz, 25 Worte, deutsch), "tags", "Themen", "Konzepte". Der Skill-Aufruf dafuer existiert als manueller Workflow. Skalierung ueber den Single-User-Use-Case scheitert aber an der Kadenz: 1.500 existierende Notes plus 5-10 neue pro Tag erzeugen Backlog statt Pflege.

### 2.2 Current State

**Note-Summary-Pflege heute:**

1. User markiert Note, ruft Skill mit Standard-Prompt auf.
2. LLM liest Note, generiert 1-Satz-Summary, 5-10 Keywords, 2-3 Themen, 2-3 Konzepte.
3. LLM ruft search_files auf, um existierende Themen/Konzepte zu finden (mehrere Tool-Calls).
4. LLM nutzt replaceInFile, um YAML-Frontmatter struktur-erhaltend zu erweitern.
5. Aufwand: 30-60 Sekunden pro Note, 5.000-10.000 Tokens pro Pflege-Pass.

**Technischer Status:**
- `vectors.text` enthaelt Volltext-Chunks plus optional LLM-Prefix (Pass-2 enrichment).
- `tags`-Tabelle enthaelt Tag-zu-Path-Mapping.
- `ontology`-Tabelle enthaelt Cluster-Membership mit Quelle (moc, implicit, ingest, louvain).
- Kein zentrales Note-Level-Summary-Feld. Kein zentraler Frontmatter-Property-Mirror jenseits der `tags`-Tabelle.

**Pain Points (nach Schwere):**

| # | Problem | Impact |
|---|---------|--------|
| 1 | Manuelle Summary-Pflege nicht skalierbar bei 1.500+ Notes | Backlog wird nie aufgeholt |
| 2 | Themen/Konzepte-Suche jedes Mal als LLM-Volltext-Search | Token-Verschwendung, langsam |
| 3 | Inkonsistente Schreibweise neu erfundener Themen/Konzepte | Cluster-Bildung leidet |
| 4 | Retrieval-Output ohne Note-Level-Kontext | Agent muss aus Chunks rekonstruieren |
| 5 | Agent hat keinen Vault-Ueberblick im Prompt | Tool-Roundtrip fuer jede Recherche |
| 6 | Frontmatter-Pflege ist Power-User-Workflow, nicht skalierbar an neue User | Knowledge-Layer-Mehrwert verfaellt ohne Pflege |

### 2.3 Desired State

**Indexing-Lauf (zukuenftig):**
1. SemanticIndexService scannt Note, baut Chunks und Vektoren wie heute.
2. Liest Frontmatter. Wenn "Zusammenfassung" existiert: in `note_summaries` uebernehmen.
3. Wenn nicht existiert und Setting "Auto-Summary generieren" aktiv: LLM-Call mit Standard-Prompt, Summary plus Keywords plus Themen plus Konzepte.
4. Themen/Konzepte werden gegen `frontmatter_properties` gemappt: existierende uebernehmen, neue als Vorschlag eintragen.
5. Wenn Setting "Frontmatter-Write aktivieren" aktiv: fehlende Properties in YAML ergaenzen via struktur-erhaltendem Pattern. Bestehende Properties unangetastet.
6. Note-Summary plus Keywords plus Themen-Liste in note_summaries und frontmatter_properties gespeichert.

**Retrieval-Output (zukuenftig):**
- search_vault liefert pro Hit zusaetzlich die Note-Summary aus note_summaries als Kontext-Zeile.
- recall_memory profitiert nicht direkt (operiert auf Memory v2 facts), aber Cross-Querying ueber UnifiedGraphService kann Note-Summary einbinden.

**Optional Vault-Awareness im Prompt:**
- Setting "Vault-Karte im Prompt-Prefix" aktivieren.
- ContextComposer haengt einen ~3k-Token-Block mit Top-30 Hub-Notes plus 1-Zeiler Summary plus Cluster-Header an.
- Block wird nur regeneriert wenn Hubs sich aendern oder Hub-Note re-summarized wird.

**MOC-File-Pflege (Subsystem):**
- MOC-Files existieren bereits pro Cluster (Thema, Konzept).
- Heute enthalten sie eine Base mit verlinkten Notizen.
- Neue Erweiterung: Header-Section mit auto-generierten Hub-Status, Implicit-Connection-Vorschlaegen, Cluster-Statistik. Strikt zwischen User-edited Body und auto-generierten Block durch Marker getrennt (analog Dataview-Pattern).

### 2.4 Gap Analysis

| Luecke | As-Is | To-Be | Mechanismus |
|--------|-------|-------|------------|
| Zentrale Summary | nicht vorhanden | `note_summaries`-Tabelle | DB-Schema-Erweiterung |
| Auto-Summary-Generierung | manueller Skill-Aufruf | beim Indexing (settings-gated) | SemanticIndexService Hook |
| Taxonomie-Mirror | nur tags, nicht Themen/Konzepte | `frontmatter_properties`-Tabelle | Schema-Erweiterung plus Indexing-Hook |
| Frontmatter-Write | nie | settings-gated, opt-in | Indexing schreibt struktur-erhaltend |
| Backfill | manuell pro Note | einmaliger Background-Job | Job-Runner mit Progress-UI |
| Vault-Awareness | search_vault nur on-demand | optional Top-Hub-Block im Prefix | ContextComposer-Erweiterung |
| MOC-Pflege | passiv, nur Base | aktiv, auto-generierter Header-Block | MOC-Marker-Konvention plus Pflege-Job |

---

## 3. Personas und Needs

### 3.1 Personas

**P1: Power-User mit grossem Vault (Sebastian)**
- Rolle: Knowledge-Worker, Forscher, Builder
- Goal: Wissen ueber Jahre kompoundieren, ohne Pflege-Last
- Pain: 1.500+ Notes, Pflege ist Engpass, Themen-Schreibweise driftet
- Quote: "Aktuell ist das ein nerviger extra Schritt"
- Top-Needs:
  - N1: Pflege-Tasks vom System uebernehmen lassen (asynchron, im Hintergrund)
  - N2: Vorhandene Pflege bewahren (kein Ueberschreiben)
  - N3: Konsistente Taxonomie ueber Jahre

**P2: Casual User mit mittelgrossem Vault**
- Rolle: Notiznehmer, gelegentlicher Researcher
- Goal: Vault wird beim Wachsen automatisch besser, ohne aktiv zu pflegen
- Pain: kennt MOC-Pattern nicht, wuerde nicht manuell pflegen
- Top-Needs:
  - N4: Default-Workflows, die ohne Setup-Wissen funktionieren
  - N5: Retrieval, das ab Tag 1 brauchbar ist

**P3: Neuer User mit kleinem Vault**
- Rolle: Erstausstieg in Obsilo
- Goal: Erleben, dass Obsilo den Vault als Wissenssystem versteht
- Pain: leerer Vault, kein Mehrwert sichtbar
- Top-Needs:
  - N6: Erste Notes werden sofort eingeordnet, nicht ignoriert

### 3.2 Cross-Persona Needs

- N7: Transparenz darueber, was das System gerade automatisch pflegt
- N8: Reversibilitaet: jede Auto-Aenderung muss zurueckdrehbar sein
- N9: Performance: Indexing darf den User nicht ausbremsen

---

## 4. Problem Analysis

### 4.1 Problem-Dimensionen

**Dimension A: Pflege-Skalierung.** Manuelle Pflege ist O(n) in der Note-Anzahl. Pro Note 30-60 Sekunden plus 5-10k Token. Bei 1.500 Notes summiert: 12-25 Stunden plus 7.500-15.000k Token Backlog. Auto-Pflege ist O(n) in der maschinellen Zeit, aber 0 in der User-Zeit.

**Dimension B: Konsistenz der Taxonomie.** Themen wie "AI-Agent" vs "KI-Agent" oder "Knowledge-Management" vs "Wissensmanagement" entstehen, wenn der Agent ohne Lookup neue Begriffe einfuehrt. SQL-Lookup gegen frontmatter_properties zwingt zur Disambiguierung beim Insert.

**Dimension C: Awareness-Asymmetrie.** Der User sieht den Vault. Der Agent sieht ihn nur ueber search_vault. Karpathys Loesung: Index in den Prompt, Awareness ohne Tool-Call. Trade-off: Tokens.

**Dimension D: Frontmatter-Hoheit.** Wer darf das Frontmatter aendern? User-Workflow heute: nur User, plus Skills die der User explizit triggert. Karpathy-Pattern: LLM darf alles. Mittelweg: Setting-gated, opt-in, struktur-erhaltend, kein Ueberschreiben.

### 4.2 Root Causes

- RC1: Karpathy-Pattern wurde in EPIC-19 konzeptionell uebernommen, aber Note-Level-Summary als verbindendes Element fehlt im Datenmodell.
- RC2: Taxonomie-Suche wurde als LLM-Aufgabe entwickelt, weil zur Entwicklungszeit kein Property-Mirror in der DB existierte.
- RC3: Frontmatter-Write war out-of-scope wegen Vault-Hoheits-Risiko, wurde aber nie als opt-in nachgereicht.
- RC4: KV-Cache-Optimierung (EPIC-18) hat den Prefix stabilisiert, aber Vault-Awareness nicht hinzugefuegt.

### 4.3 Jobs to be Done

**P1 Power-User:**
- "Wenn ich eine Note erstelle, will ich, dass Obsilo sie automatisch einordnet, damit ich den Pflege-Schritt einsparen kann."
- "Wenn ich eine bestehende Note bearbeite, will ich, dass meine bisherige Pflege nicht zerstoert wird, damit ich dem System trauen kann."
- "Wenn ich nach einem Thema frage, will ich, dass der Agent die richtigen existierenden Themen-Notes findet, damit transitives Retrieval funktioniert."

**P2 Casual User:**
- "Wenn ich einen Wissens-Vault aufbaue, will ich, dass er ohne mein Zutun strukturiert wird, damit ich den Mehrwert ohne Lernkurve bekomme."

---

## 5. Goals und KPIs

### 5.1 Business Goals

- BG1: Pflege-Last fuer Power-User auf null reduzieren, ohne User-Trust zu beschaedigen.
- BG2: Taxonomie-Konsistenz im Vault messbar erhoehen (weniger Synonym-Cluster).
- BG3: Retrieval-Qualitaet messbar verbessern, ohne Token-Budget zu sprengen.
- BG4: Casual und neue User profitieren ohne Setup.

### 5.2 KPIs (qualitativ wo Baseline fehlt)

| KPI | Baseline | Ziel | Messung |
|-----|----------|------|---------|
| Pflege-Zeit pro Note (P1) | 30-60s manuell | 0s im Default-Pfad | Telemetrie: User-Trigger vs Indexing-Trigger |
| Token-Kosten pro Indexing-Lauf (1.500 Notes) | unbekannt (heute null, weil Auto-Indexing keine LLM-Calls macht) | < 1.50 USD bei Haiku, < 5 USD bei Sonnet | LLM-Call-Tracking |
| Anteil Notes mit zentraler Summary | unbekannt (heute Frontmatter-only, nur ein Teil) | > 95% nach 1 Backfill-Lauf | DB-Query: count(notes ohne Summary) |
| Themen-Synonym-Cluster | unbekannt | reduziert um > 50% | Manueller Audit vor und nach SQL-Mapping |
| Adoption Frontmatter-Toggle (P1) | n.a. (Feature neu) | > 30% Aktivierung in 4 Wochen | Settings-Telemetrie |
| Retrieval-Recall mit Note-Summary (search_vault) | aktuelle Top-K-Recall-Rate | + 5 bis 10% messbar | A/B-Eval mit fixem Test-Set |
| KV-Cache-Top-Hub-Block: Token-Mehrkosten vs Tool-Call-Reduktion | n.a. | netto positiv, sonst Toggle off | Telemetrie: tokens_added vs search_vault_calls_avoided |

### 5.3 User Goals

- UG1: User soll spueren, dass Pflege passiert, ohne sie aktiv triggern zu muessen.
- UG2: User soll dem System vertrauen, dass es seine bisherige Pflege bewahrt.
- UG3: User soll Pflege-Aktivitaet jederzeit nachvollziehen und zurueckdrehen koennen.

---

## 6. Nordstern, Wow, Anti-Definition

### 6.1 Nordstern

Der Vault wird zum kompoundierenden Wissens-Artefakt, ohne dass der User dafuer Pflege-Zeit aufwendet. Karpathys Versprechen ("LLMs don't tire of bookkeeping") wird auf Obsilo-Niveau eingeloest, ohne den User-Workflow zu stoeren.

### 6.2 Wow

"Ich erstelle eine Note, mache mir keine Gedanken um Frontmatter, und beim naechsten Indexing-Lauf hat sie eine Zusammenfassung, passende Themen aus meiner existierenden Taxonomie, und passende Keywords. Wenn ich es nicht will, deaktiviere ich den Toggle."

### 6.3 Anti-Definition

**Was wir nicht bauen:**
- Kein automatischer Vault-Modus, der ohne User-Zustimmung Frontmatter aendert.
- Kein Ueberschreiben oder Loeschen bestehender Frontmatter-Properties.
- Kein KV-Cache-Block, der das Token-Budget unkontrolliert wachsen laesst.
- Keine Re-Generierung bestehender Summaries (alte Standards bleiben respektiert).
- Keine MOC-File-Pflege, die User-edited Content ueberschreibt.

---

## 7. Scope

### 7.1 In-Scope (existierend, wird genutzt)

- knowledge.db Schema v9 mit vectors, edges, tags, implicit_edges, ontology, note_freshness
- SemanticIndexService two-pass enrichment
- 4-Stufen Retrieval-Pipeline mit Reranker
- OntologyStore mit Cluster-Membership
- ContextComposer fuer System-Prompt-Komposition
- replaceInFile-Pattern fuer struktur-erhaltende YAML-Edits

### 7.2 In-Scope (neu zu bauen, Kandidaten fuer FEATURE-Specs)

| Kandidat | Epic-Mapping | Prioritaet |
|----------|--------------|------------|
| `note_summaries`-Tabelle plus Indexing-Hook | EPIC-15 | P0 |
| `frontmatter_properties`-Tabelle plus SQL-Taxonomie-Lookup | EPIC-15 | P0 |
| Standard-Prompt als Settings-konfigurierbarer Wert | EPIC-19 | P0 |
| Auto-Summary-Generierung beim Indexing (Setting-gated) | EPIC-19 | P0 |
| Frontmatter-Write Toggle plus Backfill-Job mit Progress-UI | EPIC-19 | P1 |
| Aktive MOC-File-Pflege mit Marker-Konvention | EPIC-19 | P2 |
| Selektiver Top-Hub-Block im KV-Cache (Setting-gated) | EPIC-03 | P2 |

### 7.3 Out-of-Scope

- Vollautomatisches Anlegen neuer Notes durch das System.
- Re-Generierung bestehender User-Pflege.
- Automatisches Ueberschreiben von User-Edits in MOC-Files.
- Token-budget-loser Vollstaendiger-Index im Prompt-Prefix (Karpathy-Variante A aus der Diskussion, verworfen wegen Skalierung).

### 7.4 Critical Hypotheses

- **H-01:** Note-Summary in note_summaries verbessert search_vault-Recall um messbare 5 bis 10% bei gleichbleibendem Token-Budget. *(Test: A/B-Eval mit fixem Query-Set vor und nach Aktivierung.)*
- **H-02:** SQL-Lookup fuer Themen/Konzepte reduziert LLM-Tokens pro neue Note um > 50% gegenueber LLM-Volltext-Suche. *(Test: Token-Tracking pro Note-Pflege-Lauf vor und nach Umstellung.)*
- **H-03:** Setting-gated Frontmatter-Write wird von > 30% der Power-User innerhalb 4 Wochen nach Release aktiviert. *(Test: Settings-Telemetrie nach Release.)*
- **H-04:** Selektiver Top-Hub-Block reduziert search_vault-Aufrufe pro Conversation um > 20% und kostet weniger Tokens als die eingesparten Tool-Roundtrips. *(Test: Telemetrie-A/B mit Block on vs off.)*
- **H-05:** Aktive MOC-File-Pflege mit auto-generiertem Header-Block stoert User-edited Content nicht und wird von Power-Usern als Mehrwert wahrgenommen. *(Test: Diff-Audit nach 4 Wochen, plus User-Befragung.)*
- **H-06:** Bestehende manuell gepflegte Frontmatter-Summaries bleiben in 100% der Faelle erhalten und werden 1:1 in note_summaries uebernommen. *(Test: Vor- und Nach-Diff aller Frontmatter-Felder im Backfill-Lauf.)*

### 7.5 Assumptions

- Sebastians Standard-Prompt repraesentiert eine Best-Practice, die als Default fuer neue User taugt.
- 1.500-Notes-Backfill mit Haiku ist token-oekonomisch tragbar.
- Der Indexing-Lauf darf langsamer werden, solange er asynchron im Hintergrund laeuft.
- Vault-Frontmatter-Edits ueber Obsidian-API kollidieren nicht mit aktiven User-Edits (Conflict-Detection im Indexing).

---

## 8. Risks

| ID | Risiko | Wahrscheinlichkeit | Impact | Mitigation |
|----|--------|--------------------|--------|------------|
| R-1 | Backfill-Lauf zerstoert User-Frontmatter durch fehlerhaftes replaceInFile-Pattern | Niedrig | Sehr hoch | Pre-Backfill-Diff plus User-Approval pro Batch, plus Vault-Backup-Empfehlung |
| R-2 | Auto-Summary-Generierung kostet bei Sonnet/Opus zu viele Tokens, User wird boese | Mittel | Hoch | Default-Modell konfigurierbar (Haiku als Default), Token-Budget-Cap, Settings-Toggle deaktivierbar |
| R-3 | SQL-Taxonomie-Lookup liefert irrelevante Vorschlaege, weil frontmatter_properties veraltet ist | Mittel | Mittel | Bei jedem Indexing-Lauf Mirror aktualisieren, Stale-Detection |
| R-4 | Top-Hub-Block bricht KV-Cache durch zu haeufige Regenerierung | Niedrig | Hoch | Regenerierung nur bei Hub-Membership-Aenderung, max 1x pro Tag |
| R-5 | MOC-Header-Block-Marker wird vom User versehentlich geloescht, Auto-Pflege macht es kaputt | Mittel | Mittel | Marker-Detection, Skip mit Warnung, kein Re-Insert |
| R-6 | iCloud-Sync-Conflicts bei Frontmatter-Write parallel zur User-Bearbeitung | Mittel | Mittel | Conflict-Detection vor Write, Skip mit Log-Eintrag |

---

## 9. Constraints

### 9.1 Technisch

- Sebastians Standard-Prompt-Wortlaut ist bindend (im konfigurierbaren Settings-Feld als Default hinterlegen).
- Frontmatter-Schreiben muss replaceInFile-Pattern nutzen, struktur-erhaltend.
- knowledge.db v9 Schema additiv erweitern, kein Breaking Change. Neue Schema-Version v10.
- Indexing-Lauf darf nicht im UI-Thread blockieren.
- Mobile (iOS/Android) muss zumindest Read-Pfad unterstuetzen, Write-Pfad kann Desktop-only sein.

### 9.2 Strategisch

- Default-Verhalten konservativ (kein Vault-Write), Power-User-Mehrwert opt-in.
- Karpathys Pattern wird umgesetzt, aber keine 1:1-Kopie. Anpassung an Obsidian-Vault-Hoheit.
- Vault-Hoheit bleibt beim User. System pflegt nur, was er aktiviert.

### 9.3 Delivery

- Implementation in vier Phasen entlang der vier Sub-Initiativen, jede einzeln deploybar.
- Phase 1: note_summaries plus frontmatter_properties als reine DB-Schicht.
- Phase 2: Auto-Summary-Generierung beim Indexing (Setting-gated, Default off).
- Phase 3: Frontmatter-Write plus Backfill-Job (Setting-gated, Default off).
- Phase 4: MOC-Pflege und KV-Cache-Block, abhaengig von Telemetrie aus Phase 1 bis 3.

---

## 10. Requirements Overview

### 10.1 Feature-Kandidaten (gehen in Requirements Engineering)

Pro Sub-Initiative ein Feature, plus ein Cross-Cutting-Feature fuer den Standard-Prompt. RE-Phase verteilt sie auf die existierenden Epics.

1. **FEAT-15-09** Note-Summary Storage (note_summaries-Tabelle + Indexing-Hook)
2. **FEAT-15-10** Frontmatter-Property Mirror (frontmatter_properties + SQL-Taxonomie-Lookup)
3. **FEAT-19-08** Konfigurierbarer Standard-Prompt (Settings-Feld, Default = Sebastians Wortlaut)
4. **FEAT-19-09** Auto-Summary-Generierung beim Indexing (Setting-gated, Default off)
5. **FEAT-19-10** Frontmatter-Write plus Backfill-Job (Setting-gated, Default off)
6. **FEAT-19-11** Aktive MOC-File-Pflege (Setting-gated, Default off)
7. **FEAT-03-26** Selektiver Top-Hub-Block im KV-Cache (Setting-gated, Default off)

(IDs sind Vorschlaege, RE bestaetigt oder vergibt neu.)

### 10.2 NFR-Prioritaet

1. Daten-Sicherheit (kein Frontmatter-Verlust).
2. User-Trust (Reversibilitaet, Transparenz).
3. Performance (Indexing-Latenz, KV-Cache-Stabilitaet).
4. Token-Oekonomie (Backfill-Kosten, Pro-Note-Kosten).
5. Skalierbarkeit (Vault-Groessen 100 bis 10.000+ Notes).

### 10.3 ADR-Bedarf (Indikatoren fuer Architecture-Phase)

- ADR fuer note_summaries-Schema-Design (separate Tabelle vs Spalte in vectors).
- ADR fuer frontmatter_properties-Schema (Erweiterung tags vs eigenstaendige Tabelle).
- ADR fuer Conflict-Detection-Strategie bei parallelem User-Edit.
- ADR fuer MOC-Marker-Konvention (HTML-Comment vs Dataview-Block vs eigene Syntax).
- ADR fuer KV-Cache-Block-Lifecycle (Trigger fuer Regenerierung).

---

## Anhang A: Verweise

- [BA-19 Knowledge Maintenance](BA-19-knowledge-maintenance.md) (Karpathy-Pattern als Leitstern)
- [EPIC-15 Knowledge Layer](../requirements/epics/EPIC-15-knowledge-layer.md)
- [EPIC-19 Knowledge Maintenance](../requirements/epics/EPIC-19-knowledge-maintenance.md)
- [EPIC-03 Context, Memory and Scaling](../requirements/epics/EPIC-03-context-memory-scaling.md)
- [Karpathys LLM Wiki Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## Anhang B: Sebastians Standard-Prompt (Default-Wert fuer FEAT-19-08)

Wird als Default in Settings hinterlegt, vom User editierbar. Inhalt steht in der Konversation, wird in PLAN-Phase woertlich uebernommen.
