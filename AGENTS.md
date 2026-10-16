# AGENTS.md — agents2ai-langgraph-showcase

> Porta de entrada para agentes de IA e devs. Governança canônica: Hub `agent-kit` (core-rules, ADR-067 arquiteto→executor). Em conflito, o Hub prevalece.

## 1. Git
- **Proibido commitar direto em `main`** (protegida, exige CI verde). Trabalhe em `feat/`, `fix/`, `docs/`, `chore/`.
- Antes de editar: `git branch --show-current`; se for `main`, crie a branch.
- Commits/PRs em pt-BR: `<tipo>(<escopo>): <descrição> [<CHAVE-NN>]` (use `[pendente]` enquanto não houver projeto Jira).
- Nunca `git add -A`/`git add .`; nunca faça merge do próprio PR; nunca force-push em branch compartilhada.

## 2. Modelo arquiteto → executor (ADR-067 do Hub)
- **Arquiteto/planner:** Dev Grok Bot — PRD, System Design, ADRs, specs, revisão de PR.
- **Executor:** Cursor + Composer (modelo econômico) — executa **uma spec por vez** de `docs/contracts/`, seguindo `guia-executor-cursor-composer.md`.
- **Aceite e merge:** Luciano Teixeira.
- Spec ambígua → **pare e pergunte**; não improvise.

## 3. Invariantes de segurança
1. Somente **dados públicos**: DuckDB local com dados públicos e BigQuery `bigquery-public-data.*`. Nenhum dado, nome ou projeto GCP de cliente.
2. SQL gerado é **somente leitura** (validado por sqlglot), com `LIMIT` obrigatório e bloqueio de colunas PII.
3. Consultas acima do limite de custo exigem **aprovação humana** (interrupt do LangGraph).
4. Segredos só em `.env` (fora do Git). Nunca imprimir, logar ou commitar valores: use `[REDACTED]`. `gitleaks` roda no CI.

## 4. Portões (iguais ao CI)
```bash
uv sync --group dev
uv run ruff check .
uv run pytest -q
```

## 5. Documentação (YYYYMM/DD)
`docs/<categoria>/YYYYMM/DD/<arquivo-kebab-case>.md` — categorias `analysis/`, `contracts/`, `decisions/`, `reports/`. Perenes: `docs/governance/`, `docs/risks/`, `docs/runbooks/`. Diretórios em lowercase. Decisões resumidas em `docs/DECISIONS-LOG.md`.

## 6. DoD
- [ ] Portões verdes localmente e no CI.
- [ ] Diff restrito aos arquivos da spec.
- [ ] Mermaid/System Design atualizado se o fluxo mudar.
- [ ] Entrada no `docs/DECISIONS-LOG.md` quando houver decisão.
