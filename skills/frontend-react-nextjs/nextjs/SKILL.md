---
name: frontend-react-nextjs-nextjs
description: Referência atual do Next.js App Router obtida do Context7. Use ao criar rotas, layouts, Server/Client Components, data fetching, caching, Server Actions ou metadata neste projeto Next.js 15.
---

# Next.js App Router — referência do projeto

Fonte: `/vercel/next.js` via `ctx7 docs` (consultado em 2026-11-10). O índice principal é
`canary`, então alguns trechos já refletem Next.js 16; as APIs que mudaram entre 15 e 16
foram conferidas contra `/vercel/next.js/v15.5.28`. O projeto fixa Next.js `^15.2.3` com
React 19, TypeScript `strict`, tRPC, Prisma e Tailwind 4.

Quando a resposta não estiver aqui, confirme na fonte:

```bash
ctx7 docs /vercel/next.js "<pergunta específica>"
ctx7 docs /vercel/next.js/v15.5.28 "<pergunta específica>"
```

## Onde procurar

| Tarefa | Arquivo |
| --- | --- |
| Rotas, layouts aninhados, route groups | `references/app-router.md` |
| `page.tsx`, `params`, `searchParams`, `generateStaticParams` | `references/app-router.md` |
| `loading.tsx`, `error.tsx`, `not-found.tsx`, `notFound()` | `references/app-router.md` |
| Route Handlers (`route.ts`), `NextRequest`, status/cache | `references/app-router.md` |
| `redirect()`, navegação programática | `references/app-router.md` |
| Server Components x Client Components, `'use client'` | `references/rendering.md` |
| `'use server'`, composição server/client, serialização de props | `references/rendering.md` |
| Suspense, streaming, `loading.tsx` como boundary | `references/rendering.md` |
| `useRouter`, `usePathname`, `useSearchParams` | `references/rendering.md` |
| `fetch`, cache, `revalidatePath`, `revalidateTag` | `references/data-fetching.md` |
| Server Actions, `useActionState`, validação com Zod | `references/data-fetching.md` |
| Consulta direta ao banco em Server Component | `references/data-fetching.md` |
| `use cache`, `cacheLife`, `cacheTag` | `references/data-fetching.md` |
| `next.config`, `next/image`, `next/font` | `references/config-and-metadata.md` |
| `metadata`, `generateMetadata`, `metadataBase` | `references/config-and-metadata.md` |

## Pegadinhas encontradas no App Router

- **`params` e `searchParams` são `Promise`.** Em Server Component use `await params`; em
  Client Component (que não pode ser `async`) use `use(params)` de `react`.
- **Layout não recebe `searchParams`.** Layouts não re-renderizam na navegação; leia query
  string na page ou com `useSearchParams` em um Client Component.
- **`fetch` não é cacheado por padrão no Next.js 15.** Cache é opt-in com
  `cache: 'force-cache'` ou `next: { revalidate }`.
- **`GET` de Route Handler não é cacheado por padrão.** Para cachear, exporte
  `dynamic = 'force-static'` (ou `revalidate`/`generateStaticParams`).
- **`error.tsx` da v15 recebe `error` e `reset`.** O `retry` que aparece na doc `canary`
  é Next.js 16; no projeto use `reset()`. O arquivo precisa de `'use client'`.
- **`page` e `route` não coexistem no mesmo segmento.** Um arquivo `route.ts` não pode
  ficar no mesmo nível de um `page.tsx`.
- **`use cache` não está disponível neste repo.** Exige
  `experimental.cacheComponents: true`; o `next.config.js` atual está vazio. `cacheLife` e
  `cacheTag` só valem junto desse flag (na v15, `unstable_cacheLife`/`unstable_cacheTag`).
- **Server Actions são endpoints POST públicos.** Valide entrada e autorização dentro de
  cada action; nunca confie em `id`/objeto inteiro vindo do cliente.
- **`redirect()` lança exceção.** Chame fora de `try/catch` e depois de
  `revalidatePath`/`revalidateTag`.
- **`revalidateTag` na v15 recebe só a tag** (`string | string[]`). A assinatura com segundo
  argumento (`'max'`) que aparece no índice `canary` é de versão mais nova.

## Padrão do repositório

Dados no cliente passam por tRPC + React Query; dados no servidor seguem `src/server/**`.
O Next.js permite consultar o Prisma direto em Server Component, mas neste repo prefira o
router/procedure existente antes de criar uma segunda camada de acesso.
