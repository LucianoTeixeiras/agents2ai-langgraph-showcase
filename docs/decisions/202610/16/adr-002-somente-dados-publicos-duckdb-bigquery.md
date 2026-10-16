# ADR-002 — Somente dados públicos: DuckDB local e BigQuery public datasets

| Campo | Valor |
|-------|-------|
| **Status** | Aceita (2026-10-16, Luciano) |
| **Data** | 2026-10-16 |
| **Decisores** | Luciano Teixeira (aceite) · Dev Grok Bot (proposta) |
| **Jira** | pendente — D-01 (projeto novo `A2LG` a ser criado por Luciano) |
| **Relacionados** | PRD e System Design `docs/contracts/202610/16/`; ADR-005 |

## Contexto (2026-10-16)
Repositório público, vitrine de consultoria. Qualquer dado ou identificador de cliente seria vazamento e dano reputacional. Precisamos de CI sem credenciais e de demo realista.

## Decisão
1. Backend padrão: **DuckDB** em arquivo local gerado por script determinístico com dados **sintéticos** no schema de `bigquery-public-data.thelook_ecommerce` (orders, order_items, products, users).
2. **Fase 2 GCP (D-04, Bloqueada):** backend opcional **BigQuery** restrito à allowlist de tabelas `bigquery-public-data.thelook_ecommerce.{orders,order_items,products,users}`; billing num projeto GCP **de testes da Agents2AI** (D-04), nunca de cliente.
3. Toda execução BigQuery faz dry-run antes e usa `maximum_bytes_billed`.
4. CI usa só DuckDB; BigQuery é mockado nos testes.
5. Nenhum arquivo `.duckdb`/`.sqlite` é versionado.

**Fase 1 (protótipo, D-04):** roda 100% em Docker com DuckDB sintético local; nenhuma credencial ou projeto GCP.

## Alternativas consideradas
| Alternativa | Prós | Contras | Por que não |
|-------------|------|---------|-------------|
| **Escolhida:** DuckDB sintético + BQ público | zero custo no CI, demo real opcional | dados sintéticos | — |
| Baixar parquet público no CI | dados reais | rede no CI, flaky | instável |
| Só BigQuery | realista | exige credencial | CI sem segredos |
| Postgres em container | popular | infra extra | complexidade |

## Consequências
- **Positivas:** zero risco de dado de cliente; CI barato.
- **Negativas:** acurácia em dados sintéticos ≠ produção.
- **Riscos residuais:** schema do thelook mudar → verificar antes da SPEC-05 (não verificado nesta data).

## Conformidade
Teste de guardrail rejeita tabela fora da allowlist; gitleaks; revisão bloqueia qualquer `project_id` que não seja da Agents2AI.

## Rastreabilidade
SPEC-LG-03, SPEC-LG-05.
