---
id: IMP-24-08-01
feature: FEAT-24-08
epic: EPIC-24
adr-refs: [ADR-114, ADR-113, ADR-12]
plan-refs: []
audit-refs: []
depends-on: []
created: 2026-05-17
---

# IMP-24-08-01: Kumulatives Token-/Kosten-Budget pro Task mit Pause+Rueckfrage

## Motivation

Aus FEAT-24-08 / ADR-114 zurueckgestellt. ADR-114 sieht drei additive
Autonomie-Mechaniken vor; in v2.11.5-beta.2 wurde nur Mechanik B
(Steering-Hook) implementiert. Mechanik A (Budget) wartet auf einen
spaeteren Pass.

Der Agent hat heute zwei Bremsen gegen Entgleisen: einen Iterations-Cap
(Default 25, plus Soft-Limit bei 60 %) und einen Wiederholungs-Detektor
(ADR-06). Was fehlt: ein kumulatives Token-/Kosten-Budget pro Task. Ein
Iterations-Cap deckelt nur die Schleifendurchlaeufe, nicht den Aufwand
pro Durchlauf. Ein einzelner Turn, der vier grosse Dateien liest und
dann durch fuenf Iterationen mit der angehaeuften History weiterlaeuft,
liegt innerhalb von 25 Iterationen, kostet aber 100k-plus Tokens.

Konkreter Praezedenzfall im ADR-114-Kontext: ein Chat, der rund 42 EUR
ueber sechs Turns verbrannte, ohne dass jemals ein Limit gegriffen
haette.

## Status quo nach v2.11.5-beta.2

- `totalInputTokens` und `totalOutputTokens` werden in `AgentTask.ts`
  bereits pro Lauf akkumuliert (Z. 473, 559, 920, 1297).
- `onApprovalRequired`-Callback-Pfad fuer Pause+Rueckfrage existiert
  (genutzt von Write-Tools fuer User-Bestaetigung).
- Steering-Hook (Mechanik B) ist seit v2.11.5-beta.2 verdrahtet; der
  Budget-Check kann denselben Iteration-Boundary-Pfad nutzen.

## Vorschlag

1. **Settings**:
   - `taskTokenBudget: number` (Default 200_000)
   - `taskCostBudgetEur: number` (Default 5.0, alternativ statt
     Token-Limit)
   - `taskBudgetWarnRatio: number` (Default 0.75)
   - Pfade in WRITABLE_PATHS der update_settings-Whitelist.
2. **Budget-Check an Iteration-Boundary** in `AgentTask.ts` (gleicher
   Ort wie der Steering-Drain, direkt nach dem Soft-Limit-Block): bei
   Erreichen des Warnschwellwerts pausieren und `onApprovalRequired`
   mit drei Optionen ausloesen ("weitermachen / Limit fuer diesen
   Task verdoppeln / abbrechen").
3. **Subtask-Anteilig**: Subtask-Tokens zaehlen auf das Task-Budget
   mit; das Per-Call-Budget aus ADR-113 (`subtaskTokenBudget`) bleibt
   eine zusaetzliche, engere Grenze fuer den einzelnen Subtask.
4. **Cost-Schaetzung**: aktuell ueber `ModelPricing` und die in
   `AgentTask` schon akkumulierten Token-Counts. Cache-Discounts noch
   nicht abgezogen -- konservativ (eher zu frueh warnen als zu spaet).

## Akzeptanz

- Lauf, der die Warnschwelle ueberschreitet, pausiert mit klarer
  Rueckfrage.
- "Limit verdoppeln" persistiert pro Task (nicht pro Setting).
- Einfache 1-3-Iterationen-Tasks beruehren nie die Warnschwelle bei
  Default-Werten.
- Subtasks werden korrekt mitgezaehlt.

## Status

Zurueckgestellt 2026-05-17 (User-Entscheidung: "Budgetfrage ist jetzt
noch zu frueh"). Siehe BACKLOG-Row IMP-24-08-01.
