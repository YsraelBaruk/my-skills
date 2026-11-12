---
name: frontend-react-nextjs-tanstack-query
description: Referência atual do TanStack Query v5 obtida do Context7. Use ao configurar cache, escrever useQuery/useMutation, invalidar dados ou fazer updates otimistas neste projeto.
---

# TanStack Query v5 (React Query)

Referência derivada exclusivamente de consultas ao Context7 (`ctx7 docs /tanstack/query`),
feitas em 2026-11-10. O `package.json` deste repositório declara
`@tanstack/react-query: ^5.69.0`; o conjunto `node_modules` não está instalado aqui, então
a versão resolvida não foi verificada.

## Avisos de versão

O Context7 serve a documentação da branch `main` do TanStack Query. Isso significa que
trechos ocasionais são mais novos que a linha 5.69:

- `queryClient.query({ queryKey, queryFn })` aparece substituindo `prefetchQuery` /
  `ensureQueryData` e é descrito como remoção futura (v6). Na prática, para v5 use
  `queryClient.prefetchQuery({ queryKey, queryFn })`, que continua funcionando.
- Os callbacks de mutação aparecem com `onMutateResult` no lugar de `context`
  (`(data, variables, onMutateResult, context)`). Se o TypeScript reclamar de aridade,
  consulte a tipagem instalada e use a forma antiga `(data, variables, context)`.
- Sinais claros de v4 (não usar): `isLoading`, `cacheTime`, `keepPreviousData: true`,
  `useQuery(key, fn, options)`, `onSuccess`/`onError` em `useQuery`.

## Tarefa -> arquivo

| Tarefa | Arquivo |
| --- | --- |
| Configurar `QueryClient`/`QueryClientProvider`, defaults, `useQuery`, `queryKey`, `enabled`, `select`, `placeholderData`, `useQueries`, `useInfiniteQuery`, `useSuspenseQuery` | `references/queries.md` |
| Escrever `useMutation`, callbacks, `mutateAsync`, invalidar, update otimista, rollback, erros | `references/mutations.md` |
| Entender `staleTime` x `gcTime`, gatilhos de refetch, prefetch, deduplicação, GC, SSR/hidratação | `references/caching.md` |

## Como isso se conecta ao tRPC deste repo

O cliente já está montado; nunca crie outro `QueryClient` em componente.

- `src/trpc/query-client.ts` — `createQueryClient()` define `staleTime: 30 * 1000` e os
  hooks de `dehydrate`/`hydrate` com SuperJSON. Alterações de default vão aqui.
- `src/trpc/react.tsx` — `TRPCReactProvider` cria o `QueryClientProvider` e o
  `api.Provider`; o `QueryClient` é singleton no browser e novo por request no servidor.
- `src/trpc/server.ts` — `api` e `HydrateClient` (`createHydrationHelpers`) para Server
  Components; `api.post.getLatest.prefetch()` + `<HydrateClient>` equivale ao
  `prefetchQuery` + `HydrationBoundary` genéricos descritos nas referências.

Uso no cliente (`src/app/_components/post.tsx` é o exemplo canônico):

```tsx
"use client";

import { api } from "~/trpc/react";

const [latestPost] = api.post.getLatest.useSuspenseQuery(); // data sempre definida
const utils = api.useUtils();

const createPost = api.post.create.useMutation({
  onSuccess: async () => {
    await utils.post.invalidate(); // invalida o router inteiro
  },
});

createPost.mutate({ name });
createPost.isPending; // v5: isPending, não isLoading
```

Regras práticas para este repo:

- Prefira `api.<router>.<proc>.useSuspenseQuery()` em componentes sob `<Suspense>`; use
  `useQuery` quando precisar de estados de carregamento/erro locais.
- A `queryKey` é derivada do caminho do router mais o input. Não invente chaves manuais
  para dados do tRPC; para invalidar, use `utils.<router>.invalidate()`,
  `utils.<router>.<proc>.invalidate(input)` ou `utils.invalidate()`.
- `utils.<proc>.setData(input, updater)` / `getData(input)` existem para update otimista.
- Server Component: `await api.x.prefetch(input)` antes de `<HydrateClient>`, e
  consuma com `useSuspenseQuery` no cliente para evitar mismatch de hidratação.
