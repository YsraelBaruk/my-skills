# Validação, transformação e erros

> Fonte: `/colinhacks/zod/v4.6.7` via `ctx7 docs` (consultado em 2026-11-10).

## `parse` e `safeParse`

`schema.parse(data)` devolve o valor parseado ou lança `ZodError`. `safeParse` não lança:
retorna uma união discriminada com `data` ou `error`.

```ts
const Player = z.object({
  username: z.string(),
  xp: z.number(),
});

// lança em caso de erro
const player = Player.parse({ username: "billie", xp: 100 });

// não lança; inspecione result.success
const result = Player.safeParse({ username: 42, xp: "100" });
if (!result.success) result.error; // ZodError
else result.data;                  // { username: string; xp: number }
```

O `ZodError` da v4 expõe `issues`, não `errors`. Cada issue traz `code`, `path`, `message`
e campos específicos do tipo de falha.

```ts
const result = z.string().safeParse(12);

result.error.issues;
// [{ expected: "string", code: "invalid_type", path: [],
//    message: "Invalid input: expected string, received number" }]
```

Em objeto com várias falhas, as issues são acumuladas:

```ts
const result = schema.safeParse({
  username: 1234,
  favoriteNumbers: [1234, "4567"],
  extraKey: 1234,
});

result.error!.issues;
// [
//   { code: "invalid_type", path: ["username"], message: ... },
//   { code: "invalid_type", path: ["favoriteNumbers", 1], message: ... },
//   { code: "unrecognized_keys", keys: ["extraKey"], path: [] }
// ]
```

## Refinements

`.refine(check)` adiciona uma checagem que retorna truthy para sucesso e falsy para falha.
A função não deve lançar. Gera uma issue com code `"custom"`.

```ts
const myString = z.string().refine((val) => val.length <= 255);

const myString2 = z.string().refine((val) => val.length <= 255, {
  error: "String must be 255 characters or fewer",
  abort: true, // encerra a validação se esta checagem falhar
});
```

`.superRefine((val, ctx) => ...)` permite adicionar várias issues e usar os tipos internos
de issue via `ctx.addIssue()`:

```ts
const UniqueStringArray = z.array(z.string()).superRefine((val, ctx) => {
  if (val.length > 3) {
    ctx.addIssue({
      code: "too_big",
      message: "Too many items",
      input: val,
    });
  }

  if (val.length !== new Set(val).size) {
    ctx.addIssue({
      code: "custom",
      message: "No duplicates allowed.",
      input: val,
    });
  }
});
```

## Transformação

`.transform()` é o atalho de schema da v4 classic para encadear uma transformação. Não há
equivalente em Zod Mini.

```ts
const stringToLength = z.string().transform((val) => val.length);
```

`.pipe(schema)` passa a saída de um schema como entrada do próximo:

```ts
const stringToLength = z.string().pipe(z.transform((val) => val.length));

stringToLength.parse("hello"); // => 5
```

`z.preprocess(fn, schema)` transforma a entrada bruta antes da validação:

```ts
const coercedInt = z.preprocess((val) => {
  if (typeof val === "string") {
    return Number.parseInt(val);
  }
  return val;
}, z.int());
```

## Coerção

`z.coerce` usa os construtores nativos. O tipo de entrada é `unknown` por padrão; um tipo
mais específico pode ser passado via genérico.

```ts
z.coerce.string(); // String(input)
z.coerce.number(); // Number(input)
z.coerce.boolean(); // Boolean(input)

const schema = z.coerce.string();
schema.parse("tuna"); // => "tuna"
schema.parse(42);     // => "42"
```

## Mensagens de erro

A v4 unifica a configuração no parâmetro `{ error }`, tanto no construtor quanto em
checagens:

```ts
z.string({ error: "Bad!" });
z.string().min(5, { error: "Too short!" });
z.uuid({ error: "Bad UUID!" });
z.iso.date({ error: "Bad date!" });
z.array(z.string(), { error: "Bad array!" });
z.array(z.string()).min(5, { error: "Too few items!" });
z.set(z.string(), { error: "Bad set!" });
```

Também é possível customizar por parse, com precedência menor que a mensagem definida no
schema:

```ts
// schema-level vence
const schema = z.string({ error: "highest priority" });
const result = schema.safeParse(12, {
  error: () => "lower priority",
});
result.error.issues;
// [{ message: "highest priority", ... }]
```

A função de erro per-parse recebe a issue (`iss`); use `iss.code` para distinguir os casos.

## Formatação de erros

Três funções de topo substituem o acesso manual às issues:

```ts
const tree = z.treeifyError(result.error);
tree.errors;                                    // erros no nível atual
tree.properties?.username?.errors;              // ["Invalid input: expected string, ..."]
tree.properties?.favoriteNumbers?.items?.[1]?.errors;

const flattened = z.flattenError(result.error);
flattened.formErrors;         // erros com path []
flattened.fieldErrors.username; // ["Invalid input: expected string, ..."]

const pretty = z.prettifyError(result.error);
// ✖ Unrecognized key: "extraKey"
// ✖ Invalid input: expected string, received number
//   → at username
// ✖ Invalid input: expected number, received string
//   → at favoriteNumbers[1]
```

- `treeifyError` espelha a hierarquia do schema e é adequado para formulários aninhados.
  Use optional chaining (`?.`) ao navegar.
- `flattenError` devolve `{ formErrors, fieldErrors }`, melhor para objetos rasos.
- `prettifyError` devolve uma string legível para log/terminal.

## Validação de variáveis de ambiente

A doc do Zod mostra as peças usadas para env: formatos de string, `z.enum`,
`z.stringbool` (introduzido em `zod@4.0`) e `.default`.

```ts
const strbool = z.stringbool();

strbool.parse("true");     // => true
strbool.parse("1");        // => true
strbool.parse("yes");      // => true
strbool.parse("on");       // => true
strbool.parse("enabled");  // => true

strbool.parse("false");    // => false
strbool.parse("0");        // => false
strbool.parse("off");      // => false

strbool.parse("qualquer outra coisa"); // ZodError code "invalid_value"
```

URL de env com restrição de host/protocolo:

```ts
const databaseUrl = z.url({
  hostname: /example\.com$/,
  protocol: /^https$/,
});

databaseUrl.parse("https://example.com");     // ok
databaseUrl.parse("http://example.com");      // erro
databaseUrl.parse("https://example.org");     // erro
```

Neste repo a validação fica em `src/env.js`, dentro de `createEnv` (API do
`@t3-oss/env-nextjs`, não do Zod). O schema Zod de cada variável é o que muda entre v3 e v4:

```ts
// src/env.js — trecho atual (estilo v3)
DATABASE_URL: z.string().url(),
NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
AUTH_SECRET:
  process.env.NODE_ENV === "production" ? z.string() : z.string().optional(),
```

Equivalentes v4 para os formatos:

```ts
DATABASE_URL: z.url(),
NODE_ENV: z.enum(["development", "test", "production"]).default("development"),
AUTH_SECRET: z.string().optional(),
DEBUG: z.stringbool().default(false),
```

Regras do repo que continuam valendo:

1. Toda variável nova entra no schema (`server` ou `client`), no bloco `runtimeEnv` e no
   `.env.example`.
2. Acesso em runtime é via `env` de `~/env`, nunca `process.env` direto na aplicação.
3. `SKIP_ENV_VALIDATION=1` pula a validação em builds Docker; não dependa disso em dev.
4. `emptyStringAsUndefined: true` faz `VAR=""` falhar como se estivesse ausente.
