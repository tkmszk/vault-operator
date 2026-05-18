---
id: IMP-26-00-01
feature: EPIC-26
epic: EPIC-26
adr-refs: [ADR-120, ADR-121]
plan-refs: []
audit-refs: []
depends-on: []
created: 2026-05-17
---

# IMP-26-00-01: EPIC-26 Hypothesen-Telemetrie ueber 2-4 Wochen Beta-Use auswerten

## Motivation

EPIC-26 wurde am 2026-05-17 als Done/Released geschlossen, weil:

- **H-02** (Klassifikator-Coverage >=90%) ist formal verifiziert per
  `ModelTierClassifier.coverage.test.ts` (9 Tests gruen mit
  expliziter H-02-Assertion).
- **H-05** (Auto-Migration >=95%) ist formal verifiziert per
  `activeModelsToProviders.test.ts` (17 Tests gruen mit explizitem
  H-05-Block).

Die anderen vier Hypothesen sind observational und brauchen aktive
Beta-Nutzung:

- **H-01:** Sonnet 4.6 liefert subjektiv vergleichbare Qualitaet wie
  Opus 4.6 fuer Strategie-/Argumentations-Chats. Validation: Sebastians
  Daily-Use, kein Vorab-Test.
- **H-03:** `consult_flagship`-Eskalations-Rate liegt zwischen 5-15%
  der Auto-Chats. Validation: Cost-Log enthaelt bereits `mode`-Tag;
  Auswertung als Reverse-Engineering ueber 2-4 Wochen.
- **H-04:** Provider-Setup auf <=1 Minute. Validation: Stoppuhr-Test
  bei naechstem Provider-Wechsel.
- **H-06:** Single-Active-Provider als Standard-Modus
  akzeptiert. Validation: subjektive Erfahrung + Per-Turn-Override-
  Nutzungs-Statistik.

## Vorschlag

1. Nach 2-4 Wochen produktiver Nutzung (>= 100 Auto-Sessions):
   Cost-Log abrufen, `mode`-Tag-Verteilung auswerten.
2. Eskalations-Rate berechnen (`consult_flagship`-Calls /
   Auto-Sessions).
3. Falls Drift:
   - **>15% Eskalation:** `consult_flagship`-Prompt-Reminder
     verschaerfen, oder `defaultMainModelTier` per Setting auf
     `flagship` flippen (Rollback-Plan).
   - **<5% Eskalation:** Klassifikator-Pattern auf mehr Mid-Tier-
     Modelle ausweiten oder Prompt-Reminder lockern.
4. Schreib-up: ein kurzer Note in `_devprocess/analysis/` mit
   Telemetrie-Daten + Decision.

## Akzeptanz

- Auswertungs-Note unter `_devprocess/analysis/
  TELEMETRY-EPIC-26-{YYYY-MM-DD}.md`.
- Decision-Record: passt der Default-Tier-Setting? Bleibt
  Sonnet-Default? Brauchen wir ADR-Update?

## Status

Siehe BACKLOG-Row IMP-26-00-01. Erinnerung in 2-4 Wochen
(2026-06-01 bis 2026-06-15).
