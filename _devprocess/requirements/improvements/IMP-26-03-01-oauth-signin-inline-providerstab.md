---
id: IMP-26-03-01
feature: FEAT-26-03
epic: EPIC-26
adr-refs: [ADR-122]
plan-refs: []
audit-refs: []
depends-on: []
created: 2026-05-17
---

# IMP-26-03-01: OAuth-Sign-In inline in ProvidersTab (replace legacy-ModelsTab redirect)

## Motivation

EPIC-26 Welle 2 (PLAN-25) hat den ProvidersTab als Provider-zentrierte
Setup-Oberflaeche eingefuehrt. Fuer Provider-Typen mit OAuth-Auth
(heute: ChatGPT) bleibt der "Authorize"-Pfad ein Stub: der Klick
oeffnet den legacy `ModelsTab` im OAuth-Modus statt den PKCE-Flow
direkt in der Provider-Card abzuwickeln.

Das funktioniert (Setup-Erfolg via Redirect bleibt moeglich), erzeugt
aber zwei UX-Kosten:

- Provider-only-Vision wird aufgeweicht; der User springt mitten im
  Setup in eine andere Tab-Struktur.
- Nach Redirect ist nicht offensichtlich, ob der Setup-Pfad
  abgeschlossen ist; der User muss zurueck in den ProvidersTab.

## Vorschlag

PKCE-Flow direkt in der Provider-Card abwickeln:

1. Authorize-Button in `ProviderDetailModal` (oder embedded card)
   loest den `ChatGptOAuthService`-Loopback aus.
2. Status-Anzeige (warten/zustimmen/abgeschlossen) inline.
3. Nach erfolgreichem Callback: `discoveredModels` neu fetchen,
   `activeProviderId` setzen, Card als gruen markieren.
4. Refresh-Button neben Tier-Slot funktioniert weiterhin fuer
   Re-Discovery nach Token-Refresh.

Funktional aequivalent zum heutigen Pfad, aber Tab-Sprung entfaellt.

## Implementation pointer

- `src/ui/settings/ProvidersTab.ts` (UI)
- `src/ui/settings/ProviderDetailModal.ts` (sofern existent; sonst neu)
- `src/core/auth/ChatGptOAuthService.ts` (Service ist da; wiederverwenden)
- Vergleich zum legacy-Flow in `src/ui/settings/ModelsTab.ts`-OAuth-Pfad.

## Akzeptanz

- Provider-Setup mit ChatGPT-OAuth komplett innerhalb von ProvidersTab.
- Keine Regression auf den Legacy-Pfad (User mit alten Configs
  funktionieren weiterhin).
- Test: Manueller PKCE-Sign-In-Flow von Scratch.

## Status

Siehe BACKLOG-Row IMP-26-03-01.
