# FIX-04-03-07: DeepSeek deepseek-reasoner 400 -- reasoning_content not echoed back

**Prioritaet:** P1 (User-Sichtbar, Issue #38, Plugin unbenutzbar mit DeepSeek-Reasoner sobald Tools laufen)
**Datei:** `src/api/providers/openai.ts` (Wire-Format), `src/api/types.ts` (ContentBlock-Union), `src/core/AgentTask.ts` (History-Assembly), `src/api/providers/anthropic.ts` + `bedrock.ts` (Cross-Provider-Safety), `src/core/history/ConversationStore.ts` + `src/ui/AgentSidebarView.ts` (UI-Replay)
**Feature-Bezug:** EPIC-04 (Providers), FEAT-04-03 (OpenAI-kompatible Provider-Familie)
**Entdeckt:** 2026-05-28 (Issue [#38](https://github.com/pssah4/vault-operator/issues/38), DavidHinton-xcpc)

---

## Problem

Mit DeepSeek `deepseek-reasoner` ueber die OpenAI-kompatible Provider-Konfiguration (`config.type === 'custom'`, `baseUrl: https://api.deepseek.com`) liefert die DeepSeek-API nach dem ersten Tool-Call:

```
400 The `reasoning_content` in the thinking mode must be passed back to the API.
```

Reproduktion (aus Issue):
- Provider: Custom OpenAI-kompatibel, baseUrl `https://api.deepseek.com`, model `deepseek-reasoner`.
- Prompt: `我想利用plugin制作思维导图. 你有什么建议来帮助你更好的操作?` (oder beliebiger Prompt der Tool-Use ausloest).
- Beobachtet: Reasoning-Anzeige sichtbar, 2 Tools (search_files, list_files) laufen, danach 400 — Plugin haengt.

Reiner Chat ohne Tool-Calls ("who are you") funktioniert. Der Fehler entsteht ausschliesslich im Tool-Use-Rueckkanal.

## Root Cause Analyse

DeepSeeks `deepseek-reasoner` (deren DeepSeek-R1-Variante via OpenAI-kompatibler API) liefert pro Antwort zwei Felder:

- `reasoning_content`: Chain-of-Thought (CoT) der intern produziert wurde.
- `content`: Die eigentliche User-sichtbare Antwort (oder bei Tool-Use: leer + `tool_calls`).

**DeepSeek-API-Vertrag:** Wenn die Assistant-Message ein `tool_calls`-Feld enthaelt und beim naechsten Request mit den Tool-Results zurueckkommt, **muss** das urspruengliche `reasoning_content` mitgeschickt werden. Sonst 400.

Andere OpenAI-kompatible Backends (`openai`, `azure`, `gemini`, OpenRouter ohne Reasoner) haben diese Anforderung **nicht** — sie ignorieren `reasoning_content` als unknown field oder kennen es gar nicht.

**Code-Pfad mit dem Bug:**

1. **Stream-Capture funktioniert:** [`src/api/providers/openai.ts:352-356`](../../../src/api/providers/openai.ts#L352-L356) liest `reasoning_content` aus dem OpenAI-kompatiblen Stream-Delta und yieldet `{ type: 'thinking', text }` als Stream-Chunk. UI zeigt "Reasoning…" live.

2. **Storage fehlt:** [`src/core/AgentTask.ts:947-948`](../../../src/core/AgentTask.ts#L947-L948) ruft `onThinking`-Callback fuer die UI — und **verwirft den Text danach**. Beim Assemble der Assistant-Message ([Z. 993-998](../../../src/core/AgentTask.ts#L993-L998)) werden nur `textParts` (sichtbarer Text) und `toolUses` aufgenommen.

3. **Internes Modell laesst keinen Platz:** [`src/api/types.ts:59-63`](../../../src/api/types.ts#L59-L63) — `ContentBlock`-Union kennt nur `text`, `image`, `tool_use`, `tool_result`. Kein Slot fuer Reasoning zwischen Turns.

4. **Wire-Konstruktion ignoriert es:** [`src/api/providers/openai.ts:439-507`](../../../src/api/providers/openai.ts#L439-L507) `convertMessages` rekonstruiert die Assistant-Message als `{ role: 'assistant', content: textOnly, tool_calls: [...] }` — ohne `reasoning_content`. Folge-Request → 400.

## Auswirkung

- **Funktional:** Hoch. DeepSeek-Reasoner ist mit Tools komplett unbenutzbar, sobald irgendein Tool ausgeloest wird. User muss auf `deepseek-chat` (non-reasoner) wechseln.
- **UX:** Hoch. Fehlermeldung ist klar, aber Plugin haengt nach der ersten Tool-Iteration. Conversation wird unbrauchbar.
- **Wirtschaftlich:** Hoch. DeepSeek-Reasoner ist eines der guenstigsten Reasoning-Modelle am Markt; ohne Tool-Unterstuetzung verliert man fast den ganzen Wert des Plugins fuer DeepSeek-User.

## Loesungsansatz

### Kern-Design (3 Bausteine)

**Baustein 1: ThinkingBlock als first-class ContentBlock.**
`{ type: 'thinking', text: string }` wird zur `ContentBlock`-Union hinzugefuegt. Persistiert automatisch via `ConversationStore` (JSON-Serialisierung), wird von `MicroCompactor` und `sanitizeHistoryForApi` nicht angefasst (die kuemmern sich nur um `tool_result`- bzw. orphan-`tool_use`-Faelle). `estimateTokens` bekommt eine Klausel damit Condensing-Threshold nicht zu spaet triggert.

**Baustein 2: Storage gated via Stream-Chunk-Flag.**
`ApiStreamChunk.thinking` bekommt optional `requiresPassback?: boolean`. Nur der OpenAI-kompatible Provider setzt das Flag. AgentTask akkumuliert ThinkingBlocks ausschliesslich wenn das Flag gesetzt ist. Damit:
- Anthropic-Provider-Verhalten ist **bit-genau** wie vorher (kein Storage).
- Speicherlast strikt auf Reasoner-Modelle ueber OpenAI-Kompatibel begrenzt.

**Baustein 3: Wire-Side Allow-List + Last-Assistant-Only.**
Im OpenAI-Provider `convertMessages`:
- **Allow-List:** Nur `config.type ∈ {custom, ollama, lmstudio}` echoed `reasoning_content`. `openai`, `azure`, `gemini`, `openrouter` bleiben unangetastet (Regressions-Schutz; OpenRouter hat eigene Reasoning-Mechanik via Top-Level `{reasoning: {max_tokens}}`-Param).
- **Last-Assistant-Only:** Nur die letzte Assistant-Message mit `tool_use`-Bloecken bekommt `reasoning_content` auf dem Draht. Aeltere ThinkingBlocks in der History werden stillschweigend verworfen. Folgt DeepSeeks Multi-Round-Konvention ("Do not include reasoning_content of previous rounds"). Maximaler Overhead pro Request: ein Turn Reasoning (~1-5k Tokens), konstant ueber Session-Laenge.
- **Empty-Guard:** Field nur setzen wenn String non-empty (Schutz vor abort-mid-stream).
- **50k-Cap:** Reasoning > 50.000 chars wird auf 50k getruncate mit Trailer (Runaway-Schutz).

### Defensive Drops in Anthropic + Bedrock

Beide Provider werfen aktuell `Error("Unknown content block type")` bei unbekannten Bloecken. Sobald ThinkingBlocks in einer geladenen History vorkommen (Cross-Provider-Switch DeepSeek → Anthropic), wuerde das die Conversation toetlich crashen. Fix: Pre-Filter `msg.content.filter(b => b.type !== 'thinking')` vor dem `.map()`. Keine Switch-Erweiterung, eine Zeile pro Provider.

### UI-Replay

UI rendert aus `UiMessage[]` (flat text + optional `toolStepsHtml`), nicht aus `MessageParam.content`. Analog ein neues Feld `reasoningText?: string` auf `UiMessage`. AgentSidebarView akkumuliert die Live-Stream-Reasoning-Texte parallel zur Anzeige, persistiert sie am Turn-Ende. Beim Reload rendert ein neuer Helper `renderReasoningBlock` eine gefaltete "Reasoning…"-Bubble vor der Assistant-Antwort (analog `toolStepsHtml`-Rendering). Backward-kompatibel — alte UiMessages ohne Feld rendern wie bisher.

## Akzeptanzkriterien

### Code

- [ ] `ContentBlock`-Union enthaelt `{ type: 'thinking', text: string }`.
- [ ] `ApiStreamChunk.thinking` hat optional `requiresPassback?: boolean`.
- [ ] OpenAI-Provider Stream-Capture setzt `requiresPassback: true` beim `reasoning_content`-Lesen.
- [ ] AgentTask akkumuliert ThinkingBlocks **nur** wenn `requiresPassback === true`.
- [ ] OpenAI-Provider `convertMessages` Allow-List = `{custom, ollama, lmstudio}`; alle anderen `config.type`-Werte bekommen **kein** `reasoning_content` auf den Draht.
- [ ] Nur die letzte Assistant-Message mit `tool_use` bekommt `reasoning_content`; aeltere werden verworfen.
- [ ] Empty-Guard: `reasoning_content` wird nicht gesetzt wenn Text leer.
- [ ] 50k-Cap mit Trailer aktiv.
- [ ] Anthropic + Bedrock `convertMessages` haben Pre-Filter fuer thinking-Blocks (kein throw).
- [ ] `estimateTokens` zaehlt ThinkingBlock-Inhalt mit chars/4.
- [ ] `UiMessage.reasoningText?` als optional Feld in ConversationStore.
- [ ] AgentSidebarView akkumuliert + persistiert + rendert Reasoning-Bubble.
- [ ] CSS `.agent-reasoning-bubble` mit collapsed/expanded-State.

### Tests (Vitest)

- [ ] OpenAI-Provider: convertMessages mit thinking + tool_use auf last msg + config.type=custom → reasoning_content auf wire.
- [ ] OpenAI-Provider: thinking auf older msg → kein reasoning_content.
- [ ] OpenAI-Provider: thinking ohne tool_use → kein reasoning_content.
- [ ] OpenAI-Provider: multiple thinking → konkateniert.
- [ ] OpenAI-Provider: > 50k chars → truncated mit Trailer.
- [ ] OpenAI-Provider: empty thinking → kein Feld auf wire.
- [ ] OpenAI-Provider: config.type=openai/azure/openrouter/gemini → kein reasoning_content (Allow-List).
- [ ] OpenAI-Provider: streaming `reasoning_content`-Delta → `{type:'thinking', requiresPassback:true}`.
- [ ] Anthropic-Provider: convertMessages mit ThinkingBlock in History → strip, kein throw.
- [ ] Bedrock-Provider: analog Anthropic.
- [ ] AgentTask: thinking-Chunk mit requiresPassback=true → in Assistant-Message gespeichert.
- [ ] AgentTask: thinking-Chunk ohne requiresPassback → nicht gespeichert.
- [ ] AgentTask: estimateTokens zaehlt ThinkingBlock.
- [ ] sanitizeHistoryForApi: thinking-Blocks bleiben unberuehrt.

### Live-Verifikation

- [ ] DeepSeek `deepseek-reasoner` mit Tool-Trigger-Prompt → vollstaendige Antwort, kein 400 (Primary Acceptance, Issue #38).
- [ ] Anthropic Direct + Tool-Calls → unveraendert, alle bestehenden Conversations funktionieren.
- [ ] OpenAI native (gpt-4) + Tool-Calls → unveraendert (Allow-List verhindert reasoning_content auf wire).
- [ ] Cross-Provider-Switch: DeepSeek-Conversation mit ThinkingBlocks → wechsle auf OpenAI native, weiterchatten → kein 400.
- [ ] Persistence: Plugin reload, Conversation laden, weiterchatten → funktioniert; UI zeigt Reasoning-Bubbles.
- [ ] Token-Bilanz: Nach 5 Tool-Iterations konstanter Reasoning-Anteil pro Request (~1-5k), keine Akkumulation.

## Out of Scope

- **Anthropic Extended-Thinking + Tool-Use round-trip mit signierten Bloecken.** Latent existierender Bug, niemand hat sich beschwert, braucht eigenes Schema fuer `thinking_signed`-ContentBlock und Anthropic-spezifische Signatur-Logik. Eigene FIX wenn ein User das anfordert.
- **OpenRouter zur Allow-List ergaenzen.** OpenRouter hat eine eigene `{reasoning: {max_tokens}}`-Mechanik fuer Claude. Verifikation und potenzielle Erweiterung ist Phase-2-Follow-up wenn OpenRouter-DeepSeek-Reasoner-Nutzer das anfragen.
- **Settings-Toggle "Send reasoning back to API" pro Provider.** Aktuell hardcoded via Allow-List. Optional Power-User-Feature fuer spaeter.
- **Pre-Compaction-Hook der historic ThinkingBlocks strippt (Speicher-Hygiene).** Aktuell akzeptabel — pro 50-Turn-Session ~150KB extra. Condensing summarisiert Mittelfeld automatisch weg.

## Quellen

- Issue #38: https://github.com/pssah4/vault-operator/issues/38
- DeepSeek API Doku zu deepseek-reasoner: Multi-Round-Konvention sagt "do not include reasoning_content of previous rounds"; bei Tool-Use ist das aktuelle reasoning_content fuer den naechsten Request erforderlich.
- Plan: lokal unter `~/.claude/plans/warm-tickling-scone.md` (Rev 2 nach kritischem Review: Allow-List, UI-Replay, ET+TU out of scope).
