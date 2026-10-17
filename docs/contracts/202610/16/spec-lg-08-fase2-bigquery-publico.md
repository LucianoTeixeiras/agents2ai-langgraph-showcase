# SPEC-LG-08 — Fase 2 GCP: backend BigQuery público (`thelook_ecommerce`)

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-08 |
| **Status** | **Bloqueada** — Fase 2 (precisa do projeto GCP de testes da Agents2AI com alerta de US$ 10) |
| **PRD / System Design / ADRs** | `system-design-agente-nl2sql-langgraph.md` §5, §7, §8 · ADR-002, ADR-005 |
| **Jira** | [A2LG-9](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-9) (épico A2LG-1) |
| **Branch** | `feat/lg-08-bigquery` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-high` |
| **Depende de** | SPEC-LG-06 mergeada; projeto GCP de testes criado por Luciano |
| **Tamanho máximo** | ≤ 250 linhas de diff, ≤ 7 arquivos |

## 1. Objetivo
`showcase ask "<pergunta>" --backend bigquery` usa `BigQueryExecutor`, que só aceita tabelas `bigquery-public-data.thelook_ecommerce.*`, estima custo por dry-run e executa com `maximum_bytes_billed`.

## 2. Fora de escopo
Criar o projeto GCP (Luciano), outros datasets, deploy em GCP (Cloud Run fica para spec futura).

## 3. Contexto mínimo
API **verificada em 2026-10-16** com `google-cloud-bigquery==3.46.2` (assinaturas da biblioteca instalada; use `google-cloud-bigquery>=3.46,<4`):
```python
from google.cloud import bigquery
client = bigquery.Client(project=settings.gcp_project)
# Client.query(query: str, job_config: QueryJobConfig | None = None, ..., location: str | None = None, ...) -> QueryJob

def estimate_bytes(self, sql: str) -> int:
    cfg = bigquery.QueryJobConfig(dry_run=True, use_query_cache=False)
    job = self._client.query(sql, job_config=cfg)   # dry-run: não fatura, retorna na hora
    return int(job.total_bytes_processed or 0)      # QueryJob.total_bytes_processed: int | None

def run(self, sql: str, max_rows: int) -> list[dict]:
    cfg = bigquery.QueryJobConfig(maximum_bytes_billed=self._settings.max_bytes_billed)  # int
    rows = self._client.query(sql, job_config=cfg).result(max_results=max_rows)
    return [dict(r.items()) for r in rows]
```
Se o job passar de `maximum_bytes_billed`, o BigQuery falha o job (sem cobrança) e `.result()` levanta `google.api_core.exceptions.BadRequest`: capture e devolva a mensagem fixa `"Consulta excede o limite de bytes faturáveis."`. Testes usam client fake que registra `job_config` (sem rede). Colunas reais do catálogo: System Design §7.

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `pyproject.toml` / `uv.lock` | alterar | grupo opcional `bigquery = ["google-cloud-bigquery>=3.25"]` |
| `src/showcase/settings.py` | alterar | + `gcp_project: str = ""`, `max_bytes_billed: int = 10_000_000_000`, `backend: Literal["duckdb","bigquery"]` |
| `src/showcase/executors/bigquery_executor.py` | criar | `BigQueryExecutor(settings, client=None)`; `catalog()` fixo com as 4 tabelas qualificadas e as colunas reais do System Design §7; import de `google.cloud.bigquery` **dentro** do construtor |
| `src/showcase/cli.py` | alterar | `--backend duckdb|bigquery` |
| `.env.example` | alterar | `GCP_PROJECT=`, `MAX_BYTES_BILLED=10000000000` |
| `tests/test_bigquery_executor.py` | criar | client fake, sem rede |

## 6. Testes
`test_bq_dry_run_usa_bytes`, `test_bq_run_define_maximum_bytes_billed`, `test_bq_catalogo_so_tabelas_publicas`, `test_cli_backend_bigquery_sem_projeto_falha_com_mensagem`.

## 8. Ações proibidas
Mesmas da SPEC-LG-05, e **nunca** colocar `project_id` real em código/teste (use `"test-project"`); nunca projeto de cliente.

## 10. PR
Título: `feat(bigquery): backend BigQuery público na fase 2 [A2LG-9]`
