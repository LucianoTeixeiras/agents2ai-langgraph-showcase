# SPEC-LG-01 — Criar a fundação: dependências, settings, estado e grafo com nós stub

| Campo | Valor |
|-------|-------|
| **ID** | SPEC-LG-01 |
| **Status** | Pronta |
| **PRD / System Design / ADRs** | `prd-agente-nl2sql-langgraph.md` · `system-design-agente-nl2sql-langgraph.md` §2, §4, §5, §8, §10 · ADR-001, ADR-003 |
| **Jira** | pendente — D-01 (projeto novo `A2LG` a ser criado por Luciano) |
| **Branch** | `feat/lg-01-fundacao` |
| **Executor sugerido** | `spec-executor` em Cursor + Composer |
| **Risco QA** | `qa:risk-low` |
| **Depende de** | nenhuma |
| **Tamanho máximo** | ≤ 300 linhas de diff, ≤ 12 arquivos (acima disso: pare e escale) |
| **Commit de referência** | `main` após merge do PR de planejamento |

## 1. Objetivo
`build_graph(deps)` compila um `StateGraph(AgentState)` com os 8 nós do System Design §1 (implementações stub determinísticas) e as arestas condicionais; com `InMemorySaver`, `invoke` de uma pergunta termina com `answer` preenchido, e um custo acima do limite pausa em `__interrupt__`.

## 2. Fora de escopo
LLM real, sqlglot, DuckDB, BigQuery, CLI, tracing (só o campo `trace`). Não editar README/AGENTS.md.

## 3. Contexto mínimo
Repo novo: `src/showcase/__init__.py` (`__version__`), `tests/test_placeholder.py`, `pyproject.toml` (sem dependências). API verificada em langgraph 1.2.15:
```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver
# pausa: valor = interrupt({...}); retomada: graph.invoke(Command(resume=True), cfg)
# cfg = {"configurable": {"thread_id": "t1"}}; resultado pausado tem chave "__interrupt__"
```

## 4. Arquivos e interfaces
| Arquivo | Ação | O que muda |
|---------|------|------------|
| `pyproject.toml` | alterar | `dependencies = ["langgraph==1.2.15", "pydantic-settings>=2.6"]` |
| `uv.lock` | alterar | `uv lock` |
| `.env.example` | criar | variáveis do System Design §8, **valores vazios ou padrão**, sem chaves |
| `src/showcase/settings.py` | criar | `class Settings(BaseSettings)` com os campos do §8 (tipos: str/int/bool/Literal), `model_config = SettingsConfigDict(env_file=".env", extra="ignore")`; `llm_api_key: str = ""`; `llm_model: str = "nvidia/nemotron-3.5-lightning"` (D-03); **sem** campos GCP/BigQuery (vêm na SPEC-LG-08) |
| `src/showcase/state.py` | criar | `AgentState` exatamente como System Design §2 |
| `src/showcase/deps.py` | criar | `Protocol LLMClient.complete(self, system: str, user: str) -> str`; `Protocol SqlExecutor` com `catalog() -> dict[str, list[str]]`, `estimate_bytes(sql: str) -> int`, `run(sql: str, max_rows: int) -> list[dict]`; `@dataclass Deps(llm, executor, settings)` |
| `src/showcase/nodes/__init__.py` | criar | reexporta os nós |
| `src/showcase/nodes/core.py` | criar | 8 funções `(state: AgentState, deps: Deps) -> dict` (stubs abaixo) |
| `src/showcase/graph.py` | criar | `build_graph(deps: Deps, checkpointer=None)` + funções de rota |
| `tests/fakes.py` | criar | `FakeLLM(responses: list[str])` (pop em ordem) e `FakeExecutor(bytes_=0, rows=[{"n": 1}])` |
| `tests/test_graph.py` | criar | testes da §6 |
| `tests/test_placeholder.py` | **não tocar** | |

Stubs: `classify_intent` → `{"intent": "out_of_scope"}` se a pergunta estiver vazia, senão `"sql"`; `retrieve_schema` → texto `"tabela: col1, col2"` de `deps.executor.catalog()`; `generate_sql` → `{"sql": deps.llm.complete("sql", question), "attempts": state.get("attempts", 0) + 1}`; `guardrail` → `{"guardrail_errors": []}`; `estimate_cost` → `{"estimated_bytes": deps.executor.estimate_bytes(sql)}`; `human_approval` → `decision = interrupt({"sql": ..., "estimated_bytes": ...}); return {"approved": bool(decision)}`; `execute_sql` → `{"rows": deps.executor.run(sql, 50)}`; `answer` → recusa fixa `"Desculpe, só respondo perguntas sobre o dataset de e-commerce."` para out_of_scope, `"Consulta não executada: aprovação negada."` se `approved is False`, `"Não consegui gerar um SQL seguro."` se houver `guardrail_errors`, senão `f"{len(rows)} linha(s)."`.
Rotas: `route_intent`, `route_guardrail` (erros e attempts<3 → `generate_sql`; erros → `answer`; ok → `estimate_cost`), `route_cost` (`estimated_bytes > settings.max_bytes_without_approval` → `human_approval`), `route_approval`. Nós ligados via `functools.partial(fn, deps=deps)`.

## 5. Passo a passo
1. Branch a partir de `main`. 2. Editar `pyproject.toml`, `uv lock`. 3. Criar arquivos na ordem da tabela. 4. Portões. 5. PR.

## 6. Testes (`tests/test_graph.py`)
- `test_fluxo_feliz_duckdb`: custo 0 → `answer == "1 linha(s)."`, sem `__interrupt__`.
- `test_fora_de_escopo`: pergunta `""` → recusa fixa; `FakeLLM` não consumido.
- `test_interrupt_e_retomada_aprova`: bytes acima do limite → `__interrupt__[0].value["estimated_bytes"]`; `Command(resume=True)` → `"1 linha(s)."`.
- `test_retomada_nega`: `Command(resume=False)` → mensagem de negação; `FakeExecutor.run` não chamado.
- `test_settings_padroes`: `Settings()` sem env → `backend == "duckdb"`, `max_bytes_without_approval == 1_000_000_000`, `llm_model == "nvidia/nemotron-3.5-lightning"`.

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

Título: `feat(grafo): fundação LangGraph com estado, settings e nós stub [pendente]`
