---
description: "Analisa módulos, cria commits, faz push e abre uma PR"
---

Execute o fluxo completo de commits e pull request descrito abaixo.

**Instruções adicionais do usuário:** $ARGUMENTS

## Pré-condições

1. Leia o padrão de commits em `AGENTS.md`. Este comando não executa o gate de
   verificação: ele assume que o gate já foi executado.
2. Confirme que `gh` está autenticado:

   ```bash
   gh auth status
   ```

3. Descubra a branch padrão e o remoto:

   ```bash
   git remote -v
   git fetch origin
   BASE=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name)
   ```

4. Verifique se há PR aberta para a branch atual:

   ```bash
   gh pr view --json number,state,url,title
   ```

5. Pare imediatamente se houver:
   - arquivos com segredos ou credenciais;
   - banco local, `.env`, `.venv`, build, cache ou artefato gerado;
   - conflitos de merge;
   - mudanças não relacionadas que não possam ser separadas com segurança.

## 1. Analisar os módulos

Inspecione o estado real do repositório:

```bash
git status --short
git diff --name-only
git diff --cached --name-only
git ls-files --others --exclude-standard
git diff --stat
```

Agrupe as alterações por módulo e por intenção:

- `backend/` — API Flask, domínio, persistência, migrações e testes.
- `app/` — Flutter, telas, widgets, estado, modelos e testes.
- `docs/` — contrato da API, domínio e documentação.
- `openspec/` — propostas, especificações e tarefas.
- `.opencode/` e skills — configuração do harness e workflows.
- Raiz e configuração — `README.md`, `.gitignore`, dependências e tooling.

Dentro de `backend/` e `app/`, separe subáreas quando as mudanças forem
independentes. Não separe arquivos que precisam estar juntos para manter build
ou testes verdes.

Monte um plano de commits com:

- ordem de dependência: backend → app → documentação → OpenSpec → tooling;
- arquivos exatos de cada commit;
- tipo e escopo de cada commit;
- justificativa de qualquer arquivo que fique fora dos commits.

Apresente o plano antes de começar. Se o usuário já tiver autorizado o fluxo
completo, prossiga sem pedir nova confirmação.

## 2. Criar a branch, se necessário

Se a branch atual for a branch padrão, crie uma branch de trabalho:

```bash
git switch -c "<tipo>/<slug>"
```

Use `feat/` para funcionalidade, `fix/` para correção, `docs/` para documentação
e `chore/` para manutenção. Se já estiver em uma branch de trabalho, mantenha-a.

Não faça rebase, amend, reset ou force push sem confirmação explícita.

## 3. Criar os commits

Para cada grupo do plano:

1. Adicione somente os caminhos explícitos:

   ```bash
   git add -- caminho/1 caminho/2
   ```

2. Confira o que será commitado:

   ```bash
   git diff --cached --stat
   git diff --cached
   git diff --cached --check
   ```

3. Use o padrão `tipo(escopo): assunto` definido em `AGENTS.md`.
4. Use o skill `conventional-commit-review` para validar a mensagem quando
   disponível.
5. Crie o commit com mensagem e corpo explicando o motivo e a verificação:

   ```bash
   git commit -m "<tipo>(<escopo>): <assunto>" -m "<corpo>"
   ```

Nunca use `git add -A`, `git add .` ou `git commit -a`. Nunca inclua arquivos
gerados, bancos locais, caches, credenciais ou mudanças não relacionadas.

Se um commit falhar, não continue para o próximo sem corrigir a causa.

## 4. Fazer o push

Depois que todos os commits forem criados:

```bash
git push -u origin HEAD
```

Nunca use `--force` ou `--force-with-lease` sem confirmação explícita. Se o push
for rejeitado, pare e mostre o erro; não reescreva o histórico automaticamente.

## 5. Criar a pull request

Antes de criar:

```bash
gh pr view --json number,state,url,title
```

Se já existir uma PR aberta para a branch, não crie duplicata. Atualize o título
e o corpo com `gh pr edit` e informe o link.

Caso contrário, crie a PR com título curto no padrão Conventional Commits e
corpo em português do Brasil:

```bash
gh pr create \
  --base "$BASE" \
  --head "$(git branch --show-current)" \
  --title "<tipo>(<escopo>): <resumo>" \
  --body-file - <<'EOF'
## Resumo

<o que foi feito e por quê>

## Módulos alterados

- `backend/`: <mudanças>
- `app/`: <mudanças>
- `docs/`: <mudanças>
- `openspec/`: <mudanças>

## Commits

- `<hash>` — <mensagem>

## Observações

<riscos, limitações ou mudanças de contrato>
EOF
```

Use `--draft` somente se o usuário pedir. Não declare testes, cobertura ou
aprovação que não tenham sido executados.

## 6. Relatar o resultado

Ao final, informe:

- branch utilizada;
- commits criados, com hash e mensagem;
- URL da pull request;
- qualquer arquivo ou mudança que tenha ficado fora dos commits.
