---
id: FIX-23-09-03
epic: EPIC-23
adr-refs: []
plan-refs: []
depends-on: [FIX-23-09-01]
created: 2026-06-20
---

# FIX-23-09-03: RelayClient forwards prompts/list + prompts/get

## Symptom

Nach FIX-23-09-01 ist der neutrale `vault-operator-context`-Prompt fuer Claude-Desktop-User ueber `prompts/list` waehlbar. Claude.ai-Connector-User gehen aber ueber den Cloudflare-Worker-Relay (siehe ADR-55, `src/mcp/RelayClient.ts` + `relay/src/index.ts`), und dieser leitet derzeit nur `initialize`, `tools/list`, `tools/call` und `resources/list` an `McpBridge.handleJsonRpc` weiter. `prompts/list` und `prompts/get` fallen in den Default-Branch und antworten mit leerem `{}`.

Resultat: Im Claude.ai-Connector-Mode taucht der `vault-operator-context`-Prompt nicht im "/"-Menue auf. Solange der User den Prompt nicht waehlt, sieht das LLM gar keinen Vault-Kontext mehr (bevor FIX-23-09-01 lieferte der Auto-Inject ihn ungefragt aus -- das WAR aber gerade das Problem).

## Plan

1. `RelayClient`-Switch um `prompts/list` und `prompts/get` ergaenzen, beide unveraendert an `McpBridge.handleJsonRpc` durchreichen.
2. Spiegel in `relay/src/index.ts` (Cloudflare-Worker) ziehen, falls dort auch eine Methoden-Whitelist haengt.
3. Test: `RelayClient.test.ts` ergaenzen um zwei Cases, die einen Mock-Request fuer `prompts/list` / `prompts/get` schicken und das forwarded Payload erwarten.

## Status

Open. Bis das durch ist, bleibt Claude.ai-Connector-Mode ohne Auto-Context -- bewusste Trade-off-Akzeptanz, weil Auto-Inject die Injection-Quelle war.
