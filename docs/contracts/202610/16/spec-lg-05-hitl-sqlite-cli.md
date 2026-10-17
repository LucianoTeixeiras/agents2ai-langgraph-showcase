# SPEC-LG-05 — Adicionar aprovação humana, SqliteSaver e CLI (somente DuckDB)

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-05 |
| **Status** | Pronta após SPEC-LG-04 |
| **PRD / System Design / ADRs** | `prd-agente-nl2sql-langgraph.md` · `system-design-agente-nl2sql-langgraph.md` §5, §8 · ADR-001, ADR-005 |
| **Jira** | [A2LG-6](https://lucianoteixeiras-1773934569563.atlassian.net/browse/A2LG-6) (épico A2LG-1) |
| **Branch** | `feat/lg-05-hitl-cli` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-medium` |
| **Depende de** | SPEC-LG-04 mergeada |
| **Tamanho máximo** | ≤ 250 linhas de diff, ≤ 6 arquivos (acima disso: pare e escale) |
| **Commit de referência** | `main` após merge da SPEC-LG-04 |

## 1. Objetivo
`uv run showcase ask "<pergunta>" [--thread <id>]` roda o grafo com `SqliteSaver` sobre o DuckDB local; se houver interrupt, mostra o SQL e os bytes estimados e pergunta `Aprovar? [s/N]`, retomando com `Command(resume=...)`.

## 2. Fora de escopo
BigQuery/GCP (SPEC-LG-08, fase 2), Docker (SPEC-LG-07), UI/API HTTP, autenticação.

## 3. Contexto mínimo
Verificado (langgraph-checkpoint-sqlite 3.1.2): `from langgraph.checkpoint.sqlite import SqliteSaver`; `with SqliteSaver.from_conn_string(path) as cp: graph = build_graph(deps, checkpointer=cp)`. Para testar o HITL com DuckDB (custo 0), use `max_bytes_without_approval=-1` nos testes.

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `pyproject.toml` / `uv.lock` | alterar | + `langgraph-checkpoint-sqlite==3.1.2`; `[project.scripts] showcase = "showcase.cli:main"` |
| `src/showcase/cli.py` | criar | `main(argv=None) -> int` com argparse (subcomando `ask`, `--thread`); `thread_id` padrão `uuid4().hex`; checkpoint em `checkpoints/showcase.sqlite` (cria a pasta); backend fixo `duckdb` |
| `tests/test_hitl.py` | criar | §6 |
| `tests/test_cli.py` | criar | §6 com `monkeypatch` de `input` e FakeLLM |

## 5. Passo a passo
Dependências → CLI → testes → portões → PR.

## 6. Testes
- `test_hitl_sqlite_retoma_em_novo_grafo`: 1º grafo pausa; 2º grafo com o mesmo arquivo SQLite (`tmp_path`) e mesmo `thread_id` retoma e executa.
- `test_hitl_nega_nao_executa`.
- `test_cli_aprova` / `test_cli_nega` (input `s` / `n`), exit code 0.

## 7. Definição de Pronto (DoD)
- [ ] Só os arquivos da §4 mudaram (`git diff --stat main`).
- [ ] `uv sync --locked --group dev`, `uv run ruff check .`, `uv run pytest -q` verdes localmente e no CI (incl. gitleaks).
- [ ] Nenhum teste com `skip`/`xfail` novo; nenhuma chamada de rede nos testes.
- [ ] PR aberto (não mergeado) com o corpo da §10.

## 8. Ações proibidas
- Commitar em `main`, fazer merge, force-push, `git add -A`/`git add .`.
- Adicionar/atualizar dependências além das listadas na §4; usar `pip install`; importar `google.cloud`.
- Tocar arquivos fora da §4; reformatar/renomear arquivos vizinhos.
- Incluir chaves, `.env`, dados/nomes/projetos GCP de cliente; chamar LLM real nos testes.
- Afrouxar o guardrail para um teste passar.

## 9. Regra de escalonamento
**Se a spec for ambígua, pare e pergunte; não improvise.** Pare se: um símbolo/API não existir como descrito; um teste só passar mudando algo fora da §4; o diff passar do limite; surgir decisão de produto, segurança, custo ou dependência. Abra o PR em rascunho e comente citando a seção da spec.

## 10. PR
Corpo: Spec · O que foi feito · Evidência dos portões (saída real) · Desvios · Checklist DoD · Risco QA.

Título: `feat(hitl): aprovação humana com SqliteSaver e CLI [A2LG-6]`
