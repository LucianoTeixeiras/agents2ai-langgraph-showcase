# ADR-005 — Guardrails de segurança do agente

| Campo | Valor |
|-------|-------|
| **Status** | Aceita (2026-10-16, Luciano) |
| **Data** | 2026-10-16 |
| **Decisores** | Luciano Teixeira (aceite) · Dev Grok Bot (proposta) |
| **Jira** | pendente — D-01 (projeto novo `A2LG` a ser criado por Luciano) |
| **Relacionados** | PRD e System Design `docs/contracts/202610/16/`; ADR-002, ADR-003 |

## Contexto (2026-10-16)
O LLM gera SQL a partir de texto livre (superfície de prompt injection). O repositório é público e representa a marca Agents2AI.

## Decisão
1. **Guardrail determinístico pós-LLM** (`validate_sql`, sqlglot): 1 statement; somente SELECT/CTE/UNION; nenhum DML/DDL/`Command`; tabelas na allowlist; colunas PII bloqueadas (inclusive via `SELECT *`); `LIMIT` injetado/reduzido a `max_limit=1000`.
2. Parse falho = reprovado (fail closed).
3. Execução DuckDB em conexão `read_only=True`; BigQuery com dry-run + `maximum_bytes_billed`.
4. Custo acima de `MAX_BYTES_WITHOUT_APPROVAL` exige aprovação humana (interrupt); negação encerra sem executar.
5. Retry de geração limitado a 3 tentativas totais.
6. Segredos só via env/`.env` (ignorado); gitleaks no CI; logs nunca contêm chave, prompt completo ou linhas de resultado.
7. Resposta ao usuário mostra no máximo 50 linhas.
8. Proibido no repositório: dados, nomes ou projetos GCP de clientes.

## Alternativas consideradas
| Alternativa | Prós | Contras | Por que não |
|-------------|------|---------|-------------|
| **Escolhida:** sqlglot + read-only + HITL | defesa em profundidade | manutenção da allowlist | — |
| Confiar no prompt | simples | inseguro | inaceitável |
| Regex de palavras proibidas | trivial | fácil de burlar | frágil |

## Consequências
- **Positivas:** O2/O3 do PRD testáveis. **Negativas:** algumas perguntas legítimas recusadas.

## Conformidade
`tests/test_guardrails.py` (≥ 12 casos adversariais), `tests/test_hitl.py`, job gitleaks.

## Rastreabilidade
SPEC-LG-02, SPEC-LG-03, SPEC-LG-05.
