# tRPC v11 — cliente

> Fonte: `/trpc/trpc` via `ctx7 docs` (consultado em 2026-11-10).

Escopo: `createTRPCClient`, links, transformer no v11, vanilla client, tipos inferidos e
chamadas em Server Components. No app, o cliente React está em `src/trpc/react.tsx` e o
cliente de servidor em `src/trpc/server.ts`.

## 1. `createTRPCClient` + `httpBatchLink`

O client recebe `AppRouter` como **type-only import**: isso é apagado no build e garante
tipagem ponta a ponta sem vazar implementação do servidor.

```ts
import { createTRPCClient, httpBatchLink } from "@trpc/client";
import type { AppRouter } from "../server/appRouter";
//     👆 type-only: removido em build time

const trpc = createTRPCClient<AppRouter>({
  links: [httpBatchLink({ url: "http://localhost:3000" })],
});

const bilbo = await trpc.getUser.query("id_bilbo");
// => { id: 'id_bilbo', name: 'Bilbo' }
```

## 2. Transformer é por link (mudança do v11)

No v11 o transformer saiu do `createClient` e passou para os links. Isso é breaking change
em relação ao v10.

```ts
httpBatchLink({
  url: "/api/trpc",
  transformer: superjson, // <-- obrigatório no v11 se o server usa transformer
});
```

Fatos da doc:

- “Every terminating link needs the transformer”, incluindo os dois ramos de um `splitLink`.
- Com `wsLink`, o transformer vai em `wsLink()`, não em `createWSClient()`.
- O servidor continua com `transformer: superjson` no `initTRPC`.

Este repo usa `httpBatchStreamLink` (equivalente de batch com streaming) já com
`transformer: SuperJSON`. Ao adicionar um segundo link terminal, repita o transformer nele.

## 3. `splitLink` e subscriptions

Quando há mais de um tipo de operação, o `splitLink` roteia por `op.type`. Cada ramo terminal
precisa do transformer.

```ts
import {
  createTRPCClient,
  httpBatchLink,
  httpSubscriptionLink,
  splitLink,
} from "@trpc/client";

export const client = createTRPCClient<AppRouter>({
  links: [
    splitLink({
      condition: (op) => op.type === "subscription",
      true: httpSubscriptionLink({ url: "http://localhost:3000", transformer: superjson }),
      false: httpBatchLink({ url: "http://localhost:3000", transformer: superjson }),
    }),
  ],
});
```

Este repo não usa subscriptions hoje; se introduzir, este é o formato e o transformer precisa
estar nos dois ramos. Confirme detalhes de `httpSubscriptionLink` com
`ctx7 docs /trpc/trpc "httpSubscriptionLink"` antes de implementar.

## 4. Tipos inferidos do router

```ts
import type { inferRouterInputs, inferRouterOutputs } from "@trpc/server";
import type { AppRouter } from "./server";

type RouterInput = inferRouterInputs<AppRouter>;
type RouterOutput = inferRouterOutputs<AppRouter>;

type PostCreateInput = RouterInput["post"]["create"];
type PostCreateOutput = RouterOutput["post"]["create"];
```

O repo já exporta atalhos em `src/trpc/react.tsx`:

```ts
export type RouterInputs = inferRouterInputs<AppRouter>;
export type RouterOutputs = inferRouterOutputs<AppRouter>;
```

Prefira esses atalhos a redeclarar `inferRouterOutputs` em cada componente.

## 5. Vanilla client — quando usar

Use `createTRPCClient` diretamente quando não houver React no caminho: scripts, workers,
jobs, testes de integração ou uma segunda aplicação TypeScript consumindo este backend.
Chamadas são funções locais tipadas (`client.post.create.mutate(...)`,
`client.post.all.query()`).

Não faça: usar o vanilla client dentro de componente React. Nesse contexto use os hooks de
`src/trpc/react.tsx` (`api.*.useQuery`), que integram cache e invalidação.

## 6. Server Components — `createHydrationHelpers`

Para RSC, a doc mostra `@trpc/react-query/rsc`, que é exatamente o caminho deste repo:

```tsx
import "server-only"; // <-- garante que este arquivo não vai para o client

import { createHydrationHelpers } from "@trpc/react-query/rsc";
import { cache } from "react";
import { createCallerFactory, createTRPCContext } from "./init";
import { makeQueryClient } from "./query-client";
import { appRouter } from "./routers/_app";

// getter estável: o MESMO client durante a request
export const getQueryClient = cache(makeQueryClient);
const caller = createCallerFactory(appRouter)(createTRPCContext);

export const { trpc, HydrateClient } = createHydrationHelpers<typeof appRouter>(
  caller,
  getQueryClient,
);
```

Equivalente neste repo (`src/trpc/server.ts`), com o contexto vindo de `next/headers` e
`cache`:

```ts
const createContext = cache(async () => {
  const heads = new Headers(await headers());
  heads.set("x-trpc-source", "rsc");
  return createTRPCContext({ headers: heads });
});

const getQueryClient = cache(createQueryClient);
const caller = createCaller(createContext);

export const { trpc: api, HydrateClient } = createHydrationHelpers<AppRouter>(
  caller,
  getQueryClient,
);
```

Regras que a doc reforça:

1. `import "server-only"` no topo — impede import acidental em Client Component.
2. `cache(...)` no contexto e no QueryClient — sem isso você cria client novo a cada render e
   a hidratação quebra.
3. Um único caller por request; ele chama as procedures sem HTTP.

## 7. Alternativa nova: `@trpc/tanstack-react-query`

A doc v11 também apresenta o pacote `@trpc/tanstack-react-query` com
`createTRPCContext`/`useTRPC` e `createTRPCOptionsProxy` no servidor. É um modelo diferente
do `createTRPCReact` usado aqui. **Não migre por conta própria**: este repo está no padrão
T3 com `createTRPCReact`. Se a migração for pedida, confirme a API com
`ctx7 docs /trpc/trpc "createTRPCOptionsProxy"`.

## 8. Erros no cliente

Formato do erro que chega ao client (quando o servidor lança `TRPCError`):

```json
{
  "id": null,
  "error": {
    "message": "An unexpected error occurred, please try again later.",
    "code": -32603,
    "data": {
      "code": "INTERNAL_SERVER_ERROR",
      "httpStatus": 500,
      "stack": "...",
      "path": "hello"
    }
  }
}
```

Com o `errorFormatter` deste repo, `error.data.zodError` existe quando a falha foi de
validação, e o tipo é inferido até o componente.

## 9. Não faça / faça

- **Não** instancie `createTRPCClient` dentro do corpo de um componente (recria a cada
  render). Use `useState(() => ...)` como em `TRPCReactProvider`.
- **Não** importe `~/trpc/react` em Server Component, nem `~/trpc/server` em Client
  Component — são camadas separadas por `server-only`.
- **Não** coloque `transformer` no `createTRPCClient` no v11; ele pertence ao link.
- **Faça** um único `httpBatchStreamLink` com URL derivada de `getBaseUrl()`; adicione
  headers de origem (`x-trpc-source`) para diagnóstico.
- **Faça** importar `AppRouter` sempre como `import { type AppRouter }`.
