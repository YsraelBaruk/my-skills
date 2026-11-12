# tRPC v11 — servidor

> Fonte: `/trpc/trpc` via `ctx7 docs` (consultado em 2026-11-10).

Escopo: `initTRPC`, contexto, router, procedures, middleware, validação de entrada/saída,
erros e chamadas server-side (`createCaller`). No app, quase tudo isto vive em
`src/server/api/trpc.ts`, `src/server/api/root.ts` e `src/server/api/routers/`.

## 1. `initTRPC` — uma única vez por app

A doc é explícita: `initTRPC` deve ser usado **uma vez por aplicação**, exportando apenas o que se usa para forçar quais procedures base existem.

```ts
// src/server/api/trpc.ts (padrão atual do repo)
import { initTRPC } from "@trpc/server";
import superjson from "superjson";

const t = initTRPC.context<typeof createTRPCContext>().create({
  transformer: superjson, // @see trpc.io/docs/server/data-transformers
  errorFormatter(opts) {
    return opts.shape; // @see trpc.io/docs/server/error-formatting
  },
});

export const createTRPCRouter = t.router;
export const publicProcedure = t.procedure;
export const createCallerFactory = t.createCallerFactory;
```

Não faça / faça:

- Não exporte o objeto `t` inteiro (a doc cita o conflito com a convenção `t` de i18n).
  Exporte helpers com nome (`createTRPCRouter`, `publicProcedure`).
- Não chame `initTRPC.create()` num router só para ter `t.procedure`.

## 2. Contexto

O contexto dá acesso a banco, sessão e headers dentro das procedures. O criador aceita
`{ headers }` porque é reutilizado pelo handler HTTP e pelo caller de RSC.

```ts
export const createTRPCContext = async (opts: { headers: Headers }) => {
  const session = await auth();
  return { db, session, ...opts };
};

const t = initTRPC.context<Awaited<ReturnType<typeof createTRPCContext>>>().create();
```

A doc descreve esse reuso: “This context creator accepts `headers` so it can be reused in both the RSC server caller and the API route handler.”

Regra prática neste repo: precisa de algo novo (ex.: `tenantId`, feature flags)? Adicione em
`createTRPCContext`, nunca leia `headers()` direto dentro de uma procedure.

## 3. Router e procedures

`t.router({...})` cria o router; `t.procedure` é a base; `.query()` e `.mutation()` definem o
tipo da operação.

```ts
export const postRouter = createTRPCRouter({
  hello: publicProcedure
    .input(z.object({ text: z.string() }))
    .query(({ input }) => ({ greeting: `Hello ${input.text}` })),

  create: protectedProcedure
    .input(z.object({ name: z.string().min(1) }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.post.create({
        data: { name: input.name, createdBy: { connect: { id: ctx.session.user.id } } },
      });
    }),
});
```

Um router agrega outros routers como namespace. É o que o repo faz em `root.ts`:

```ts
export const appRouter = createTRPCRouter({
  post: postRouter,
});
export type AppRouter = typeof appRouter;
```

`mergeRouters` existe para juntar routers em um único namespace plano, mas **não** é o padrão
deste repo — aqui o namespace explícito (`post:`, `user:`) é preferível.

## 4. Input e output com Zod

`.input()` valida a entrada; se falhar, a procedure não executa e a chamada falha com
`BAD_REQUEST`, com o erro do validador como `cause`. `.output()` valida a saída e, quando
retorna um `z.object`, **chaves desconhecidas são removidas** da resposta.

```ts
export const appRouter = t.router({
  hello: publicProcedure
    .input(z.object({ name: z.string() }))
    .output(z.object({ greeting: z.string() }))
    .query(({ input }) => ({ greeting: `hello ${input.name}` })),
});
```

Fatos da doc sobre `output`: o client recebe o valor que o validador devolve (daí o strip de
chaves); se a validação falhar, o servidor responde `INTERNAL_SERVER_ERROR`; o validator é
opcional, porque a tipagem já é inferida do retorno — use quando a fonte não é confiável ou
para impedir vazamento de campos extras.

## 5. Middleware e autorização

Middleware recebe `{ ctx, path, type, rawInput, next }` e **precisa** retornar `opts.next()`.
O `ctx` de `next({ ctx })` é mesclado ao contexto existente — só chaves novas ou sobrescritas
precisam ser informadas.

```ts
const protectedProcedure = publicProcedure.use(async function isAuthed(opts) {
  const { ctx } = opts;
  if (!ctx.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return opts.next({
    ctx: {
      user: ctx.user, // ✅ agora não nulo para o resto da cadeia
    },
  });
});
```

Este é exatamente o padrão de `protectedProcedure` deste repo. A doc também mostra o uso
para adicionar contexto a partir de uma fonte externa:

```ts
const serverActionProcedure = t.procedure.use(async (opts) => {
  const user = await currentUser();
  return opts.next({ ctx: { user } });
});
```

Não duplique checagem de sessão dentro da procedure; use `protectedProcedure` e deixe o tipo
`ctx.session.user` já estreitado. Se precisar de outro nível (admin, dono do recurso), crie
outro `...Procedure` em `trpc.ts`.

Para exportar métricas, use `t.middleware` (como o `timingMiddleware` do repo), sempre
aguardando `next()` antes de medir o fim.

## 6. Erros

`TRPCError` é a forma suportada de falhar com semântica HTTP correta. Qualquer outro erro
lançado vira `INTERNAL_SERVER_ERROR` com o erro original em `cause`.

```ts
throw new TRPCError({
  code: "INTERNAL_SERVER_ERROR",
  message: "An unexpected error occurred, please try again later.",
  cause: theError, // preserva stack trace
});
```

Use `NOT_FOUND`, `FORBIDDEN`, `BAD_REQUEST` etc. para o status correto no client.

Aviso da doc: a `message` de um erro que **não** é `TRPCError` também chega ao client, mesmo
em produção. Se a mensagem pode conter detalhe interno, esconda no formatter:

```ts
const t = initTRPC.create({
  errorFormatter(opts) {
    const { shape, error } = opts;
    if (error.code === "INTERNAL_SERVER_ERROR" && process.env.NODE_ENV === "production") {
      return { ...shape, message: "Internal server error" };
    }
    return shape;
  },
});
```

O repo usa o formatter para expor `zodError` tipado ao client:

```ts
errorFormatter({ shape, error }) {
  return {
    ...shape,
    data: {
      ...shape.data,
      zodError: error.cause instanceof ZodError ? error.cause.flatten() : null,
    },
  };
}
```

Isso permite ler `mutation.error.data.zodError` no componente com tipo inferido até o client.

## 7. Chamadas server-side com `createCaller`

`createCallerFactory` gera uma função que monta um caller a partir do contexto. É o caminho
para Server Components e testes, **sem** passar por HTTP.

```ts
const createCaller = createCallerFactory(appRouter);
const caller = createCaller({ foo: "bar" });

const addedPost = await caller.post.add({ title: "server-side call" });
const postList = await caller.post.list();
```

Neste repo, o factory vem de `t.createCallerFactory` e é exportado em `root.ts` como
`export const createCaller = createCallerFactory(appRouter);`.

Regra da doc: **não** chame `createCaller` dentro de outra procedure — extraia a lógica
compartilhada para uma função de domínio.

## 8. Adapter HTTP (App Router) e caching

O endpoint é um route handler com `fetchRequestHandler`, exportado como `GET` e `POST`:

```ts
const handler = (req: Request) =>
  fetchRequestHandler({
    endpoint: "/api/trpc",
    req,
    router: appRouter,
    createContext: () => createTRPCContext({ headers: req.headers }),
  });

export { handler as GET, handler as POST };
```

Este repo usa `NextRequest`, `env` e `onError` em dev — mesma estrutura, apenas tipada, e sem `responseMeta`.

Sobre cache no App Router: o Next envia o `Response` do handler como está. Configurações de
segmento (`dynamic`, `revalidate`) **não** mudam o header; use `responseMeta`:

```ts
responseMeta(opts) {
  const { info, errors, type } = opts;
  const allPublic = info?.calls.every((call) => call.path.startsWith("public."));
  if (allPublic && errors.length === 0 && type === "query") {
    return {
      headers: new Headers([["cache-control", "s-maxage=60, stale-while-revalidate=86400"]]),
    };
  }
  return {};
}
```

Só aplique cache quando todas as rotas chamadas são públicas e sem sessão; caso contrário
dados de usuário podem ser cacheados.

Checklist antes de fechar um router: entrada validada com Zod; `protectedProcedure` para
qualquer operação dependente de usuário; dados sempre via `ctx.db`; falhas esperadas como
`TRPCError` com o código correto; router registrado em `src/server/api/root.ts`;
`npm run check` verde.
