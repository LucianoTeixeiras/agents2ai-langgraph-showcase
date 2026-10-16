# Guia para o executor (Cursor / Composer)

> Como executar as specs `spec-lg-*` no Cursor com o modelo econômico Composer. Arquiteto (planner): Dev Grok Bot. Aceite e merge: Luciano.

## 1. Preparação (uma vez)
1. Copie a regra do executor do agent-kit **sem commitar**: `agent-kit/integrations/cursor/.cursor/rules/spec-executor.mdc` → `agents2ai-langgraph-showcase/.cursor/rules/spec-executor.mdc` (adicione `.cursor/` ao seu exclude local: `.git/info/exclude`). Ative anexando `@spec-executor`. Persona canônica: `agent-kit/.agent-kit/content/agents/spec-executor.agent.md`.
2. Ambiente: `uv sync --locked --group dev`.
3. **Nunca** anexe `.env`, `.secrets/`, `checkpoints/` ou `*.duckdb` ao contexto do Cursor.

## 2. Uma spec por vez
1. Pegue a próxima spec com **Status: Pronta**, na ordem do System Design §10 (01 → 02 → 03 → 04 → 05 → 06). Bloqueada não começa.
2. Chat novo no Cursor (modo Agent, modelo Composer), anexando **só**: `@spec-executor`, o arquivo da spec, os arquivos existentes da tabela §4 e, se citados, o System Design e a ADR.
3. Prompt inicial:
   > Execute exatamente a spec anexada, seguindo a regra spec-executor. Não altere arquivos fora da seção 4. Se algo for ambíguo ou diferente do descrito, pare e me diga qual seção da spec está em conflito.
4. Antes de aceitar: `git diff --stat main` deve listar só os arquivos da §4.

## 3. Portões locais (iguais ao CI)
```bash
uv sync --locked --group dev
uv run ruff check .
uv run pytest -q
```
O CI também roda **gitleaks** em todo o histórico: um segredo commitado reprova o PR (e exige rotação da chave).

## 4. Commit e PR
- `git add caminho` arquivo por arquivo; **nunca** `git add -A`/`git add .`.
- Mensagem/título pt-BR: `<tipo>(<escopo>): <descrição> [pendente]` (até existir Jira — D-01).
- Corpo: §10 da spec (Spec · O que foi feito · Evidência dos portões com saída real · Desvios · Checklist DoD · Risco QA).
- `git push -u origin <branch>` e `gh pr create`. **Não faça merge**: `main` é protegida (CI obrigatório) e o arquiteto revisa com `agent-kit/.agent-kit/content/templates/pr-review-checklist.md`.

## 5. Quando parar e chamar o arquiteto
**Se a spec for ambígua, pare e pergunte; não improvise.** Pare quando: arquivo/símbolo/API não existir como descrito (as specs marcam o que não foi verificado); teste só passar mudando algo fora da §4; diff passar do limite; versão de pacote mudar sem a spec mandar; surgir decisão de produto, segurança, custo ou dependência. Abra PR em rascunho e comente citando a seção.

## 6. Erros comuns do modelo econômico (evite)
- "Aproveitar" para reformatar/renomear arquivos vizinhos.
- Trocar mensagens de erro exatas (ex.: `"forbidden_statement"`) por variações.
- Afrouxar guardrail/allowlist ou marcar teste com `skip` para o CI passar.
- Chamar LLM/BigQuery reais em teste ou colocar `project_id` real no código.
- Colocar efeito colateral **antes** de `interrupt()` (o nó é reexecutado na retomada).
- Dizer que editou sem ter editado: confira com `git diff`.
