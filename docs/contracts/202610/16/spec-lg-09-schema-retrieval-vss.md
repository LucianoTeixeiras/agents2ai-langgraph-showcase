# SPEC-LG-09 — Recuperação de schema por similaridade (DuckDB `vss`/HNSW)

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-09 |
| **Status** | Pronta após SPEC-LG-06 |
| **PRD / System Design / ADRs** | System Design §3 (`retrieve_schema`) · ADR-006, ADR-002 |
| **Jira** | [A2LG-10](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-10) (épico A2LG-1) |
| **Branch** | `feat/lg-09-schema-vss` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-medium` |
| **Depende de** | SPEC-LG-06 mergeada (evals medem o ganho) |
| **Tamanho máximo** | ≤ 250 linhas de diff, ≤ 7 arquivos |

## 1. Objetivo
`retrieve_schema` devolve só as `k` (padrão 4) colunas/tabelas mais similares à pergunta, via índice HNSW em memória reconstruído no startup. Evals não pioram.

## 2. Fora de escopo
Persistência do índice, pgvector/vector DB, embeddings pagos.

## 3. Contexto mínimo (verificado nas docs DuckDB, 2026-10-16)
```sql
INSTALL vss; LOAD vss;
CREATE TABLE schema_docs(id INTEGER, table_name VARCHAR, text VARCHAR, vec FLOAT[256]);
CREATE INDEX schema_hnsw ON schema_docs USING HNSW (vec) WITH (metric = 'cosine');
SELECT table_name, text FROM schema_docs ORDER BY array_cosine_distance(vec, ?::FLOAT[256]) LIMIT 4;
```
HNSW só em banco **em memória** (`duckdb.connect(":memory:")`). **Proibido** `hnsw_enable_experimental_persistence`.

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `src/showcase/retrieval/embedder.py` | criar | `Protocol Embedder.embed(text) -> list[float]`; `HashingEmbedder(dim=256)` determinístico (tokens → hash → L2-normalizado), sem rede |
| `src/showcase/retrieval/schema_index.py` | criar | `SchemaIndex.build(catalog, embedder)` em `:memory:` com `vss`; fallback sem índice se `LOAD vss` falhar (log WARNING); `search(question, k) -> list[str]` |
| `src/showcase/nodes/core.py` | alterar | `retrieve_schema` usa `deps.schema_index` se presente; senão catálogo inteiro |
| `src/showcase/deps.py` | alterar | `schema_index: SchemaIndex | None = None` |
| `Dockerfile` | alterar | `RUN uv run --no-sync python -c "import duckdb; duckdb.sql('INSTALL vss')"` no build |
| `tests/test_schema_retrieval.py` | criar | §6 |

## 6. Testes
- `test_busca_retorna_tabela_relevante` ("preço dos produtos" → `products` no top-k).
- `test_indice_hnsw_usado` (`EXPLAIN` contém `HNSW_INDEX_SCAN`; `skip` permitido **só** se `vss` não puder ser instalado offline no CI, com motivo explícito).
- `test_fallback_sem_vss` (simula falha no `LOAD`; busca exata devolve o mesmo top-1).
- `test_sem_persistencia_experimental` (grep no código).

## 7. DoD
Só os arquivos da §4; ruff + pytest + CI verdes; evals (SPEC-LG-06) sem regressão, relatório anexado ao PR.

## 8. Ações proibidas
As mesmas da SPEC-LG-05; mais: ligar persistência experimental, adicionar dependências Python, chamar API de embeddings.

## 9. Escalonamento
**Se a spec for ambígua, pare e pergunte; não improvise.**

## 10. PR
Título: `feat(retrieval): schema por similaridade com DuckDB vss [A2LG-10]`
