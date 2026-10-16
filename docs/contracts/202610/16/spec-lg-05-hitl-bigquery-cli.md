# SPEC-LG-05 — Adicionar custo, aprovação humana, SqliteSaver, BigQuery público e CLI

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-05 |
| **Status** | Bloqueada (D-02, D-04) |
| **PRD / System Design / ADRs** | `prd-agente-nl2sql-langgraph.md` · `system-design-agente-nl2sql-langgraph.md` §5, §7, §8 · ADR-001, ADR-002, ADR-005 |
| **Jira** | pendente — D-01 |
| **Branch** | `feat/lg-05-hitl-bigquery-cli` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-high` |
| **Depende de** | SPEC-LG-04 mergeada; D-02 e D-04 decididas |
| **Tamanho máximo** | ≤ 300 linhas de diff, ≤ 8 arquivos (acima disso: pare e escale) |
| **Commit de referência** | `main` após merge do PR de planejamento |

## 1. Objetivo
`uv run showcase ask "<pergunta>" [--backend duckdb|bigquery] [--thread <id>]` roda o grafo com `SqliteSaver`; se houver interrupt, mostra SQL e bytes estimados e pergunta `Aprovar? [s/N]`, retomando com `Command(resume=...)`. `BigQueryExecutor` só aceita tabelas `bigquery-public-data.thelook_ecommerce.*`.

## 2. Fora de escopo
UI/API HTTP, autenticação, outros datasets, criar projeto GCP (Luciano cria — D-04).

## 3. Contexto mínimo
Verificado (langgraph-checkpoint-sqlite 3.1.2): `from langgraph.checkpoint.sqlite import SqliteSaver`; `with SqliteSaver.from_conn_string(path) as cp: graph = build_graph(deps, checkpointer=cp)`. BigQuery: `google.cloud.bigquery.Client(project=settings.gcp_project)`; `QueryJobConfig(dry_run=True, use_query_cache=False)` → `job.total_bytes_processed`; execução com `QueryJobConfig(maximum_bytes_billed=settings.max_bytes_billed)` (**API do cliente BigQuery não verificada nesta data — confirme no Cursor; se divergente, pare**).

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `pyproject.toml` / `uv.lock` | alterar | + `langgraph-checkpoint-sqlite==3.1.2`; grupo opcional `bigquery = ["google-cloud-bigquery>=3.25"]`; `[project.scripts] showcase = "showcase.cli:main"` |
| `src/showcase/executors/bigquery_executor.py` | criar | `BigQueryExecutor(settings, client=None)`; `catalog()` fixo com as 4 tabelas qualificadas e colunas do System Design §7; `estimate_bytes` via dry-run; `run` com `maximum_bytes_billed`; import de `google.cloud.bigquery` **dentro** do construtor |
| `src/showcase/cli.py` | criar | `main(argv=None) -> int` com argparse; `thread_id` padrão `uuid4().hex`; checkpoint em `checkpoints/showcase.sqlite` |
| `src/showcase/nodes/core.py` | alterar | só se necessário para `backend` vindo do estado inicial |
| `tests/test_hitl.py` | criar | §6 |
| `tests/test_bigquery_executor.py` | criar | §6 com client fake (sem rede) |
| `tests/test_cli.py` | criar | §6 com `monkeypatch` de `input` |

## 5. Passo a passo
Dependências → executor BQ → CLI → testes → portões → PR. **Nunca** coloque um `project_id` real em código ou teste: use `"test-project"`.

## 6. Testes
- `test_hitl_sqlite_retoma_em_novo_grafo`: 1º grafo pausa; 2º grafo com o mesmo arquivo SQLite (tmp_path) e mesmo `thread_id` retoma e executa.
- `test_hitl_nega_nao_executa`.
- `test_bq_dry_run_usa_bytes` e `test_bq_run_define_maximum_bytes_billed` (client fake registra o `job_config`).
- `test_bq_catalogo_so_tabelas_publicas` (todas começam com `bigquery-public-data.thelook_ecommerce.`).
- `test_cli_aprova` / `test_cli_nega` (input `s` / `n`), exit code 0.
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

Título: `feat(hitl): aprovação humana com SqliteSaver, BigQuery público e CLI [pendente]`
