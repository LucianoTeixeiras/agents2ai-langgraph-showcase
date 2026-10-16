# ADR-004 — Observabilidade: trace por nó em JSON; LangSmith e OTel opcionais

| Campo | Valor |
|-------|-------|
| **Status** | Proposta |
| **Data** | 2026-10-16 |
| **Decisores** | Luciano Teixeira (aceite) · Dev Grok Bot (proposta) |
| **Jira** | pendente — D-01 |
| **Relacionados** | PRD e System Design `docs/contracts/202610/16/`; ADR-003; copilot ADR-009 (OTel) |

## Contexto (2026-10-16)
Recrutadores valorizam ver latência/custo por etapa. Não queremos dependência obrigatória de SaaS nem segredo no CI.

## Decisão
1. Decorador `traced` registra por nó `{"node", "ms", "tokens_in", "tokens_out"}` no campo `trace` do estado.
2. Uma linha JSON `INFO` por execução (logger `showcase`), com `thread_id`, totais e lista de nós; **sem** SQL bruto com valores literais, sem prompts, sem chaves.
3. LangSmith: ativado apenas se `LANGSMITH_TRACING=true` e chave presentes (comportamento nativo do LangChain/LangGraph; **não verificado** nesta data para 1.2.x).
4. OpenTelemetry: fase 2, alinhado ao destino escolhido no copilot (Langfuse, D-05).
5. Evals reportam p50/p95 por nó a partir de `trace`.

## Alternativas consideradas
| Alternativa | Prós | Contras | Por que não |
|-------------|------|---------|-------------|
| **Escolhida:** trace próprio + opcionais | sem dependência, testável | menos visual | — |
| LangSmith obrigatório | UI pronta | SaaS + chave | CI sem segredos |
| OTel já na v1 | padrão | mais código | escopo |

## Consequências
- **Positivas:** métricas testáveis no CI. **Negativas:** sem UI de traces na v1.

## Conformidade
`tests/test_tracing.py` verifica 1 span por nó executado e ausência de chave/PII no log.

## Rastreabilidade
SPEC-LG-01 (campo), SPEC-LG-06 (decorador, log, evals).
