# tRPC v11 — React Query

> Fonte: `/trpc/trpc` via `ctx7 docs` (consultado em 2026-11-10).

Escopo: `createTRPCReact`, provider, `useQuery`, `useMutation`, `useUtils`/invalidação,
`useSuspenseQuery`, `useInfiniteQuery` e prefetch com `HydrateClient`. Tudo aqui é Client
Component e vive em `src/trpc/react.tsx` + componentes de rota.

## 1. `createTRPCReact` e o provider

A doc usa `createTRPCReact<AppRouter>()` como único ponto de geração de hooks tipados e o
`Provider` combinado com `QueryClientProvider`:

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { httpBatchLink } from "@trpc/client";
import { createTRPCReact } from "@trpc/react-query";
import { useState } from "react";
import type { AppRouter } from "../server";

export const trpc = createTRPCReact<AppRouter>();

export function App() {
  const [queryClient] = useState(() => new QueryClient());
  const [trpcClient] = useState(() =>
    trpc.createClient({
      links: [
        httpBatchLink({
          url: "http://localhost:3000/trpc",
          async headers() {
            return { authorization: getAuthCookie() };
          },
        }),
      ],
    }),
  );

  return (
    <trpc.Provider client={trpcClient} queryClient={queryClient}>
      <QueryClientProvider client={queryClient}>{/* app */}</QueryClientProvider>
    </trpc.Provider>
  );
}
```

Este repo tem a mesma estrutura em `src/trpc/react.tsx`, com três detalhes próprios:
`httpBatchStreamLink`, `transformer: SuperJSON` **no link** (v11) e QueryClient singleton no
browser / novo no servidor. Não crie outro `createTRPCReact` em outro arquivo.

## 2. `useQuery`

Assinatura: `useQuery(input, opts?)`. O input é opcional quando a procedure não exige entrada.

```tsx
const helloNoArgs = trpc.hello.useQuery();
const helloWithArgs = trpc.hello.useQuery({ text: "client" });
```

Opções específicas de tRPC:

- `trpc.ssr: boolean` — desliga SSR só para essa query.
- `trpc.abortOnUnmount: boolean` — sobrescreve o comportamento global de abort no unmount.
- `trpc.context: Record<string, unknown>` — metadados extras para os links.

Import no repo: `import { api } from "~/trpc/react";` e então `api.post.getLatest.useQuery()`.

## 3. `useMutation`

`useMutation` segue o TanStack: `mutate`, `mutateAsync`, `isPending`, `error`, `data`. O
`onSuccess` recebe o input, o que permite invalidar com o dado correto.

```tsx
const mutation = trpc.post.edit.useMutation({
  onSuccess(input) {
    utils.post.all.invalidate();
    utils.post.byId.invalidate({ id: input.id }); // não invalida outros ids 👍
  },
});

mutation.mutate({ id: 1, title: "novo" });
```

Cuidado: no TanStack Query 5, `isLoading` de mutation virou `isPending`; use `isPending` para desabilitar botão.

## 4. Invalidação com `useUtils`

`trpc.useUtils()` devolve os helpers de cache. A doc mostra os três níveis de invalidação:

```tsx
const utils = trpc.useUtils();

utils.invalidate();                    // 1️⃣ todos os routers
utils.post.invalidate();               // 2️⃣ todo o router `post`
utils.post.byId.invalidate({ id: 1 }); // 3️⃣ só queries com input { id: 1 }
```

Efeito combinado, conforme o exemplo da doc:

```tsx
trpc.user.all.useQuery();             // afetada só por 1️⃣
trpc.post.all.useQuery();             // afetada por 1️⃣ e 2️⃣
trpc.post.byId.useQuery({ id: 1 });   // afetada por 1️⃣, 2️⃣ e 3️⃣
trpc.post.byId.useQuery({ id: 2 });   // afetada por 1️⃣ e 2️⃣, NÃO por 3️⃣
```

Prefira o nível mais específico possível: invalidação ampla derruba cache de telas que não
mudaram. Helpers correlatos citados: `getData` (wrapper de `getQueryData`), `setData`
(wrapper de `setQueryData`), `cancel`, `setInfiniteData`.

## 5. `useSuspenseQuery`

Suspende até os dados existirem. Retorna tupla `[data, query]` — diferente de `useQuery`, que
retorna só o objeto da query.

```tsx
function PostView() {
  const [post, postQuery] = trpc.post.byId.useSuspenseQuery({ id: "1" });
  return <>{post.title}</>;
}
```

Uso: dentro de `<Suspense>` e com prefetch no Server Component (seção 8). O `data` já é
non-nullable; use `postQuery` para `refetch`, `isFetching` etc.

## 6. `useInfiniteQuery`

Exige procedure com entrada de cursor e `getNextPageParam`:

```tsx
const myQuery = trpc.infinitePosts.useInfiniteQuery(
  { limit: 10 },
  {
    getNextPageParam: (lastPage) => lastPage.nextCursor,
    // initialCursor: 1, // opcional
  },
);
```

Resultado traz `data.pages`, `fetchNextPage`, `hasNextPage`.

Atualização de cache de query infinita usa `setInfiniteData`, sempre depois de `cancel`:

```tsx
const utils = trpc.useUtils();

const myMutation = trpc.infinitePosts.delete.useMutation({
  async onMutate(opts) {
    await utils.infinitePosts.list.cancel();

    utils.infinitePosts.list.setInfiniteData({ limit: 10 }, (data) => {
      if (!data) {
        return { pages: [], pageParams: [] };
      }
      return {
        ...data,
        pages: data.pages.map((page) => ({
          ...page,
          items: page.items.filter((item) => item.status === "published"),
        })),
      };
    });
  },
});
```

Note o shape obrigatório `{ pages, pageParams }` quando não há dados — devolver outra coisa
quebra a query infinita.

## 7. Update otimista

O material consultado cobre as peças do padrão (cancelar, ler, escrever cache no `onMutate`),
mas **não** trouxe um exemplo completo com snapshot + rollback em `onError`. Esqueleto
correto, usando apenas os helpers documentados:

```tsx
const utils = trpc.useUtils();

trpc.post.edit.useMutation({
  async onMutate(input) {
    await utils.post.all.cancel();          // evita refetch sobrescrever o otimista
    const previous = utils.post.all.getData();
    utils.post.all.setData(undefined, (old) => optimistic(old, input));
    return { previous };                    // vira o `context` do onError
  },
  onError(_err, _input, context) {
    utils.post.all.setData(undefined, context?.previous); // rollback
  },
  onSettled() {
    void utils.post.all.invalidate();
  },
});
```

Confirme assinaturas de `setData`/`getData` e o rollback com
`ctx7 docs /trpc/trpc "optimistic update onMutate onError rollback setData"` antes de usar em
produção. `setInfiniteData` segue o mesmo padrão, guardando as páginas.

## 8. Prefetch no Server Component + hidratação

O caminho: o Server Component faz prefetch no mesmo QueryClient cacheado por request, envolve
a árvore em `HydrateClient`, e o Client Component usa `useSuspenseQuery` (ou `useQuery`).

```tsx
// app/page.tsx (Server Component)
import { api, HydrateClient } from "~/trpc/server";
import { PostList } from "./_components/post-list";

export default async function Page() {
  void api.post.getLatest.prefetch(); // não await: deixa o Suspense cuidar

  return (
    <HydrateClient>
      <PostList />
    </HydrateClient>
  );
}
```

```tsx
// app/_components/post-list.tsx (Client Component)
"use client";
import { api } from "~/trpc/react";

export function PostList() {
  const [post] = api.post.getLatest.useSuspenseQuery();
  return <>{post?.name ?? "sem posts"}</>;
}
```

Regras da doc:

- `HydrateClient` usa `HydrationBoundary state={dehydrate(queryClient)}` do QueryClient
  cacheado; sem `cache()` no getter, a hidratação não vê o que foi prefetchado.
- `prefetch` **não** é aguardado; combine com `useSuspenseQuery` para o streaming funcionar.
- Para queries infinitas, o prefetch precisa usar `prefetchInfiniteQuery` — o helper da doc
  detecta `queryOptions.queryKey[1]?.type === "infinite"`.
- Neste repo, `createQueryClient` já configura `serializeData`/`deserializeData` com SuperJSON
  e `shouldDehydrateQuery` incluindo queries `pending` — é isso que permite streaming.

## 9. Não faça / faça

- **Não** chame `trpc.post.all.useQuery()` sem condição quando o input depende de um estado
  ainda indefinido; use `enabled` (opção do TanStack) ou `skipToken`.
- **Não** invalide tudo (`utils.invalidate()`) como reflexo; escolha o nível 2 ou 3.
- **Não** faça `setData` sem `cancel` antes — um refetch em voo sobrescreve o update otimista.
- **Não** use `useQuery` em Server Component; lá é `api.*.prefetch()` + `HydrateClient`.
- **Faça** `useSuspenseQuery` quando já houve prefetch e você quer render sem loading state.
- **Faça** `utils.<router>.<proc>.invalidate()` no `onSuccess` da mutation correspondente.
- **Faça** importar `api` de `~/trpc/react` em client e de `~/trpc/server` em server — nunca
  troque os dois.
