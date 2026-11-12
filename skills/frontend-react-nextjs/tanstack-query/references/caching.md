# Cache e ciclo de vida

> Fonte: `/tanstack/query` via `ctx7 docs` (consultado em 2026-11-10).

## staleTime x gcTime

São dois relógios independentes e é o erro de leitura mais comum:

- `staleTime` — por quanto tempo os dados são considerados frescos. Enquanto frescos, não
  há refetch automático (mount, focus, reconnect). Default `0`, então tudo é stale assim
  que chega. Aceita `Infinity`/`'static'` para nunca ficar stale.
- `gcTime` — por quanto tempo uma entrada **sem observers** permanece na memória antes de
  ser coletada. Default `5 * 60 * 1000` (5 min), `Infinity` no servidor. Não tem efeito
  enquanto a query está em uso.

Ou seja: `gcTime` não é "tempo de cache" no sentido de validade; é tempo de retenção de
lixo. O nome antigo (`cacheTime`) foi renomeado justamente por isso.

```ts
export interface QueryOptions<...> {
  /**
   * Tempo em ms que dados não usados/inativos permanecem na memória.
   * Default `5 * 60 * 1000`, ou `Infinity` durante SSR.
   */
  gcTime?: number;
}
```

```tsx
const MINUTE = 60 * 1000;

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 10 * MINUTE, // v5 (era cacheTime na v4)
      staleTime: 60 * 1000,
    },
  },
});
```

Regra prática: dado que muda pouco → `staleTime` alto (minutos); dado que precisa sobreviver
à navegação → `gcTime` alto. Não combine `staleTime: 0` com `refetchOnWindowFocus` e depois
reclame de excesso de requests.

## Quando o refetch acontece

O observer refaz o fetch quando a query está stale e:

| Gatilho | Default | Como desativar |
| --- | --- | --- |
| mount do componente | `true` (só se stale) | `refetchOnMount: false` |
| foco da janela | `true` (só se stale) | `refetchOnWindowFocus: false` |
| reconexão de rede | efetivo `true` (salvo `networkMode: 'always'`) | `refetchOnReconnect: false` |
| intervalo | `false` | `refetchInterval: number \| (query) => number \| false` |
| invalidação manual | — | `invalidateQueries` ignora `staleTime` e sempre refetch para observers ativos |

`"always"` força o refetch mesmo com dados frescos; a exceção é `staleTime: 'static'`.
Todos os três aceitam também função `(query) => boolean | "always"`.

`refetchIntervalInBackground: true` mantém o intervalo rodando com a aba em segundo plano
(default `false`).

## Deduplicação

Duas camadas:

1. **Mesma queryKey = mesma `Query`.** `QueryCache.build()` procura por hash antes de
   criar; prefetch e `useQuery` com a mesma chave compartilham instância,
   `dataUpdatedAt` e `gcTime`.
2. **Fetch em voo é compartilhado.** Se já existe fetch em andamento, `Query.fetch`
   devolve a promise do retryer atual em vez de iniciar outro request; retries cancelados
   por unmount são retomados com `continueRetry()`. É o que evita request duplicado em
   StrictMode (mount → unmount → mount).

```ts
// se um fetch já está em voo, retorna a promise existente
if (this.state.fetchStatus !== "idle" && this.#retryer?.status() !== "rejected") {
  if (this.state.data !== undefined && fetchOptions?.cancelRefetch) {
    this.cancel({ silent: true });
  } else if (this.#retryer) {
    this.#retryer.continueRetry();
    return this.#retryer.promise;
  }
}
```

Consequência: chamar `useQuery` com a mesma chave em 5 componentes gera um único request.
Chaves diferentes com o mesmo `queryFn` geram requests separados.

## Garbage collection

Sem LRU, sem limite de quantidade, sem limite de memória. O cache é limitado apenas por
tempo: ao remover o último observer, a query agenda o GC com `gcTime`; se um observer
assinar antes do timeout, o GC é cancelado. Quando `gcTime` aparece em vários pontos, vale
o maior (`Math.max`).

```ts
protected scheduleGc(): void {
  this.clearGcTimeout();
  if (isValidTimeout(this.gcTime)) {
    this.#gcTimeout = timeoutManager.setTimeout(() => {
      this.optionalRemove();
    }, this.gcTime);
  }
}

protected updateGcTime(newGcTime: number | undefined): void {
  this.gcTime = Math.max(
    this.gcTime || 0,
    newGcTime ?? (isServerEnvironment() ? Infinity : 5 * 60 * 1000),
  );
}
```

`Infinity` desativa o GC (dados de referência, SSR). O timeout máximo prático é ~24 dias
(inteiro de 32 bits do `setTimeout`).

## Prefetch

- `prefetchQuery` inicia o fetch sem lançar e sem rejeitar (erros engolidos) — para
  disparar "fire and forget" antes de navegar/renderizar.
- `queryClient.query({ queryKey, queryFn })` aparece na documentação atual como substituto
  de `fetchQuery`/`prefetchQuery`/`ensureQueryData` (depreciação marcada para v6). Na
  linha 5.69 use `prefetchQuery`/`fetchQuery`; se o tipo existir no seu build,
  `queryClient.query` é preferível.

```ts
// v5 estável
await queryClient.prefetchQuery({ queryKey: ["posts"], queryFn: getPosts });
// fire and forget com erro engolido
void queryClient.prefetchQuery({ queryKey: ["posts"], queryFn: getPosts });
```

Importante: o `staleTime` do prefetch e o do `useQuery` são avaliados de forma
independente. `query()` decide se busca com base no `staleTime` da chamada; o observer
depois compara seu próprio `staleTime` com o mesmo `dataUpdatedAt`. Um prefetch com
`staleTime: 0` não torna a query fresca para um `useQuery` com `staleTime` alto, nem o
contrário.

```ts
// dentro de queryClient.query / prefetchQuery
const isStale = query.isStaleByTime(resolveQueryValue(defaultedOptions.staleTime, query));
const queryData = isStale ? await query.fetch(defaultedOptions) : query.state.data;
```

No tRPC deste repo isso é `api.post.getLatest.prefetch()` (ou `void` quando não se quer
aguardar).

## SSR e hidratação

Fluxo: criar `QueryClient` por request → popular no servidor → `dehydrate` → entregar ao
cliente → `HydrationBoundary` hidrata o cache.

```tsx
// Server Component
import { dehydrate, HydrationBoundary, QueryClient } from "@tanstack/react-query";

export default async function PostsPage() {
  const queryClient = new QueryClient();

  // docs atuais usam queryClient.query({...}); em v5 estável: prefetchQuery/fetchQuery
  await queryClient.prefetchQuery({ queryKey: ["posts"], queryFn: getPosts });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Posts />
    </HydrationBoundary>
  );
}
```

```tsx
// Client Component
"use client";

function Posts() {
  // já hidratado: renderiza com dados no primeiro paint
  const { data } = useQuery({ queryKey: ["posts"], queryFn: getPosts });

  // esta não foi prefetchada: começa a buscar no cliente (padrão válido)
  const { data: comments } = useQuery({
    queryKey: ["posts-comments"],
    queryFn: getComments,
  });
}
```

Regras de `HydrationBoundary`:

- Só queries são desidratadas; mutações não.
- Se o cache do cliente já tem dados, a mesclagem é por timestamp de atualização — só
  entra o que for mais novo.
- Queries novas são hidratadas durante o render; queries já existentes recebem dados mais
  novos em um efeito após o commit, então os children podem renderizar brevemente antes.

### Por que `staleTime > 0` no SSR

Sem `staleTime`, o `dataUpdatedAt` vem do servidor e o cliente considera stale no primeiro
render, disparando refetch imediato (request duplicado logo após o load). É exatamente o
motivo do `staleTime: 30 * 1000` em `src/trpc/query-client.ts`.

### useSuspenseQuery vs useQuery no SSR

Se você prefetchou e `await`ou, os dados estão no cache: `useQuery` renderiza direto e
`useSuspenseQuery` também não suspende. Se o prefetch for apenas disparado (não aguardado)
e a query for lida com `useQuery`, o servidor pode renderizar o estado de loading enquanto
o browser já tem dados — o React reporta mismatch de hidratação. Nesse caso use
`useSuspenseQuery` (ou aguarde o prefetch).

### Streaming no App Router

`ReactQueryStreamedHydration` (`@tanstack/react-query-next-experimental`) envia para o
cliente o que foi buscado com suspense no servidor, mantendo singleton no browser e novo
client por request no servidor:

```tsx
"use client";

import { ReactQueryStreamedHydration } from "@tanstack/react-query-next-experimental";

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={getQueryClient()}>
      <ReactQueryStreamedHydration>{children}</ReactQueryStreamedHydration>
    </QueryClientProvider>
  );
}

// getQueryClient(): novo QueryClient por request no servidor, singleton no browser
```

Este repositório **não** usa esse provider: usa `createHydrationHelpers` do tRPC
(`HydrateClient` em `src/trpc/server.ts`), que segue o mesmo modelo de dehydrate/hydrate
com SuperJSON configurado em `src/trpc/query-client.ts`.

## Checklist rápido

- `staleTime` é "quanto tempo evito refetch"; `gcTime` é "quanto tempo guardo depois de
  ficar sem observers".
- Para forçar atualização ignore o `staleTime`: `invalidateQueries`.
- Antes de update otimista, `cancelQueries` na mesma chave.
- Prefetch e `useQuery` compartilham a entrada se a chave for idêntica (incluindo input).
- Em SSR, sempre `staleTime > 0` e `QueryClient` novo por request.
