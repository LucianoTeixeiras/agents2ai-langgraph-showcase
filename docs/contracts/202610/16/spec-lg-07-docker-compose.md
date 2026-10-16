# SPEC-LG-07 — Empacotar o protótipo em Docker (Dockerfile + compose)

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-07 |
| **Status** | Pronta após SPEC-LG-05 |
| **PRD / System Design / ADRs** | `system-design-agente-nl2sql-langgraph.md` §8a · ADR-002 (Fase 1), ADR-005 |
| **Jira** | pendente — D-01 (projeto novo `A2LG` a ser criado por Luciano) |
| **Branch** | `feat/lg-07-docker` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-low` |
| **Depende de** | SPEC-LG-05 mergeada |
| **Tamanho máximo** | ≤ 150 linhas de diff, ≤ 6 arquivos (acima disso: pare e escale) |

## 1. Objetivo
`docker compose build` e `docker compose run --rm showcase ask "quantos pedidos em 2024?"` funcionam numa máquina só com Docker, usando DuckDB sintético local (D-04). Sem GCP.

## 2. Fora de escopo
BigQuery/GCP, registry/push de imagem, deploy, UI.

## 3. Contexto mínimo
Imagem do uv: `COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/` (**fixe a tag de versão que estiver no `uv.lock`/CI; se não conseguir verificar, pare**). `scripts/generate_sample_db.py` existe desde a SPEC-LG-03.

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `Dockerfile` | criar | `python:3.12-slim`; `WORKDIR /app`; copiar `pyproject.toml`, `uv.lock`, `src/`, `scripts/`; `uv sync --locked --no-dev`; `useradd -m app` + `USER app`; `ENTRYPOINT ["sh","/app/docker/entrypoint.sh"]` |
| `docker/entrypoint.sh` | criar | `set -eu`; se `data/sample.duckdb` não existir, roda o gerador; `exec uv run --no-sync showcase "$@"` |
| `.dockerignore` | criar | `.env`, `.git`, `data/`, `checkpoints/`, `.venv/`, `__pycache__/` |
| `compose.yaml` | criar | serviço `showcase` (`build: .`, `env_file: [{path: .env, required: false}]`, volumes `./data:/app/data`, `./checkpoints:/app/checkpoints`, `stdin_open: true`, `tty: true`) |
| `.github/workflows/ci.yml` | alterar | + job `docker-build` (`docker build -t showcase:ci .`, sem push) |
| `tests/test_docker_files.py` | criar | `.dockerignore` contém `.env`; `Dockerfile` contém `USER app`; nenhum arquivo contém `GCP`/`gcloud` |

## 5. Passo a passo
Arquivos → `docker compose build` local → rodar uma pergunta (com `LLM_API_KEY` no `.env` local, nunca commitado) → portões → PR.

## 6. Testes
`tests/test_docker_files.py` (§4) + job `docker-build` verde.

## 7. Definição de Pronto (DoD)
- [ ] Só os arquivos da §4 mudaram.
- [ ] `uv run ruff check .`, `uv run pytest -q`, `docker compose build` verdes; CI verde (incl. gitleaks e docker-build).
- [ ] Imagem roda como não-root e não contém `.env`.
- [ ] PR aberto (não mergeado), com saída real do `docker compose run`.

## 8. Ações proibidas
- Commitar em `main`, merge, force-push, `git add -A`/`git add .`.
- Copiar `.env` ou chaves para a imagem; `ARG`/`ENV` com segredos; push de imagem.
- Adicionar dependências Python; qualquer referência a GCP.

## 9. Regra de escalonamento
**Se a spec for ambígua, pare e pergunte; não improvise.** Abra o PR em rascunho e comente citando a seção.

## 10. PR
Título: `build(docker): Dockerfile e compose do protótipo local [pendente]`
