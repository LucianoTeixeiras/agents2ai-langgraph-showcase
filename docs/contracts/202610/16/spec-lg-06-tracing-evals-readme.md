# SPEC-LG-06 — Adicionar tracing por nó, evals e README vitrine

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-06 |
| **Status** | Pronta após SPEC-LG-07 |
| **PRD / System Design / ADRs** | `prd-agente-nl2sql-langgraph.md` · `system-design-agente-nl2sql-langgraph.md` §9 · ADR-004 |
| **Jira** | [A2LG-7](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-7) (épico A2LG-1) |
| **Branch** | `feat/lg-06-tracing-evals` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-low` |
| **Depende de** | SPEC-LG-07 mergeada |
| **Tamanho máximo** | ≤ 300 linhas de diff, ≤ 8 arquivos (acima disso: pare e escale) |
| **Commit de referência** | `main` após merge do PR de planejamento |

## 1. Objetivo
Cada nó executado anexa um span a `trace`; ao fim, uma linha JSON INFO no logger `showcase`. `uv run python -m showcase.evals [--offline]` roda `evals/golden.jsonl` e grava `evals/report.md` com acurácia e p50/p95 por nó. README ganha diagrama mermaid, exemplo de sessão e seção "o que este projeto demonstra".

## 2. Fora de escopo
OpenTelemetry (fase 2, D-05), LangSmith além de documentar as variáveis, UI.

## 3. Contexto mínimo
`trace: Annotated[list[dict], operator.add]` já existe; nós em `nodes/core.py` ligados em `graph.py` via `partial`. Modo offline: `FakeLLM` roteirizado com os SQL de referência (testa o pipeline, não o modelo). Online: `OpenAICompatLLM` (precisa `.env` local; nunca no CI).

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `src/showcase/tracing.py` | criar | `traced(name)` decorador: mede `time.perf_counter`, retorna `{**result, "trace": [{"node": name, "ms": round(x, 1)}]}`; `log_run(thread_id, state)` emite JSON com `thread_id`, `total_ms`, `nodes`, `approved`, `rows_count` — **sem** sql, pergunta, linhas ou chaves |
| `src/showcase/graph.py` | alterar | envolve cada nó com `traced` |
| `src/showcase/cli.py` | alterar | chama `log_run` |
| `src/showcase/evals.py` | criar | `run_evals(offline: bool) -> dict`; acerto = multiconjunto de linhas igual ao do SQL de referência no DuckDB |
| `evals/golden.jsonl` | criar | 10 casos `{"question", "reference_sql"}` sobre o dataset sintético |
| `tests/test_tracing.py` | criar | §6 |
| `tests/test_evals.py` | criar | §6 |
| `README.md` | alterar | seções novas (pt-BR + EN curto), sem mencionar clientes |

## 5. Passo a passo
Tracing → grafo → CLI → golden → evals → testes → README → portões → PR.

## 6. Testes
- `test_um_span_por_no_executado` (fluxo feliz: 7 spans, nomes na ordem do grafo).
- `test_log_sem_dados_sensiveis` (caplog: não contém a pergunta, o SQL nem `"sk-"`).
- `test_evals_offline_100pct` (FakeLLM com SQL de referência → acurácia 1.0, `report.md` gerado em tmp_path).
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

Título: `feat(obs): tracing por nó, evals e README vitrine [A2LG-7]`
