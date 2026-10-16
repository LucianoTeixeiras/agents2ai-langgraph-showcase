# Log de decisões

## 2026-10-16 — Planejamento inicial (PR docs/planejamento-inicial)
- **ADR-001** LangGraph como framework (Proposta).
- **ADR-002** Somente dados públicos: DuckDB sintético + BigQuery public datasets (Proposta).
- **ADR-003** LLM OpenAI-compatível via OpenRouter, barato por padrão (Proposta).
- **ADR-004** Observabilidade: trace por nó; LangSmith/OTel opcionais (Proposta).
- **ADR-005** Guardrails de segurança (Proposta).

## Decisões pendentes (Luciano)
- **D-01** Projeto Jira (novo, ou épico em AKIT?). Até lá, commits com `[pendente]`.
- **D-02** Dataset: confirmar `thelook_ecommerce` como schema de referência.
- **D-03** Modelo LLM padrão no OpenRouter (sugestão: o Nemotron já usado, ou outro modelo barato com bom SQL).
- **D-04** Projeto GCP de testes da Agents2AI para billing do BigQuery (com alerta de US$10).
- **D-05** Destino OTel na fase 2 (sugestão: Langfuse, igual ao copilot).
