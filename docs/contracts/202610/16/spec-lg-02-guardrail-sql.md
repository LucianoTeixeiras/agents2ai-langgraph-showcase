# SPEC-LG-02 — Implementar o guardrail de SQL com sqlglot

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-02 |
| **Status** | Pronta |
| **PRD / System Design / ADRs** | `prd-agente-nl2sql-langgraph.md` · `system-design-agente-nl2sql-langgraph.md` §6 · ADR-005 |
| **Jira** | pendente — D-01 (projeto novo `A2LG` a ser criado por Luciano) |
| **Branch** | `feat/lg-02-guardrail-sql` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-medium` |
| **Depende de** | SPEC-LG-01 mergeada |
| **Tamanho máximo** | ≤ 300 linhas de diff, ≤ 5 arquivos (acima disso: pare e escale) |
| **Commit de referência** | `main` após merge do PR de planejamento |

## 1. Objetivo
`validate_sql` (System Design §6) reprova todo SQL não-somente-leitura, multi-statement, fora da allowlist ou com PII, e garante `LIMIT ≤ max_limit`; o nó `guardrail` passa a usá-lo.

## 2. Fora de escopo
Executar SQL; mudar outros nós; mudar a allowlist de dados (vem da SPEC-03 via `executor.catalog()`).

## 3. Contexto mínimo
API verificada (sqlglot 30.23.0): `sqlglot.parse(sql, read=dialect)` → lista; classes `exp.Insert, Update, Delete, Drop, Create, Alter, Merge, Command, Select, Union, Table, Column, Star`; `tree.args.get("limit")`; `select.limit(n)` retorna cópia com LIMIT. Nome qualificado de tabela: `".".join(p.name for p in table.parts)` (minúsculas para comparar). Erros de parse: `sqlglot.errors.ParseError`.

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `pyproject.toml` / `uv.lock` | alterar | + `sqlglot==30.23.0` |
| `src/showcase/guardrails.py` | criar | `@dataclass(frozen=True) GuardrailResult(ok: bool, sql: str, errors: list[str])`; `validate_sql(sql, dialect, allowed_tables: set[str], pii_columns: set[str], max_limit: int = 1000) -> GuardrailResult`; constante `PII_COLUMNS = {"email", "first_name", "last_name", "street_address", "postal_code", "latitude", "longitude"}` |
| `src/showcase/nodes/core.py` | alterar | `guardrail` chama `validate_sql` (allowlist = chaves de `deps.executor.catalog()`, dialect = `state.get("backend", "duckdb")`), retorna `{"sql": result.sql, "guardrail_errors": result.errors}` |
| `tests/test_guardrails.py` | criar | casos da §6 |

Mensagens de erro exatas (usar literalmente): `"parse_error"`, `"multiple_statements"`, `"not_select"`, `"forbidden_statement"`, `"table_not_allowed:<nome>"`, `"pii_column:<nome>"`, `"star_on_pii_table:<nome>"`. Tabela com PII = tabela cujo catálogo contém alguma coluna de `pii_columns`; para isso a assinatura recebe também `catalog: dict[str, list[str]] | None = None` (opcional; sem catálogo, `SELECT *` é permitido).

## 5. Passo a passo
Dependência → módulo → nó → testes → portões → PR.

## 6. Testes (mín. 14, parametrizados)
Aprovam: select simples (LIMIT injetado 1000); LIMIT 5000 → 1000; LIMIT 10 mantido; CTE; UNION; join entre 2 tabelas permitidas; agregação com GROUP BY.
Reprovam: `DROP TABLE orders`; `DELETE FROM orders`; `INSERT ...`; `UPDATE ...`; `select 1; drop table orders` (multiple_statements); `SELECT email FROM users`; `SELECT * FROM users` com catálogo contendo `email`; tabela `secrets`; texto não-SQL (`parse_error`); `ATTACH 'x.db'` / `COPY` (forbidden_statement ou parse_error); CTE contendo DELETE.

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

Título: `feat(guardrail): validação somente-leitura de SQL com sqlglot [pendente]`
