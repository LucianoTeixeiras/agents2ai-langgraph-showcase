# ADR-001 — Adotar LangGraph como framework de orquestração

| Campo | Valor |
|-------|-------|
| **Status** | Proposta |
| **Data** | 2026-10-16 |
| **Decisores** | Luciano Teixeira (aceite) · Dev Grok Bot (proposta) |
| **Jira** | pendente — D-01 |
| **Relacionados** | PRD e System Design `docs/contracts/202610/16/`; ADR-002…005 |

## Contexto (2026-10-16)
O produto principal usa Pydantic AI (ADR-007 do copilot). A pesquisa de frameworks do Learning Bot recomenda LangGraph como padrão de mercado para agentes com estado, HITL e durabilidade. Esta vitrine precisa demonstrar exatamente essas capacidades.

## Decisão
1. O agente é um `StateGraph` do pacote `langgraph`, fixado em `langgraph==1.2.15` (versão verificada no PyPI e testada localmente em 2026-10-16).
2. Estado é um `TypedDict` (`AgentState`) com reducer apenas em `trace`.
3. HITL usa `langgraph.types.interrupt` + `Command(resume=...)`; checkpointers `InMemorySaver` (testes) e `SqliteSaver` (`langgraph-checkpoint-sqlite==3.1.2`, CLI).
4. LangChain é usado **só** como cliente de chat (`langchain-openai`) atrás do protocolo `LLMClient`; nenhuma chain/agent pronto do LangChain.
5. Upgrades de LangGraph exigem PR próprio com os testes de HITL verdes.

## Alternativas consideradas
| Alternativa | Prós | Contras | Por que não |
|-------------|------|---------|-------------|
| **Escolhida:** LangGraph | padrão de mercado, checkpoint/interrupt nativos | API evolui rápido | — |
| Pydantic AI | já dominado no copilot | não demonstra skill nova | vitrine seria redundante |
| Google ADK | deploy GCP | menos procurado em vagas | coberto pela spike ADK do copilot |
| LangGraph Platform | deploy gerenciado | custo, lock-in | fora de escopo |

## Consequências
- **Positivas:** demonstra skill de mercado; HITL e retomada nativos.
- **Negativas:** dependência transitiva de `langchain-core`.
- **Riscos residuais:** breaking changes → versão fixada no lock + testes de HITL.

## Conformidade
`uv.lock` fixa a versão; `tests/test_hitl.py` cobre interrupt/resume; revisão rejeita uso de `create_react_agent`/chains.

## Rastreabilidade
Specs SPEC-LG-01, 04, 05.
