# System Design — Agente NL→SQL em LangGraph

| Campo | Valor |
|-------|-------|
| **Status** | Aprovado (2026-10-16) |
| **PRD** | `prd-agente-nl2sql-langgraph.md` |
| **ADRs** | ADR-001 (LangGraph), ADR-002 (dados), ADR-003 (LLM), ADR-004 (observabilidade), ADR-005 (guardrails), ADR-006 (schema por similaridade) |
| **Versões verificadas (2026-10-16, PyPI + teste local)** | `langgraph==1.2.15`, `langgraph-checkpoint-sqlite==3.1.2`, `sqlglot==30.23.0`, `duckdb==1.5.6`, `langchain-openai==1.7.2` |

## 1. Grafo

```mermaid
flowchart TD
    START([START]) --> intent[classify_intent]
    intent -->|out_of_scope| answer[answer]
    intent -->|sql| schema[retrieve_schema]
    schema --> gen[generate_sql]
    gen --> guard[guardrail<br/>sqlglot: só leitura, 1 statement,<br/>LIMIT, tabelas permitidas, PII]
    guard -->|reprovado e attempts < 3| gen
    guard -->|reprovado e attempts >= 3| answer
    guard -->|aprovado| cost[estimate_cost<br/>DuckDB=0 · BigQuery dry-run (fase 2)]
    cost -->|bytes <= limite| exec[execute_sql<br/>DuckDB local · BigQuery public (fase 2)]
    cost -->|bytes > limite| hitl[human_approval<br/>interrupt]
    hitl -->|aprovado| exec
    hitl -->|negado| answer
    exec --> answer
    answer --> END([END])
```

## 2. Estado (`src/showcase/state.py`)

```python
import operator
from typing import Annotated, Literal, TypedDict

class AgentState(TypedDict, total=False):
    question: str
    backend: Literal["duckdb", "bigquery"]
    intent: Literal["sql", "out_of_scope"]
    schema_context: str
    sql: str
    guardrail_errors: list[str]
    attempts: int
    estimated_bytes: int
    approved: bool
    rows: list[dict]
    answer: str
    trace: Annotated[list[dict], operator.add]   # reducer: acumula spans
```

Regras: nós retornam **dict parcial**; só `trace` usa reducer. Nenhum nó guarda objetos não serializáveis no estado (conexões ficam em `deps`, ver §4), senão o checkpointer quebra.

## 3. Nós (`src/showcase/nodes/`)

| Nó | Entrada | Saída | Efeitos externos |
|----|---------|-------|------------------|
| `classify_intent` | question | intent | LLM |
| `retrieve_schema` | backend | schema_context | leitura do catálogo permitido; top-k por similaridade (ADR-006, SPEC-LG-09) |
| `generate_sql` | question, schema_context, guardrail_errors | sql, attempts+1 | LLM |
| `guardrail` | sql, backend | sql (com LIMIT), guardrail_errors | nenhum (puro) |
| `estimate_cost` | sql, backend | estimated_bytes | BigQuery dry-run |
| `human_approval` | sql, estimated_bytes | approved | `interrupt({"sql", "estimated_bytes"})` |
| `execute_sql` | sql, backend | rows (≤ 50) | DuckDB / BigQuery |
| `answer` | tudo | answer | LLM (exceto recusas, que são texto fixo) |

## 4. Injeção de dependências
`build_graph(deps: Deps, checkpointer=None)` em `src/showcase/graph.py`. `Deps` (dataclass) contém `llm` (protocolo `LLMClient.complete(system: str, user: str) -> str`), `executor` (protocolo `SqlExecutor`), `settings`. Nós são fechados sobre `deps` via `functools.partial`. Testes usam `FakeLLM` e `DuckDBExecutor` em memória — **CI nunca chama rede**.

## 5. Checkpointer e HITL
- Testes: `langgraph.checkpoint.memory.InMemorySaver`.
- CLI: `langgraph.checkpoint.sqlite.SqliteSaver.from_conn_string("checkpoints/showcase.sqlite")` (context manager; pasta ignorada no Git).
- Pausa: `human_approval` chama `interrupt(payload)`; o `invoke` retorna estado com chave `__interrupt__` (lista de `Interrupt(value=...)`).
- Retomada: `graph.invoke(Command(resume=True|False), {"configurable": {"thread_id": tid}})`.
- **Verificado localmente** em langgraph 1.2.15 (script de prova no PR). Atenção: o nó com `interrupt` é **re-executado do início** na retomada — não coloque efeitos colaterais antes do `interrupt()`.

## 6. Guardrail (`src/showcase/guardrails.py`)
`validate_sql(sql: str, dialect: str, allowed_tables: set[str], pii_columns: set[str], max_limit: int = 1000) -> GuardrailResult(ok: bool, sql: str, errors: list[str])`. Regras: parse com `sqlglot.parse(sql, read=dialect)`; exatamente 1 statement; raiz `exp.Select` (ou `exp.Union`/CTE terminando em select); nenhum nó `Insert/Update/Delete/Drop/Create/Alter/Merge/Command`; todas as `exp.Table` em `allowed_tables`; nenhuma `exp.Column` em `pii_columns` e nenhum `SELECT *` em tabela com PII; `LIMIT` ausente → injetado `max_limit`; `LIMIT` > max → reduzido.

## 7. Dados
- Schema espelha `bigquery-public-data.thelook_ecommerce` (tabelas `orders`, `order_items`, `products`, `users`) — dataset público do Google (**existência e colunas: verificar no console antes da SPEC-05**).
- DuckDB: gerador determinístico (`scripts/generate_sample_db.py`, seed fixa) cria dados **sintéticos** com esse schema; PII sintética em `users.email`, `users.first_name`, `users.last_name`, `users.street_address` (para exercitar o bloqueio).
- BigQuery: allowlist de prefixo `bigquery-public-data.thelook_ecommerce.`; job com `dry_run=True` para estimar e `maximum_bytes_billed` para executar.

## 8. Configuração (`src/showcase/settings.py`, env)
`LLM_BASE_URL` (padrão `https://openrouter.ai/api/v1`), `LLM_API_KEY` (obrigatória só fora dos testes), `LLM_MODEL` (`nvidia/nemotron-3.5-lightning`, D-03), `BACKEND` (`duckdb`), `DUCKDB_PATH` (`data/sample.duckdb`), `MAX_BYTES_WITHOUT_APPROVAL` (`1_000_000_000`), `TRACE_LOG` (`true`), `LANGSMITH_TRACING` (opcional). **Fase 2 GCP (SPEC-LG-08):** `GCP_PROJECT` (projeto de testes **da Agents2AI**, nunca de cliente) e `MAX_BYTES_BILLED` (`10_000_000_000`).

## 8a. Execução local em Docker (D-04 — Fase 1)
- `Dockerfile`: `python:3.12-slim` + binário `uv` copiado de `ghcr.io/astral-sh/uv`; `uv sync --locked --no-dev`; usuário não-root `app`; sem segredos na imagem (`.dockerignore` exclui `.env`, `data/`, `checkpoints/`, `.git`).
- `compose.yaml`: serviço `showcase` com `env_file: .env` (opcional), volumes `./data:/app/data` e `./checkpoints:/app/checkpoints`; `docker compose run --rm showcase ask "..."` gera o DuckDB se faltar e roda a CLI.
- Nada de GCP na fase 1; BigQuery entra só na Fase 2 (SPEC-LG-08).

## 9. Observabilidade e evals
- Decorador `traced(node_name)` mede duração e anexa `{"node", "ms", "tokens_in", "tokens_out"}` a `trace`; ao final, uma linha JSON INFO por execução (sem SQL de usuário com PII, sem chaves).
- LangSmith: ativado só por env (`LANGSMITH_TRACING=true` + chave); OTel: fase 2 (ADR-004).
- Evals: `evals/golden.jsonl` (pergunta, SQL de referência); métrica = igualdade dos resultados executados (multiconjunto de linhas) + p50/p95 de latência; relatório em `evals/report.md`.

## 10. Ordem de execução das specs
| # | Spec | Depende de | Status |
|---|------|-----------|--------|
| 1 | SPEC-LG-01 fundação (deps, settings, estado, grafo com nós stub) | — | Pronta |
| 2 | SPEC-LG-02 guardrail sqlglot | 01 | Pronta |
| 3 | SPEC-LG-03 dataset DuckDB + catálogo + executor | 01 | Pronta |
| 4 | SPEC-LG-04 nós LLM (intent, SQL, answer) + retry | 02, 03 | Pronta após 02/03 |
| 5 | SPEC-LG-05 HITL + SqliteSaver + CLI (só DuckDB) | 04 | Pronta após 04 |
| 6 | SPEC-LG-07 Dockerfile + compose | 05 | Pronta após 05 |
| 7 | SPEC-LG-06 tracing + evals + README vitrine | 07 | Pronta após 07 |
| 8 | SPEC-LG-09 schema por similaridade (DuckDB vss, ADR-006) | 06 | Pronta após 06 |
| 9 | SPEC-LG-08 Fase 2 GCP: BigQuery público | 06 + projeto GCP de testes | **Bloqueada** (fase 2) |
