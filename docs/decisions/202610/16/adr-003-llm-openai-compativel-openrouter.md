# ADR-003 — Provedor de LLM OpenAI-compatível (OpenRouter), barato por padrão

| Campo | Valor |
|-------|-------|
| **Status** | Proposta |
| **Data** | 2026-10-16 |
| **Decisores** | Luciano Teixeira (aceite) · Dev Grok Bot (proposta) |
| **Jira** | pendente — D-01 |
| **Relacionados** | PRD e System Design `docs/contracts/202610/16/`; ADR-001, ADR-004 |

## Contexto (2026-10-16)
Luciano é sensível a custo e já usa OpenRouter. A vitrine deve rodar com qualquer endpoint OpenAI-compatível (OpenRouter, Ollama local, OpenAI).

## Decisão
1. Cliente único `OpenAICompatLLM` (via `langchain_openai.ChatOpenAI` com `base_url`, `api_key`, `model`, `temperature=0`) atrás do protocolo `LLMClient.complete(system, user) -> str`.
2. Configuração por env: `LLM_BASE_URL` (padrão `https://openrouter.ai/api/v1`), `LLM_API_KEY`, `LLM_MODEL` (padrão decidido em D-03).
3. Testes usam `FakeLLM` com respostas roteirizadas; o CI nunca chama provedor real.
4. Timeout 30 s e no máximo 1 retry de rede por chamada; nunca logar prompt completo nem chave.

## Alternativas consideradas
| Alternativa | Prós | Contras | Por que não |
|-------------|------|---------|-------------|
| **Escolhida:** OpenAI-compatível | troca de provedor por env | menos features específicas | — |
| SDK por provedor | recursos nativos | acoplamento | mais código |
| `init_chat_model` do LangChain | multi-provider | mais dependências | desnecessário |

## Consequências
- **Positivas:** custo baixo; roda com Ollama offline.
- **Negativas:** modelos baratos erram mais SQL → retry + evals.

## Conformidade
`tests/test_llm.py` com FakeLLM; revisão rejeita import de SDK de provedor fora de `llm.py`.

## Rastreabilidade
SPEC-LG-04.
