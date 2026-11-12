# TypeScript — inferência, input/output e integração

> Fonte: `/colinhacks/zod/v4.6.7` via `ctx7 docs` (consultado em 2026-11-10).

## `z.infer`: schema como fonte do tipo

`z.infer<typeof Schema>` extrai o tipo de **saída** do schema. É o padrão para não manter
uma interface escrita à mão ao lado do schema.

```ts
const Player = z.object({
  username: z.string(),
  xp: z.number(),
});

type Player = z.infer<typeof Player>;
// { username: string; xp: number }

const player: Player = { username: "billie", xp: 100 };
```

O tipo inferido acompanha as validações e wrappers: `.optional()` produz `campo?: tipo`,
`.nullable()` produz `tipo | null`, enums produzem a união de literais e arrays produzem
`tipo[]`.

```ts
const FishEnum = z.enum(["Salmon", "Tuna", "Trout"]);
type Fish = z.infer<typeof FishEnum>; // "Salmon" | "Tuna" | "Trout"

const MyResult = z.discriminatedUnion("status", [
  z.object({ status: z.literal("success"), data: z.string() }),
  z.object({ status: z.literal("failed"), error: z.string() }),
]);
type Result = z.infer<typeof MyResult>;
```

## `z.input` e `z.output`

Quando entrada e saída divergem, os dois tipos são extraídos separadamente:

```ts
const mySchema = z.string().transform((val) => val.length);

type MySchemaIn = z.input<typeof mySchema>;   // string
type MySchemaOut = z.output<typeof mySchema>; // number
// z.output<typeof T> é equivalente a z.infer<typeof T>
```

Casos típicos de divergência: `.transform()`, `.default()`, `z.coerce`, `z.preprocess` e
codecs. Use `z.input` para tipar o que o chamador entrega e `z.output` para o que o código
recebe depois do parse.

```ts
const schema = z.object({
  name: z.string(),
  active: z.stringbool().default(false),
});

type SchemaIn = z.input<typeof schema>;
// { name: string; active?: string | boolean | undefined }

type SchemaOut = z.output<typeof schema>;
// { name: string; active: boolean }
```

## `z.input()` e `z.output()` em runtime

Além dos tipos, a v4 expõe funções de runtime que substituem cada pipe pelo lado de entrada
ou de saída. Isso alcança codecs aninhados em objetos, records e maps, onde `.in`/`.out` não
chegam.

```ts
const isoDate = z.codec(z.iso.datetime(), z.date(), {
  decode: (s) => new Date(s),
  encode: (d) => d.toISOString(),
});

const Event = z.object({ name: z.string(), at: isoDate });

z.input(Event).parse({ name: "launch", at: "2024-01-01T00:00:00Z" }); // ok
z.output(Event).parse({ name: "launch", at: new Date() });            // ok
```

Em schemas sem codecs/pipes, as funções são no-op. Em transform de uma via, `z.output()`
devolve a transformação (que não valida nada); em `z.preprocess()`, `z.input()` devolve o
schema que o preprocessador alimenta.

## Schema como contrato de função

Valide na fronteira (request, env, formulário) e deixe o restante do código trabalhar com o
tipo de saída. O schema vira a única definição do contrato.

```ts
const createPostSchema = z.object({
  name: z.string().min(1).max(120),
});

type CreatePostInput = z.input<typeof createPostSchema>;
type CreatePostData = z.output<typeof createPostSchema>;

function createPost(input: CreatePostData) {
  // input.name já é string validada
}

const parsed = createPostSchema.parse(payload);
createPost(parsed);
```

Prefira `z.input` na assinatura de quem recebe dado não confiável e `z.output` na de quem
consome o dado já parseado. Se o schema só tem validação (sem transform/default), os dois
tipos coincidem.

## Integração com tRPC neste repo

O `.input(schema)` faz o tRPC inferir o tipo de `input` a partir do schema Zod. Não declare
uma interface paralela nem faça cast no handler.

```ts
import { z } from "zod";

import {
  createTRPCRouter,
  protectedProcedure,
  publicProcedure,
} from "~/server/api/trpc";

export const postRouter = createTRPCRouter({
  hello: publicProcedure
    .input(z.object({ text: z.string() }))
    .query(({ input }) => {
      return { greeting: `Hello ${input.text}` };
    }),

  create: protectedProcedure
    .input(z.object({ name: z.string().min(1) }))
    .mutation(async ({ ctx, input }) => {
      return ctx.db.post.create({
        data: {
          name: input.name,
          createdBy: { connect: { id: ctx.session.user.id } },
        },
      });
    }),
});
```

Pontos que o schema não resolve sozinho:

- Autorização continua sendo responsabilidade de `protectedProcedure`; validação de input
  não substitui checagem de sessão.
- O retorno da procedure é inferido pelo TypeScript, mas não é validado em runtime por um
  `.output(...)` neste repo. Se precisar garantir o formato de saída, adicione um schema de
  output explicitamente.
- Erros de validação chegam ao cliente pelo `errorFormatter` de `src/server/api/trpc.ts`.
  A doc v4 formata com `z.flattenError(error.cause)`; o arquivo atual usa o estilo v3
  `error.cause.flatten()`. Alinhe ao Zod instalado antes de mudar.

## Padrões de reuso

### Derivar create/update do mesmo schema

```ts
const postSchema = z.object({
  title: z.string().min(1),
  body: z.string(),
  published: z.boolean().default(false),
});

const createPostSchema = postSchema;
const updatePostSchema = postSchema.partial().extend({
  id: z.uuid(),
});

type CreatePost = z.output<typeof createPostSchema>;
type UpdatePost = z.output<typeof updatePostSchema>;
```

### Separar schema de entrada e DTO de saída

```ts
const userInputSchema = z.object({
  name: z.string(),
  email: z.email(),
});

type UserInput = z.output<typeof userInputSchema>;

type UserDto = {
  id: string;
  name: UserInput["name"];
  email: UserInput["email"];
};
```

### Exportar schema e tipo juntos

```ts
export const signupSchema = z.object({
  email: z.email(),
  password: z.string().min(8),
});

export type SignupInput = z.output<typeof signupSchema>;
```

Quem importa `signupSchema` para validar e `SignupInput` para tipar mantém as duas pontas
amarradas. Evite redefinir o tipo manualmente: a divergência entre schema e interface é
silenciosa em runtime.

## Aliases de tipo depreciados na v4

A camada de compatibilidade da v4 marca:

```ts
/** @deprecated Use `z.output<T>` instead. */
type TypeOf<T> = z.output<T>;

/** @deprecated Use `z.output<T>` instead. */
type Infer<T> = z.output<T>;
```

Também foram substituídos `ZodTypeAny`/`ZodSchema`/`Schema` por `z.ZodType` sem genéricos.
`ZodRawShape` continua disponível como alias de `core.$ZodShape` para compatibilidade.
Em código novo, use `z.infer`, `z.input` e `z.output`; não introduza os aliases antigos.
