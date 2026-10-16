# ADR-006 — Recuperação de schema por similaridade com DuckDB `FLOAT[n]` + extensão `vss` (HNSW)

| Campo | Valor |
|-------|-------|
| **Status** | Aceita (2026-10-16, Luciano) |
| **Data** | 2026-10-16 |
| **Decisores** | Luciano Teixeira (aceite) · Dev Grok Bot (proposta) |
| **Jira** | pendente — D-01 (projeto novo `A2LG` a ser criado por Luciano) |
| **Relacionados** | System Design §3 (`retrieve_schema`); ADR-002, ADR-005; SPEC-LG-09 |

## Contexto (2026-10-16)
O nó `retrieve_schema` hoje injeta o catálogo inteiro no prompt. Com mais tabelas/colunas isso gasta tokens e piora o SQL de modelos baratos. Queremos mostrar, na vitrine, recuperação de schema por similaridade sem adicionar serviço externo (Fase 1 é 100% Docker local — D-04).

## Decisão
1. Guardar descrições de tabelas/colunas com embeddings em coluna de array fixo nativo do DuckDB `FLOAT[n]`.
2. Usar a extensão oficial `vss` (`INSTALL vss; LOAD vss;`) com `CREATE INDEX ... USING HNSW (vec) WITH (metric = 'cosine')` e consulta `ORDER BY array_cosine_distance(vec, ?::FLOAT[n]) LIMIT k`.
3. **Índice em memória, reconstruído no startup** a partir da tabela de embeddings; **não** ligar `hnsw_enable_experimental_persistence`.
4. `INSTALL vss` acontece no build da imagem Docker (sem download em runtime); fallback: se `vss` não carregar, busca exata por `array_cosine_distance` sem índice (mesmo resultado, só mais lenta).

## Fatos verificados (docs DuckDB `core_extensions/vss`, 2026-10-16)
- "The vss extension is an **experimental** extension" — índice HNSW sobre o tipo `ARRAY` de tamanho fixo.
- Métricas: `l2sq` (padrão, `array_distance`), `cosine` (`array_cosine_distance`), `ip` (`array_negative_inner_product`).
- Persistência: por padrão o HNSW só pode ser criado em bancos **em memória**; em arquivo exige `SET hnsw_enable_experimental_persistence = true`, cuja recuperação de WAL não está implementada (risco de perda/corrupção do índice) e **não é recomendada em produção**.

## Alternativas consideradas
| Alternativa | Prós | Contras | Por que não |
|-------------|------|---------|-------------|
| **Escolhida:** DuckDB + `vss` | zero serviço extra; roda no Docker | experimental; índice em memória | — |
| Catálogo inteiro no prompt | simples | não escala; mais tokens | mantido só como fallback para catálogos pequenos |
| pgvector (Postgres) | maduro, persistente | mais um container | **caminho para escala de produção** |
| Vector DB dedicado (Pinecone, Qdrant…) | escala, filtros | custo/infra | **caminho para escala de produção** |

## Consequências
- **Positivas:** prompt menor; demonstra RAG de schema sem infra.
- **Negativas:** extensão experimental (versão fixada no `uv.lock` do `duckdb`); índice reconstruído a cada startup (custo desprezível no catálogo da vitrine); produção deve migrar para pgvector ou vector DB dedicado.

## Conformidade
`tests/test_schema_retrieval.py` (SPEC-LG-09); revisão rejeita `hnsw_enable_experimental_persistence`.

## Rastreabilidade
SPEC-LG-09.
