---
name: frontend-react-nextjs-trpc
description: Referência atual do tRPC v11 (server, client e React Query) obtida do Context7. Use ao criar routers, procedures, middlewares, chamadas tipadas no cliente ou invalidação de cache tRPC.
---

# tRPC v11 — referência de implementação

Fontes consultadas via CLI `ctx7` (Context7) em 2026-11-10, biblioteca `/trpc/trpc`.
Consultas cobriram: init/router/procedures, contexto, middlewares de auth, validators
Zod, `TRPCError` + `errorFormatter`, setup do client com links, `useQuery`/`useMutation`/
`useUtils`, `useSuspenseQuery`/`useInfiniteQuery`, `createCaller` + `HydrateClient`,
transformers/superjson, `splitLink`/subscriptions e `fetchRequestHandler`/caching.

Quando este guia não responder, consulte a fonte:

```bash
ctx7 docs /trpc/trpc "<pergunta específica>"
```

## Onde cada tipo de mudança entra (estrutura deste repo)

| Tarefa                                                     | Arquivo                             |
| ---------------------------------------------------------- | ----------------------------------- |
| Criar/editar procedure, schema Zod, regra de autorização   | `src/server/api/routers/<domínio>.ts` |
| Registrar um router novo                                   | `src/server/api/root.ts`            |
| Novo middleware, contexto, errorFormatter, tipo de procedure | `src/server/api/trpc.ts`            |
| Hook/provider do cliente (links, transformer, base URL)    | `src/trpc/react.tsx`                |
| Chamada server-side / prefetch / `HydrateClient`           | `src/trpc/server.ts`                |
| Estado global do React Query (staleTime, dehydrate)        | `src/trpc/query-client.ts`          |
| HTTP handler (`fetchRequestHandler`) e `responseMeta`      | `src/app/api/trpc/[trpc]/route.ts`  |

## Mapa de consulta

| Se você precisa de…                                        | Leia                         |
| ---------------------------------------------------------- | ---------------------------- |
| `initTRPC`, router, procedures, contexto, middleware, input/output, erros, `createCaller` | `references/server.md` |
| `createTRPCClient`, links, transformer, vanilla client, tipos inferidos, chamadas em Server Components | `references/client.md` |
| `createTRPCReact`, `useQuery`/`useMutation`/`useSuspenseQuery`/`useInfiniteQuery`, invalidação, updates otimistas, prefetch | `references/react-query.md` |

## Regras que este repo já segue (não invente outro padrão)

1. Existe **um único** `initTRPC` em `src/server/api/trpc.ts`. Nada de criar um `t` novo
   dentro de router ou de componente.
2. Só se exporta o que é usado: `createTRPCRouter` (= `t.router`), `createCallerFactory`,
   `publicProcedure` e `protectedProcedure`. O objeto `t` não é exportado.
3. `createTRPCContext` recebe `{ headers }` e é reutilizado pelo HTTP handler e pelo caller
   de RSC. Contexto novo (ex.: usuário, tenant) entra ali, não em cada procedure.
4. Autorização se faz por middleware em `trpc.ts` (`protectedProcedure` estreita
   `ctx.session.user` para não nulo). Router não repete checagem de sessão.
5. `src/server/api/root.ts` agrega os routers num namespace (`post: postRouter`) e exporta
   `AppRouter` + `createCaller`. Todo router novo precisa ser registrado lá.
6. Cliente React usa `createTRPCReact<AppRouter>()` em `src/trpc/react.tsx` e consome
   `api` de `~/trpc/react`; não crie um segundo client.
7. Server Components usam `api`/`HydrateClient` de `~/trpc/server` (que vem de
   `createHydrationHelpers` de `@trpc/react-query/rsc`) e nunca importam `~/trpc/react`.
8. No v11 o transformer é configurado **por link** (`httpBatchStreamLink({ transformer })`),
   não mais no `createClient`. O servidor mantém `transformer: superjson` no `initTRPC`.
9. Entrada de toda procedure é validada com Zod; `protectedProcedure` é obrigatório para
   qualquer leitura/escrita dependente de usuário.
10. Rode `npm run check` após qualquer mudança nesta camada.
