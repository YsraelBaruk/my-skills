# Queries

> Fonte: `/tanstack/query` via `ctx7 docs` (consultado em 2026-11-10).

Docs do Context7 vêm da branch `main`; divergências da linha 5.69 do `package.json` estão sinalizadas.

## QueryClient e QueryClientProvider

O `QueryClient` guarda o cache. Em SSR precisa ser criado por request; no browser, ser
singleton. Criar no topo do módulo compartilha o cache entre requests e vaza dados entre
usuários.

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import * as React from "react";

export default function App({ children }: { children: React.ReactNode }) {
  const [queryClient] = React.useState(
    () =>
      new QueryClient({
        defaultOptions: {
          // Com SSR, staleTime acima de 0 evita refetch imediato no cliente
          queries: { staleTime: 60 * 1000, gcTime: 10 * 60 * 1000, retry: 3 },
        },
      }),
  );
  return <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>;
}
```

Defaults relevantes (`QueryObserverOptions` / `QueryOptions`):

| Opção | Default | Observação |
| --- | --- | --- |
| `staleTime` | `0` | `number`, `Infinity`, `'static'` ou `(query) => number` |
| `gcTime` | `5 * 60 * 1000` | `Infinity` no servidor; `Infinity` desativa GC |
| `retry` | `3` cliente / `0` servidor | `false`, `true`, número ou `(failureCount, error) => boolean` |
| `retryDelay` | backoff exponencial, teto 30s | `(attemptIndex) => number` |
| `refetchOnMount` | `true` | `"always"` força, exceto com `staleTime: 'static'` |
| `refetchOnWindowFocus` | `true` | `"always"` força; aceita função |

Em testes, `defaultOptions.queries.retry = false` evita timeout: o `retry` do `QueryClient`
só vale quando o `useQuery` não define o seu.

## useQuery: assinatura única em objeto

A v5 removeu as sobrecargas posicionais:

```tsx
useQuery(key, fn, options); // v4, removido
useQuery({ queryKey, queryFn, ...options }); // v5
```

```tsx
import { useQuery } from "@tanstack/react-query";

function Posts() {
  const { status, data, error, isFetching } = useQuery({ queryKey: ["posts"], queryFn: fetchPosts });

  if (status === "pending") return "Loading...";
  if (status === "error") return <span>Error: {error.message}</span>;
  return (
    <ul>
      {data.map((post) => <li key={post.id}>{post.title}</li>)}
      {isFetching ? <div>Background Updating...</div> : null}
    </ul>
  );
}
```

### Flags de estado (v5)

`isPending` substitui o antigo `isLoading`. Na v5, `isLoading` significa
`isPending && isFetching` (primeira carga com fetch em andamento), não "carregando" genérico.

```tsx
function Todos() {
  const { isPending, isError, data, error } = useQuery({
    queryKey: ["todos"],
    queryFn: fetchTodoList,
  });

  if (isPending) return <span>Loading...</span>;
  if (isError) return <span>Error: {error.message}</span>;
  // aqui `isSuccess === true` e `data` está estreitado pelo union discriminado
  return <ul>{data.map((todo) => <li key={todo.id}>{todo.title}</li>)}</ul>;
}
```

| Flag | Quando é verdadeira |
| --- | --- |
| `isPending` | `status === "pending"` (sem dados); `isLoading` = `isPending && isFetching` |
| `isSuccess` | `status === "success"`; estreita `data` |
| `isError` | `status === "error"`; `error` tipado |
| `isFetching` | qualquer fetch em andamento, inclusive background |
| `isPlaceholderData` | renderizando `placeholderData` |
| `isRefetching` | `isFetching && !isPending` |

## queryKey

Array no topo, determinístico, contendo todas as dependências do `queryFn`. A ordem de
chaves de objeto é ignorada no hash; a ordem dos arrays importa.

```ts
useQuery({ queryKey: ["todos"], ... });
useQuery({ queryKey: ["todo", 5, { preview: true }], ... });
useQuery({ queryKey: ["todos", { type: "done" }], ... });
```

`hashKey(['todos', { page: 1, filter: 'done' }])` → `'["todos",{"filter":"done","page":1}"]`.
Logo `['todos', { status, page }]` e `['todos', { page, status }]` são a mesma chave, mas
`['todos', status, page]` e `['todos', page, status]` não são. Chaves aninhadas permitem
invalidação parcial: `['todos']` atinge `['todos', 5]` e `['todos', { status }]`.

```tsx
function Todos({ status, page }: { status: string; page: number }) {
  return useQuery({ queryKey: ["todos", { status, page }], queryFn: fetchTodoList });
}
```

A `queryFn` pode ler as mesmas variáveis de `queryKey` via `QueryFunctionContext`
(`function fetchTodoList({ queryKey }) { const [_key, { status, page }] = queryKey; ... }`).

## enabled (queries dependentes)

`enabled: false` desliga a query: sem request e sem estado stale. A variável deve estar na
`queryKey` para que a troca de input dispare a busca.

```tsx
function Post({ postId }: { postId: number | undefined }) {
  const { data, isPending, isError, error } = useQuery({
    queryKey: ["post", postId],
    queryFn: () => fetchPost(postId!),
    enabled: postId != null,
  });

  if (postId == null) return "Select a post";
  if (isPending) return "Loading...";
  if (isError) return <span>Error: {error.message}</span>;
  return <h1>{data?.title}</h1>;
}
```

Forma curta, com o filtro na `queryKey`:
`useQuery({ queryKey: ["todos", filter], queryFn: () => fetchTodos(filter), enabled: !!filter })`.

## select e placeholderData

`select` transforma o resultado antes de expor `data`, sem alterar o cache; só roda quando
`data` ou a referência de `select` mudam — memoize. `placeholderData` renderiza conteúdo
inicial sem gravar no cache: a query começa em `success` com `isPlaceholderData: true`.

```tsx
const { data } = useQuery({
  queryKey: ["todos"],
  queryFn: fetchTodos,
  select: (todos) => todos.filter((t) => !t.done),
});

// mantém o resultado anterior durante a troca de chave
useQuery({
  queryKey: ["todos", id],
  queryFn: () => fetch(`/todos/${id}`),
  placeholderData: (previousData) => previousData,
});
```

Migração v4 → v5: `keepPreviousData: true` virou `placeholderData: keepPreviousData`
(helper importado do pacote, `import { keepPreviousData } from "@tanstack/react-query"`) e
`isPreviousData` virou `isPlaceholderData`.

## useQueries (quantidade variável)

Sem `combine` retorna um array na ordem da entrada; com `combine`, um valor único
estruturalmente compartilhado. `combine` só reexecuta se mudar de referência ou se algum
resultado mudar — inline roda a cada render, então use `useCallback`.

```tsx
import { useQueries } from "@tanstack/react-query";

const { data, isPending, isError } = useQueries({
  queries: ids.map((id) => ({ queryKey: ["post", id], queryFn: () => fetchPost(id) })),
  combine: (queries) => ({
    data: queries.map((q) => q.data),
    isPending: queries.some((q) => q.isPending),
    isError: queries.some((q) => q.isError),
  }),
});

if (isPending) return "Loading...";
if (isError) return "Error loading posts";
return <ul>{data.map((post) => <li key={post?.id}>{post?.title}</li>)}</ul>;
```

Cuidados: chaves duplicadas compartilham dados; `placeholderData` não recebe dados de
render anterior; `select` inline não infere o tipo do `queryFn` irmão.

## useInfiniteQuery

Mesmas opções de `useQuery` mais `initialPageParam`, `getNextPageParam`,
`getPreviousPageParam` e `maxPages`. Retorna ainda `fetchNextPage`, `fetchPreviousPage`,
`hasNextPage`, `hasPreviousPage` e `isFetchingNextPage`; `data` tem `pages` e `pageParams`.

```tsx
const { data, fetchNextPage, hasNextPage, isFetching, isFetchingNextPage } =
  useInfiniteQuery({
    queryKey: ["projects"],
    queryFn: ({ pageParam }) => fetchProjects(pageParam),
    initialPageParam: 0,
    getNextPageParam: (lastPage) => lastPage.nextId,
  });
```

`fetchNextPage` e afins competem com o refetch automático: chame em reação a ação do
usuário e condicione com `hasNextPage && !isFetching`.

## useSuspenseQuery

`data` é garantida, sem checagem de `isPending`. As opções são as de `useQuery` menos
`enabled`, `throwOnError` e `placeholderData` (o hook não renderiza estado desabilitado,
erro local nem placeholder), com `queryFn` obrigatório. Erro sem cache é lançado ao error
boundary, então envolva com `<ErrorBoundary>` + `QueryErrorResetBoundary` (o `reset` vai em
`onReset`) para permitir nova tentativa.

```tsx
import { useSuspenseQuery } from "@tanstack/react-query";

function Posts() {
  // `data` garantida; `isFetching` cobre o refetch em background
  const { data, isFetching } = useSuspenseQuery({ queryKey: ["posts"], queryFn: fetchPosts });

  if (isFetching) return <p>Refreshing...</p>;
  return <ul>{data.map((post) => <li key={post.id}>{post.title}</li>)}</ul>;
}
```

`useSuspenseQueries` paraleliza queries de suspense, evitando o waterfall de
`useSuspenseQuery` consecutivos. Cada `query` não aceita `enabled`, `throwOnError` nem
`placeholderData`, e `data` é garantida. O componente só remonta depois que todas terminam:
use `staleTime` alto o bastante para não refazer tudo no remount.

```tsx
const [users, teams, projects] = useSuspenseQueries({
  queries: [
    { queryKey: ["users"], queryFn: fetchUsers },
    { queryKey: ["teams"], queryFn: fetchTeams },
    { queryKey: ["projects"], queryFn: fetchProjects },
  ],
});
```
