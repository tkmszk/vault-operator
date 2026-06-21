---
id: FIX-19-99-03
epic: EPIC-19
feature: FEAT-19-13
adr-refs: []
plan-refs: []
depends-on: []
audit-refs: [STABILITY-AUDIT-v2.14.0-2026-06-21]
created: 2026-06-21
---

# FIX-19-99-03: Tension-Detection verdrahten + ModelDiscoveryService Pricing-Felder

## Symptom

Stabilitaets-Audit 2026-06-21:

1. **FEAT-19-13 Tension-Detection** ist als Released markiert, der DetectorHook in `IngestDeepTool.ts:225` war aber hardcoded `const tensionDetector: TensionDetector | undefined = undefined`. Folge: `DeepIngestPipeline` sieht keinen Detector und ueberspringt den Marker-Step. Feature wirkt im Plan, aber nie in der Praxis.

2. **FEAT-26-02 Pricing/Capability-Fallback** ist als Released markiert; `ModelDiscoveryService` bekommt aber nur `{id, displayName}` aus `fetchProviderModels`. `RawDiscoveredModel` deklariert `contextWindow`, `maxOutputTokens`, `pricingPromptUsd`, `pricingCompletionUsd` -- die werden nie gesetzt. OpenRouter liefert genau diese Felder inline mit `/v1/models`, aber die Mapping-Stufe wirft sie weg.

## Fix

### FIX-19-13-01 (Tension-Detector)

- `IngestDeepTool.ts`: Detector wird konstruiert wenn `semanticIndex` UND `helperApi` verfuegbar sind (defensive Degradation sonst).
- `candidateLookup`: `semanticIndex.search(claim, topK)` -> `CandidateNote[]` (path, summary=excerpt, excerpt).
- `classifier`: helper-api `classifyText(prompt)`, parsed Antwort als `{relationship, targetNotePath, confidence, rationale}`. Bei classify-Fehler -> `{relationship: 'neutral', confidence: 0}` (Pipeline-skip).
- Pre-fix: `undefined` -> kein Marker. Post-fix: tatsaechliche LLM-Klassifikation, Marker erscheint im Sense-Making-Note wo die Konfidenz >= Threshold ist (Default 0.6 in TensionDetector).

### FIX-26-99-04 (ModelDiscovery Pricing)

- `ApiModelEntry` in `testModelConnection.ts` um `context_length`, `pricing.prompt`, `pricing.completion`, `top_provider.context_length`, `top_provider.max_completion_tokens` erweitert.
- Neue `FetchedModelEntry`-Shape: `{ id, label, contextWindow?, maxOutputTokens?, pricingPromptUsd?, pricingCompletionUsd? }`.
- `fetchProviderModels(...)` returnt jetzt `FetchedModelEntry[]`. OpenRouter-Branch parst pricing (string oder number) via `parseFloat`, mapped `top_provider.context_length` und `top_provider.max_completion_tokens`. Andere Provider lassen die optionalen Felder undefined.
- `main.ts` ModelDiscoveryService-Fetcher reicht alle Felder als `RawDiscoveredModel` durch (Audit hatte das als "dead code path" markiert).

## Tests

Keine neuen Unit-Tests in dieser Welle (zwei isolierte Wiring-Diffs). Volle Suite 2971 passing + 1 expected fail. tsc clean. Build clean.
