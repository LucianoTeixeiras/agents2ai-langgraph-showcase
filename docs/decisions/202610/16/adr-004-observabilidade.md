# ADR-004 — Observabilidade: trace por nó em JSON; LangSmith e OTel opcionais

| Campo | Valor |
|-------|-------|
| **Status** | Aceita (2026-10-16, Luciano) |
| **Data** | 2026-10-16 |
| **Decisores** | Luciano Teixeira (aceite) · Dev Grok Bot (proposta) |
| **Jira** | [A2LG-11](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-11) (aceite ADRs 001–006) |
| **Relacionados** | PRD e System Design `docs/contracts/202610/16/`; ADR-003; copilot ADR-009 (OTel) |

## Contexto (2026-10-16)
Recrutadores valorizam ver latência/custo por etapa. Não queremos dependência obrigatória de SaaS nem segredo no CI.

## Decisão
1. Decorador `traced` registra por nó `{"node", "ms", "tokens_in", "tokens_out"}` no campo `trace` do estado.
2. Uma linha JSON `INFO` por execução (logger `showcase`), com `thread_id`, totais e lista de nós; **sem** SQL bruto com valores literais, sem prompts, sem chaves.
3. LangSmith: ativado apenas por env, comportamento nativo (verificado em 2026-10-16 no código de `langsmith==0.14.11`, dependência transitiva de `langchain-core==1.6.11`/`langgraph==1.2.15`): `langsmith.utils.tracing_is_enabled()` retorna `True` quando `LANGSMITH_TRACING_V2` ou `LANGSMITH_TRACING` (fallback: prefixo `LANGCHAIN_`) vale exatamente `"true"`. Demais variáveis: `LANGSMITH_API_KEY`, `LANGSMITH_PROJECT` (opcional), `LANGSMITH_ENDPOINT` (padrão `https://api.smith.langchain.com`). O código **não** lê essas variáveis; só as documenta no `.env.example` (vazias, `LANGSMITH_TRACING=false`). Os testes fazem `monkeypatch.delenv` de `LANGSMITH_TRACING`, `LANGSMITH_TRACING_V2`, `LANGCHAIN_TRACING_V2` para nunca enviar traces no CI.
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
