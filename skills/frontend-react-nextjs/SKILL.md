---
name: frontend-react-nextjs
description: Índice das referências de bibliotecas deste projeto (Next.js, React, tRPC, Tailwind CSS, Zod, TanStack Query, TypeScript), coletadas do Context7. Use antes de escrever código para consultar a API atual da biblioteca em vez de confiar na memória.
---

# frontend-react-nextjs — referências de bibliotecas do projeto

Cada pasta é uma biblioteca com um `SKILL.md` próprio (índice interno) e uma pasta
`references/` com os tópicos. O conteúdo foi obtido do Context7 com o CLI `ctx7`
em 2026-11-10 e organizado para consulta antes de escrever código.

## Quando consultar o quê

| Tecnologia | Pasta | Consulte quando | Fonte (Context7) |
| --- | --- | --- | --- |
| Next.js 15 (App Router) | `nextjs/` | rotas, layouts, Server/Client Components, data fetching, caching, Server Actions, route handlers, metadata | `/vercel/next.js` |
| React 19 | `react/` | componentes, props/children, hooks, estado, efeitos, memoização, actions | `/reactjs/react.dev` |
| tRPC 11 | `trpc/` | procedures, routers, contexto/middleware, validação de input, erros, cliente React Query, chamadas no server | `/trpc/trpc` |
| Tailwind CSS 4 | `tailwindcss/` | tema CSS-first (`@theme`), variantes, utilitários, `@utility`, `@apply`, migração de config JS | `/tailwindlabs/tailwindcss.com` |
| Zod | `zod/` | schemas, `parse`/`safeParse`, refine/transform, mensagens de erro, inferência de tipos | `/colinhacks/zod` |
| TanStack Query v5 | `tanstack-query/` | `useQuery`/`useMutation`, cache (`staleTime`/`gcTime`), invalidação, updates otimistas, prefetch/SSR | `/tanstack/query` |
| TypeScript 5.8 | `typescript/` | tipos, generics, narrowing, utility types, opções `strict`/`noUncheckedIndexedAccess`/`verbatimModuleSyntax` | `/microsoft/typescript-website` |

Prisma não está aqui: o projeto já tem o skill dedicado
`.opencode/skills/prisma-client-api/`. Consulte-o para schema e queries.

Roteamento também não tem pasta aqui: o roteador do projeto é o próprio Next.js App Router.
As decisões e os limites (route groups, rotas dinâmicas, guards por cargo, quando não
adicionar biblioteca de rotas) ficam no skill `.opencode/skills/nextjs-app-router/`.

## Como usar

1. Abra o `SKILL.md` da tecnologia para achar o arquivo certo (tabela "Onde procurar").
2. Leia só o `references/*.md` que cobre a tarefa; não carregue a pasta inteira.
3. Se a resposta não estiver lá, pergunte à fonte viva em vez de adivinhar:

```bash
ctx7 docs /vercel/next.js "<pergunta específica>"
ctx7 docs /trpc/trpc "How to prefetch a query on the server"
ctx7 search "<pergunta>" --library Tailwind\ CSS
```

4. Ao descobrir um padrão que falta nos arquivos, adicione-o no `references/` certo
   (mesmo formato: `> Fonte: ... via ctx7 docs`).

## Regras

- **A doc é a fonte, não a memória.** Toda afirmação de API nos arquivos veio de uma
  consulta ao `ctx7` registrada no topo do arquivo. Se algo não está documentado, o texto
  diz para confirmar com `ctx7 docs` em vez de inventar.
- **Versão instalada manda.** O Context7 serve a documentação mais recente; este repo fixa
  versões em `package.json`. Divergências conhecidas e sinalizadas nos arquivos:
  - `zod@^3.24.2` no repo x documentação v4 (construtores de topo como `z.email()`,
    `{ error }`, `z.strictObject` são v4) — `zod/SKILL.md` explica como confirmar.
  - `next@^15.2.3` x índice `canary` do Context7 (algumas APIs são Next.js 16, ex.:
    `error.tsx` recebendo `retry`).
  - `tailwindcss@^4.0.15` x recursos marcados como v4.1 (`@source not`, `@source inline`).
  - `typescript@^5.8.2` x handbook, que nem sempre marca a versão de cada recurso.
- **Não duplique entre tecnologias.** tRPC + React Query: o fluxo de cache está em
  `trpc/references/react-query.md`; regras gerais de cache estão em
  `tanstack-query/references/caching.md`.
- **pt-BR** nos textos, nomes de API e código em inglês.

## Atualizar uma referência

```bash
# 1. descubra o ID (e a versão, se necessário)
ctx7 library next.js "app router caching" --json | jq '.[0].id, .[0].versions'

# 2. consulte o tópico e reescreva só a seção afetada
ctx7 docs /vercel/next.js/v15.5.28 "How to revalidate a tag"

# 3. valide a formatação
npx prettier --write ".opencode/skills/frontend-react-nextjs/**/*.md"
```
