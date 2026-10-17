# SPEC-LG-03 — Criar o dataset DuckDB sintético, o catálogo e o executor local

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-03 |
| **Status** | Pronta |
| **PRD / System Design / ADRs** | `prd-agente-nl2sql-langgraph.md` · `system-design-agente-nl2sql-langgraph.md` §7 · ADR-002, ADR-005 |
| **Jira** | [A2LG-4](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-4) (épico A2LG-1) |
| **Branch** | `feat/lg-03-duckdb` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-low` |
| **Depende de** | SPEC-LG-01 mergeada |
| **Tamanho máximo** | ≤ 300 linhas de diff, ≤ 6 arquivos (acima disso: pare e escale) |
| **Commit de referência** | `main` após merge do PR de planejamento |

## 1. Objetivo
`uv run python scripts/generate_sample_db.py` cria `data/sample.duckdb` determinístico (seed 42) com o schema do System Design §7; `DuckDBExecutor` implementa `SqlExecutor` em conexão somente leitura.

## 2. Fora de escopo
BigQuery; LLM; versionar o `.duckdb`; baixar dados da internet.

## 3. Contexto mínimo
Protocolo em `src/showcase/deps.py` (`catalog`, `estimate_bytes`, `run`). `.gitignore` já ignora `*.duckdb`. DuckDB: `duckdb.connect(path, read_only=True)`; `con.execute(sql).fetchall()` e `con.description`.

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `pyproject.toml` / `uv.lock` | alterar | + `duckdb==1.5.6` |
| `scripts/generate_sample_db.py` | criar | `generate(path: str, seed: int = 42) -> None`; CLI `--path` (padrão `data/sample.duckdb`); usa só `random` da stdlib |
| `src/showcase/executors/__init__.py` | criar | vazio |
| `src/showcase/executors/duckdb_executor.py` | criar | `class DuckDBExecutor(path)`: `catalog()` via `information_schema.columns` (schema `main`) → `{"orders": [...], ...}`; `estimate_bytes` → `0`; `run(sql, max_rows)` → `list[dict]` com no máx. `max_rows` (`fetchmany`) |
| `tests/conftest.py` | criar | fixture `sample_db(tmp_path)` que chama `generate` |
| `tests/test_duckdb_executor.py` | criar | §6 |

Schema (tipos DuckDB): `users(id INTEGER, first_name VARCHAR, last_name VARCHAR, email VARCHAR, age INTEGER, country VARCHAR, street_address VARCHAR, created_at TIMESTAMP)` 200 linhas; `products(id INTEGER, name VARCHAR, category VARCHAR, brand VARCHAR, retail_price DOUBLE, cost DOUBLE)` 50; `orders(order_id INTEGER, user_id INTEGER, status VARCHAR, created_at TIMESTAMP, num_of_item INTEGER)` 500; `order_items(id INTEGER, order_id INTEGER, user_id INTEGER, product_id INTEGER, status VARCHAR, sale_price DOUBLE, created_at TIMESTAMP)` 1200. Status ∈ {Complete, Shipped, Processing, Cancelled, Returned}; datas em 2023–2024. E-mails no domínio `example.com`.

## 5. Passo a passo
Dependência → gerador → executor → fixture → testes → portões → PR.

## 6. Testes
- `test_gerador_deterministico`: dois `generate` com seed 42 → mesma soma de `sale_price` (arredondada a 2 casas).
- `test_catalogo_tem_4_tabelas` e colunas exatas de `users`.
- `test_run_respeita_max_rows` (`max_rows=5` → 5 linhas, dicts com nomes de coluna).
- `test_conexao_read_only`: `run("DELETE FROM orders", 1)` levanta `duckdb.Error`.
- `test_estimate_bytes_zero`.

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

Título: `feat(dados): dataset DuckDB sintético e executor somente leitura [A2LG-4]`
