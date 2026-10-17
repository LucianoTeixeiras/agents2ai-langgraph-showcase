# Log de decisões

## 2026-10-16 — Planejamento inicial (PR docs/planejamento-inicial)
- **ADR-001** LangGraph como framework (**Aceita** 2026-10-16).
- **ADR-002** Somente dados públicos: DuckDB sintético + BigQuery public datasets (**Aceita** 2026-10-16).
- **ADR-003** LLM OpenAI-compatível via OpenRouter, barato por padrão (**Aceita** 2026-10-16).
- **ADR-004** Observabilidade: trace por nó; LangSmith/OTel opcionais (**Aceita** 2026-10-16).
- **ADR-005** Guardrails de segurança (**Aceita** 2026-10-16).

## 2026-10-16 — Decisões de Luciano (merge do PR #1)
- ADR-001…005 **aceitas**; licença **MIT** confirmada.
- **D-01** Jira: **concluída** (2026-10-16) — projeto `A2LG` (*Agents2AI LangGraph Showcase*) no mesmo site do AKIT; épico [A2LG-1](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-1) e backlog SPEC-LG-01…09 ([A2LG-2](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-2) … [A2LG-10](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-10); fase 2 [A2LG-9](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-9) bloqueada). Commits/PRs de implementação usam a chave da issue da spec (ex.: `[A2LG-2]` para SPEC-LG-01); `[pendente]` só se não houver issue ligada.
- **D-02** Dataset de referência: `bigquery-public-data.thelook_ecommerce` **confirmado**.
- **D-03** Modelo padrão: Nemotron via OpenRouter, id `nvidia/nemotron-3.5-lightning` (verificado em 2026-10-16; US$ 0,07/M entrada, US$ 0,20/M saída).
- **D-04** **Docker-first:** Fase 1 = protótipo 100% em Docker (compose + DuckDB sintético local, sem GCP). Fase 2 = BigQuery público em projeto GCP de testes da Agents2AI (SPEC-LG-08, Bloqueada). SPEC-LG-05 dividida: HITL + SqliteSaver + CLI (só DuckDB); nova SPEC-LG-07 (Docker).
- **ADR-006** Recuperação de schema por similaridade: DuckDB `FLOAT[n]` + extensão oficial `vss` (HNSW), índice em memória reconstruído no startup, sem persistência experimental; pgvector/vector DB dedicado é o caminho para escala (**Aceita**, Luciano). Spec: SPEC-LG-09.

## 2026-10-17 — Jira validado (Cursor + MCP Atlassian)
- Backlog A2LG conferido contra `docs/contracts/202610/16/spec-lg-*.md` e System Design §10; aceite registrado em [A2LG-11](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-11).

## Decisões pendentes (Luciano)
- **D-05** Destino OTel na fase 2 (sugestão: Langfuse, igual ao copilot).
