# agents2ai-langgraph-showcase

> 🇧🇷 Agente de analytics em linguagem natural → SQL construído com **LangGraph**, sobre **dados públicos** (DuckDB local + BigQuery public datasets). Demonstra grafo com estado tipado, checkpointing, aprovação humana (human-in-the-loop) para consultas caras, guardrails de SQL, evals e tracing.
>
> 🇺🇸 Natural-language-to-SQL analytics agent built with **LangGraph** over **public data only** (local DuckDB + BigQuery public datasets). Showcases typed graph state, checkpointing, human-in-the-loop approval for costly queries, SQL guardrails, evals and tracing.

**Status:** planejamento (v0). Arquitetura e specs em `docs/` · *planning stage; architecture and specs live in `docs/`.*

## Por que / Why
Projeto vitrine de **Luciano Teixeira** — AI & Data Engineer, fundador da **Agents2AI**. Mostra como levar um agente de dados do protótipo à produção com segurança e custo controlado.
*Showcase project by Luciano Teixeira, AI & Data Engineer and founder of Agents2AI.*

## Stack (planejada / planned)
Python 3.12 · uv · LangGraph · sqlglot · DuckDB · BigQuery (public datasets) · LLM OpenAI-compatível (OpenRouter) · pytest · ruff · gitleaks.

## Rodando / Running
```bash
uv sync --group dev
uv run ruff check .
uv run pytest -q
```

## Licença / License
MIT — ver `LICENSE`.
