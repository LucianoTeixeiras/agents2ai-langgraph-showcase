# PRD — Agente NL→SQL em LangGraph sobre dados públicos (showcase)

| Campo | Valor |
|-------|-------|
| **Status** | Em revisão |
| **Autor (arquiteto)** | Dev Grok Bot (planner) |
| **Dono do produto** | Luciano Teixeira |
| **Jira (épico)** | pendente — projeto Jira do repositório ainda não definido (D-01) |
| **Nível de risco QA** | `qa:risk-medium` (gera e executa SQL; repositório público) |
| **Documentos relacionados** | `system-design-agente-nl2sql-langgraph.md`, ADR-001…005, specs `spec-lg-01…06` |

## 1. Problema
Recrutadores e clientes da Agents2AI precisam ver, em poucos minutos e em código aberto, que Luciano sabe construir agentes de dados **seguros, auditáveis e baratos** com o framework de orquestração mais pedido no mercado (LangGraph). O produto principal (`agents2ai-data-copilot`) usa Pydantic AI; falta uma vitrine LangGraph no mesmo domínio.

## 2. Objetivo e métricas de sucesso

| ID | Objetivo | Métrica | Linha de base | Alvo | Como medir |
|----|----------|---------|---------------|------|------------|
| O1 | Respostas corretas | acurácia de execução no golden set | n/a | ≥ 80% (modelo barato padrão) | `uv run python -m showcase.evals` |
| O2 | Zero SQL perigoso | consultas de escrita/DDL/multi-statement que passam o guardrail | n/a | 0 | `tests/test_guardrails.py` (casos adversariais) |
| O3 | Custo controlado | consultas BigQuery acima do limite executadas sem aprovação | n/a | 0 | `tests/test_hitl.py` |
| O4 | Retomada | execução retomada após interrupt com mesmo `thread_id` | n/a | 100% dos testes | `tests/test_hitl.py` |
| O5 | Reprodutível | CI verde sem chave de LLM nem credencial GCP | n/a | sim | CI (LLM fake + BigQuery mockado) |
| O6 | Vitrine | README com GIF/diagrama e "como rodar" em ≤ 3 comandos | n/a | sim | revisão do arquiteto |

## 3. Usuários e cenários
- Como **recrutador técnico**, quero ler o README e o grafo em 5 minutos para avaliar a senioridade do autor.
- Como **cliente potencial**, quero perguntar "qual a receita por categoria em 2023?" e ver SQL, custo estimado e resposta.
- Como **operador**, quero aprovar ou negar uma consulta cara antes que ela rode no BigQuery.
- Como **dev avaliador**, quero rodar os evals e ver acurácia e latência por nó.

## 4. Escopo
### 4.1 Dentro do escopo
- CLI (`uv run showcase ask "<pergunta>"`) com fluxo de aprovação interativo.
- Grafo LangGraph com estado tipado, roteamento condicional, retry limitado de geração de SQL.
- Checkpointer (memória nos testes, SQLite local na CLI) e interrupt para aprovação humana.
- Guardrail de SQL com sqlglot: só leitura, um statement, `LIMIT` obrigatório, tabelas permitidas, colunas PII bloqueadas.
- Execução em DuckDB local (dataset sintético com schema de dataset público) e BigQuery **somente** `bigquery-public-data.*` com dry-run de custo.
- Evals offline (LLM fake) e online opcionais; tracing por nó em log JSON; LangSmith/OpenTelemetry opcionais por variável de ambiente.

### 4.2 Fora do escopo (explícito)
- UI web, API HTTP, streaming, autenticação multiusuário (fase 2, se houver).
- Qualquer dado, nome ou projeto GCP de cliente; escrita em qualquer banco.
- Fine-tuning, RAG vetorial, multiagente, LangGraph Platform/Cloud.
- Portar código do `agents2ai-data-copilot` (é vitrine independente; só o domínio é o mesmo).

## 5. Requisitos funcionais
- **RF-01** Classificar intenção (`sql` | `out_of_scope`); fora de escopo responde recusa educada sem chamar SQL.
- **RF-02** Montar contexto de schema a partir do catálogo permitido.
- **RF-03** Gerar SQL no dialeto do backend ativo (`duckdb` | `bigquery`).
- **RF-04** Validar SQL (guardrail); até 2 tentativas de regeneração com o erro como feedback; depois, recusa.
- **RF-05** Estimar custo (DuckDB: 0; BigQuery: bytes via dry-run).
- **RF-06** Acima de `MAX_BYTES_WITHOUT_APPROVAL`, pausar com `interrupt()` e retomar com `Command(resume=...)`.
- **RF-07** Executar e responder em linguagem natural, exibindo SQL e linhas (máx. 50).
- **RF-08** Registrar trace por nó (nome, duração ms, tokens quando houver) no estado e em log JSON.

## 6. Requisitos não funcionais
- Python 3.12, uv, ruff, pytest; CI sem segredos; gitleaks.
- Latência p95 < 15 s com modelo barato (medida nos evals online).
- Custo: modelo econômico por padrão via OpenRouter; BigQuery limitado por `maximum_bytes_billed`.

## 7. Segurança e privacidade
Ver ADR-005. Somente dados públicos; segredos só em `.env`; nada de cliente; SQL só leitura; PII bloqueada.

## 8. Riscos
| Risco | Mitigação |
|-------|-----------|
| Prompt injection gera SQL destrutivo | guardrail determinístico pós-LLM (não confia no modelo) |
| Custo BigQuery | dry-run + HITL + `maximum_bytes_billed` |
| API do LangGraph muda | versões fixadas no `uv.lock`; ADR-001 registra a versão verificada |
| Modelo barato erra SQL | retry com feedback + evals |

## 9. Decisões em aberto
Ver `docs/DECISIONS-LOG.md` → "Decisões pendentes" (D-01…D-05).
