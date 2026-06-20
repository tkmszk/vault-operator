---
id: FIX-23-09-02
epic: EPIC-23
adr-refs: []
plan-refs: []
depends-on: [FIX-23-09-01]
created: 2026-06-20
---

# FIX-23-09-02: execute_vault_op catch-all schema

## Symptom

In GitHub-Issue [#46](https://github.com/pssah4/vault-operator/issues/46) flaggt der MCP-Client zusaetzlich zu FIX-23-09-01:

> "The full `execute_vault_op` catalog includes things like `execute_command`, `invoke_mcp_server`, `use_mcp_tool`, `update_soul`, and `search_history`, far beyond anything a note-renaming task needs."

Das Tool nimmt ein freies `operation: string` plus `params: object` und routet das dynamisch durch den ToolRegistry. Per MCP-Threat-Model ist das ein generisches "execute arbitrary thing"-Tool ohne eingrenzbares Schema. Fremde LLMs/Clients koennen die echte Operations-Liste erst zur Laufzeit erfahren und haben keine deklarative Surface.

## Plan

1. Allowlist pro `source_interface` einfuehren (welcher Connector darf welche Op aufrufen).
2. Statt eines monolithischen `execute_vault_op` pro relevanter Op eine eigene MCP-Tool-Definition mit echtem Schema generieren (`tools/list` zeigt sie einzeln; LLM kann sie typisiert aufrufen).
3. Agent-internal Tools (vgl. `AGENT_INTERNAL_TOOLS`) bleiben harte Deny-Liste.
4. Backward-Compat: `execute_vault_op` bleibt fuer interne Aufrufer (`source_interface = 'obsilo'`) erhalten, externe Clients sehen nur noch die generierten typed Tools.

## Status

Open. Folge-FIX zu FIX-23-09-01. Issue #46 ist mit FIX-23-09-01 mitigiert (kein Prompt-Injection mehr), `execute_vault_op` ist aber weiterhin ein Catch-All bis zu diesem Fix.
