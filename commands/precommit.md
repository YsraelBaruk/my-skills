---
description: "Executa o gate obrigatório de verificação antes de commit"
---

Execute o gate obrigatório definido no `AGENTS.md` da raiz do projeto antes de
criar qualquer commit.

**Escopo informado pelo usuário:** $ARGUMENTS

## Passos

1. Leia o gate em `AGENTS.md`.
2. Determine o escopo real com `git status --short`, `git diff --name-only` e
   `git diff --cached --name-only`.
3. Acione as skills aplicáveis:
   - `codehealth-mcp` — sempre.
   - `verification-loop` — sempre.
   - `security-review` — sempre.
   - `flutter-dart-code-review` — se houver mudanças em `app/`.
   - `flask-verification` — se houver mudanças em `backend/`.
4. Corrija as falhas e repita as verificações afetadas.
5. Apresente o relatório de evidências com PASS, FAIL, N/A ou INDISPONÍVEL.

Não crie o commit se qualquer verificação aplicável ainda estiver falhando. Se
`codehealth-mcp` estiver indisponível, registre o motivo e não simule o
resultado. Se o usuário tiver pedido apenas a verificação, pare após o
relatório.
