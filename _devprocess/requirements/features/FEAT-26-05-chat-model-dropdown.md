---
id: FEAT-26-05
title: Chat-Model-Dropdown-Refactor
epic: EPIC-26
priority: P0
date: 2026-05-15
related: BA-27
adr-refs: []
plan-refs: []
depends-on: [FEAT-26-01, FEAT-26-02, FEAT-26-03]
---

# FEAT-26-05: Chat-Model-Dropdown-Refactor

## Description

Das heutige Chat-Header-Dropdown listet alle Modelle aller konfigurierten Provider flach. Nach EPIC-26 wird es ersetzt durch ein Dropdown mit "Auto" als Default plus den Modellen des aktiven Providers als Override-Optionen.

Verhalten:
- **"Auto" (Default):** Advisor-Pattern ist aktiv. Hauptloop läuft auf mid-Tier, `consult_flagship`-Tool ist registriert.
- **Override mit explizitem Modell:** Loop läuft fix auf dem gewählten Modell. `consult_flagship` ist für diesen Turn nicht registriert (Tool-Schema-Filter).

Override-Scope: pro Turn. Der User kann situativ entscheiden, das Modell für die nächste Frage zu wechseln, ohne den Chat zu beenden. Der Conversation-Kontext (History, Tool-Results) bleibt erhalten und wird voll an das gewählte Modell geschickt. Beim nächsten Send wird der Dropdown-Wert erneut gelesen.

Beim Provider-Wechsel im Settings-Tab wird das Dropdown automatisch neu gefüllt mit den Modellen des neu aktiven Providers, "Auto" bleibt der Default.

Quelle: BA-27 Sektion 7.1 Welle 5. QA-Decisions 7 (Auto + Overrides) und 8 (Per-Turn).

## Benefits Hypothesis

Heute wählt der User pro Chat aus 10+ Modellen über Provider hinweg. Diese Wahl ist a) zu komplex (welches Modell wofür?), b) statisch pro Chat (Modellwechsel mitten im Chat braucht Mode-Switch), c) blockiert die Auto-Routing-Vision. Wenn der Dropdown auf "Auto + Provider-Modelle" reduziert wird, hat der User die Kontrolle (Override-Option), aber kein Setup-Friction mehr (Default funktioniert).

## User Stories

- **US-05-01 (P1 Sebastian, JTBD-5):** Als Power-User möchte ich für eine spezifische Frage das Modell pro Turn überschreiben können, damit ich situativ entscheiden kann (z.B. "diese Architektur-Frage braucht Opus, nächste Routine-Frage wieder Auto").
- **US-05-02 (P1 Sebastian):** Als Power-User möchte ich nur die Modelle des aktiven Providers im Dropdown sehen, damit ich nicht versehentlich Cross-Provider-Cost erzeuge.
- **US-05-03 (P2 Knowledge-Worker):** Als Standard-User möchte ich, dass "Auto" als Default funktioniert und das Plugin selbst entscheidet, damit ich keine Modell-Wahl treffen muss.
- **US-05-04 (P1 Sebastian):** Als Power-User möchte ich, dass mein bisheriger Conversation-Kontext beim Modellwechsel erhalten bleibt, damit ich nicht den Chat neustarten muss.

## Success Criteria

1. Das Chat-Header-Dropdown zeigt als ersten Eintrag "Auto" und darunter die Modelle des aktiven Providers.
2. Bei "Auto" läuft das Advisor-Pattern: Hauptloop auf mid-Tier, `consult_flagship` registriert. Im Cost-Log sehe ich `model=auto(mid:<id>)` als Hauptmodell.
3. Bei explizitem Override (z.B. "Claude Opus 4.7") läuft der Loop auf genau diesem Modell, `consult_flagship` ist nicht registriert.
4. Der Override gilt für genau den nächsten Turn. Beim übernächsten Send wird das Dropdown erneut gelesen.
5. Der Conversation-Kontext (History, Tool-Results, Memory) bleibt beim Modellwechsel vollständig erhalten und wird an das neue Modell geschickt.
6. Beim Provider-Wechsel im Settings-Tab wird das Chat-Dropdown automatisch aktualisiert mit den Modellen des neuen aktiven Providers. "Auto" bleibt selektiert.
7. Wenn ein einzelner Tier-Slot leer ist (z.B. kein flagship), zeigt der Dropdown im Auto-Modus einen Hinweis ("Advisor pattern disabled, flagship slot empty").
8. Die heutigen Modelle des Plugins aus anderen Providern sind im Chat-Dropdown nicht sichtbar (Single-Active-Provider-Disziplin).

## Technical NFRs

- **Per-Turn-Resolution:** API-Handler wird pro Send-Click neu gebaut. Override-Wert wird beim Send-Event gelesen, nicht bei Chat-Init gecacht.
- **Cache-Verhalten bei Modellwechsel:** Erster Turn nach Wechsel zahlt Cache-Write-Kosten (one-time, akzeptabel). Cache-Prefix unterscheidet sich pro Modell.
- **Tool-Registration-Filter:** bei explizitem Override wird `consult_flagship` aus dem Tool-Schema entfernt für den Turn (Filter, nicht Tool-Deletion).
- **UI-Updates:** Dropdown-Inhalte werden via Reactive-Subscription auf `settings.activeProviderId` aktualisiert.
- **History-Konsistenz:** Conversation-History wird unverändert übergeben. Provider-Wire-Format-Konvertierung (Anthropic vs OpenAI) übernimmt der API-Handler.
- **Telemetrie:** Cost-Log zeigt explizit ob es ein Auto-Turn oder Override-Turn war: `mode=auto` vs `mode=override(<id>)`.

## ASRs

- **ASR-CRIT-01:** Per-Turn-API-Handler-Resolution muss schnell sein (≤50ms). Kein synchroner Provider-API-Call beim Send-Click.
- **ASR-MOD-01:** Tool-Schema-Filter (Disable consult_flagship bei Override) ist deterministisch und cache-freundlich.
- **ASR-MOD-02:** Dropdown-Items werden reactive aus dem Plugin-State berechnet, kein periodisches Refresh.

## Definition of Done

- [ ] Chat-Header-Dropdown refactored: "Auto" + aktive-Provider-Modelle
- [ ] Per-Turn-Override-Resolution im Send-Handler
- [ ] Tool-Registration-Filter für `consult_flagship` bei Override
- [ ] Reactive-Update bei Provider-Wechsel in Settings
- [ ] Empty-Tier-Slot-Indikator ("Advisor pattern disabled")
- [ ] Cost-Log-Erweiterung um `mode`-Field
- [ ] Tests: Auto-Modus, Override-Modus, Modellwechsel-im-Chat, Provider-Wechsel-im-Settings, Empty-Flagship-Slot
- [ ] Live-Messlauf [AWAITING RE]: Mode-Wechsel im laufenden Chat, Kontext-Konsistenz, Cost-Anzeige korrekt

## Validation (Critical Hypothesis H-06)

H-06 sagt: User akzeptiert Single-Active-Provider als Standard-Modus, Override-Dropdown reicht für situative Modell-Wahl. Validation in Beta-Phase via User-Feedback.

## Out-of-Scope

- Cross-Provider-Modell-Auswahl im Chat-Dropdown (verboten, siehe Single-Active-Provider)
- Per-Message-Override (User wählt für jede einzelne Message ein anderes Modell)
- Modell-Wechsel mitten in einem Tool-Use-Loop (innerhalb eines Turns ist das Modell stabil)
- Custom-Default-Anzeige außer "Auto" (Plugin-Wide-Setting)
