# SPEC-LG-04 — Implementar os nós com LLM (intenção, SQL, resposta) e o retry com feedback

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-04 |
| **Status** | Pronta após SPEC-02 e SPEC-03 |
| **PRD / System Design / ADRs** | `prd-agente-nl2sql-langgraph.md` · `system-design-agente-nl2sql-langgraph.md` §3, §4 · ADR-003 |
| **Jira** | pendente — D-01 (projeto novo `A2LG` a ser criado por Luciano) |
| **Branch** | `feat/lg-04-nos-llm` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-medium` |
| **Depende de** | SPEC-LG-02 e SPEC-LG-03 mergeadas |
| **Tamanho máximo** | ≤ 300 linhas de diff, ≤ 7 arquivos (acima disso: pare e escale) |
| **Commit de referência** | `main` após merge do PR de planejamento |

## 1. Objetivo
`classify_intent`, `generate_sql` e `answer` usam `deps.llm` com prompts fixos; `generate_sql` inclui `guardrail_errors` anteriores como feedback; `OpenAICompatLLM` implementa `LLMClient` lendo `Settings`.

## 2. Fora de escopo
Custo/HITL reais, BigQuery, CLI, tracing, streaming, tool calling.

## 3. Contexto mínimo
Nós em `src/showcase/nodes/core.py`; protocolo `LLMClient.complete(system, user) -> str`; `FakeLLM` em `tests/fakes.py`. `langchain_openai.ChatOpenAI(base_url=..., api_key=..., model=..., temperature=0, timeout=30, max_retries=1)`; `.invoke([("system", s), ("human", u)]).content` (**verificar no Cursor que o atributo `.content` é str na versão instalada; se não, pare**).

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `pyproject.toml` / `uv.lock` | alterar | + `langchain-openai==1.7.2` |
| `src/showcase/llm.py` | criar | `class OpenAICompatLLM` (construtor recebe `Settings`; levanta `ValueError("LLM_API_KEY ausente")` se vazia) |
| `src/showcase/prompts.py` | criar | constantes `INTENT_SYSTEM`, `SQL_SYSTEM`, `ANSWER_SYSTEM` (pt-BR) |
| `src/showcase/nodes/core.py` | alterar | 3 nós usam LLM (abaixo) |
| `tests/fakes.py` | alterar | `FakeLLM` grava `calls: list[tuple[str, str]]` |
| `tests/test_llm_nodes.py` | criar | §6 |

Regras dos nós: `classify_intent` → resposta normalizada `strip().lower()`; `"sql"` se começar com `sql`, senão `"out_of_scope"` (pergunta vazia continua sem chamar LLM). `generate_sql`: user = `f"Dialeto: {backend}\nSchema:\n{schema_context}\nPergunta: {question}"` + se houver erros `f"\nErros da tentativa anterior: {', '.join(errors)}"`; remove cercas ```sql/``` da resposta. `answer`: recusas continuam texto fixo (sem LLM); caso normal user = pergunta + SQL + até 20 linhas em JSON; resposta do LLM + `"\n\nSQL:\n" + sql`. `SQL_SYSTEM` deve dizer: só SELECT, só tabelas do schema, nunca colunas de PII, sempre LIMIT.

## 5. Passo a passo
Dependência → prompts → llm.py → nós → fakes → testes → portões → PR.

## 6. Testes (todos com FakeLLM + DuckDB da fixture)
- `test_intent_out_of_scope_por_llm` (LLM responde "fora").
- `test_sql_com_cercas_e_limpo`.
- `test_retry_envia_erros_ao_llm`: 1ª resposta `DROP TABLE orders`, 2ª válida → `calls[2][1]` contém `"forbidden_statement"` ou `"not_select"`; resposta final contém `"SQL:"`.
- `test_tres_falhas_recusa`: 3 SQL inválidos → recusa fixa; executor não chamado.
- `test_llm_sem_chave_levanta`: `OpenAICompatLLM(Settings(llm_api_key=""))` → `ValueError`.
## 7. Definição de Pronto (DoD)
- [ ] Só os arquivos da §4 mudaram (`git diff --stat main`).
- [ ] `uv sync --locked --group dev`, `uv run ruff check .`, `uv run pytest -q` verdes localmente e no CI (incl. gitleaks).
- [ ] Nenhum teste com `skip`/`xfail` novo; nenhuma chamada de rede nos testes.
- [ ] PR aberto (não mergeado) com o corpo da §10.

## 8. Ações proibidas
- Commitar em `main`, fazer merge, force-push, `git add -A`/`git add .`.
- Adicionar/atualizar dependências além das listadas na §4; usar `pip install`.
- Tocar arquivos fora da §4; reformatar/renomear arquivos vizinhos.
- Incluir chaves, `.env`, dados/nomes/projetos GCP de cliente; chamar LLM ou BigQuery reais nos testes.
- Afrouxar o guardrail ou a allowlist para um teste passar.

## 9. Regra de escalonamento
**Se a spec for ambígua, pare e pergunte; não improvise.** Pare se: um símbolo/API não existir como descrito; um teste só passar mudando algo fora da §4; o diff passar do limite; surgir decisão de produto, segurança, custo ou dependência. Abra o PR em rascunho e comente citando a seção da spec.

## 10. PR
Título: ver abaixo. Corpo: Spec · O que foi feito · Evidência dos portões (saída real) · Desvios · Checklist DoD · Risco QA.

Título: `feat(llm): nós de intenção, SQL e resposta com retry guiado pelo guardrail [pendente]`
