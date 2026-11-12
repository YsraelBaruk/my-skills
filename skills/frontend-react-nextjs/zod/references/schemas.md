# Schemas — primitivos, objetos, coleções e composição

> Fonte: `/colinhacks/zod/v4.6.7` via `ctx7 docs` (consultado em 2026-11-10).

## Primitivos

`z.string()` valida strings e encadeia validações/transformações. Os limites contam
code points Unicode, não unidades UTF-16 — `z.string().length(1).parse("😀")` passa.

```ts
z.string().max(5);
z.string().min(5);
z.string().length(5);
z.string().nonempty(); // alias de .min(1)
z.string().regex(/^[a-z]+$/);
z.string().startsWith("aaa");
z.string().endsWith("zzz");
z.string().includes("---");
z.string().uppercase();
z.string().lowercase();

// transformações
z.string().trim().toLowerCase();
z.string().toUpperCase().normalize();
```

`z.number()` aceita apenas números finitos: rejeita `NaN` e `Infinity`.

```ts
const schema = z.number();

schema.parse(3.14);     // ok

z.number().gt(5);
z.number().gte(5);        // alias .min(5)
z.number().lt(5);
z.number().lte(5);        // alias .max(5)
z.number().positive();    // alias .gt(0)
z.number().nonnegative();
z.number().multipleOf(5); // alias .step(5)
```

`z.int()` é usado pela própria doc em exemplos de record e preprocess para exigir inteiro.
`z.boolean()` aparece como schema de tuple; `z.literal` aceita string, number, bigint e
boolean.

## Objetos

`z.object(shape)` exige todas as propriedades por padrão e **remove chaves desconhecidas**
do resultado do parse.

```ts
const Person = z.object({
  name: z.string(),
  age: z.number(),
});

type Person = z.infer<typeof Person>;
// => { name: string; age: number; }
```

### Opcional, nullable, default e catch

```ts
const Dog = z.object({
  name: z.string(),
  age: z.number().optional(),
  nickname: z.string().nullable(),
});

Dog.parse({ name: "Yeller" }); // ok

const defaultTuna = z.string().default("tuna");
defaultTuna.parse(undefined); // => "tuna"

const numberWithCatch = z.number().catch(42);
numberWithCatch.parse(5);      // => 5
numberWithCatch.parse("tuna"); // => 42

const numberWithRandomCatch = z.number().catch((ctx) => {
  ctx.error; // ZodError capturado
  return Math.random();
});
```

Formas standalone: `z.optional(schema)`, `z.nullable(schema)`, `z._default(schema, valor)` (Mini), `z.catch(schema, fallback)`.

### Strictness

```ts
const StrictDog = z.strictObject({ name: z.string() });
StrictDog.parse({ name: "Yeller", extraKey: true }); // erro

const DogWithStrings = z
  .object({
    name: z.string(),
    age: z.number().optional(),
  })
  .catchall(z.string());

DogWithStrings.parse({ name: "Yeller", extraKey: "extraValue" }); // ok
DogWithStrings.parse({ name: "Yeller", extraKey: 42 });           // erro
```

- `z.object()` remove chaves desconhecidas (comportamento padrão).
- `z.strictObject()` rejeita chaves desconhecidas; `.strict()` produz o mesmo efeito.
- `z.looseObject()` não define `additionalProperties: false` na conversão para JSON Schema.
- `.catchall(schema)` valida qualquer chave não reconhecida com outro schema.

## Arrays, tuplas e records

```ts
const stringArray = z.array(z.string()); // ou z.string().array()
const MyTuple = z.tuple([z.string(), z.number(), z.boolean()]);
type MyTuple = z.infer<typeof MyTuple>; // [string, number, boolean]

const IdCache = z.record(z.string(), z.string());
type IdCache = z.infer<typeof IdCache>; // Record<string, string>
```

Na v4 a assinatura mostrada para record é `z.record(keySchema, valueSchema)`. A chave também
pode ter constraints:

```ts
const numberKeys = z.record(z.number(), z.string());
numberKeys.parse({ 1: "one", "1.5": "one", abc: "one" }); // "abc" falha

const intKeys = z.record(z.int().step(1).min(0).max(10), z.string());
```

## Unions e discriminated unions

`z.union` testa cada opção na ordem e devolve o primeiro valor válido.

```ts
const stringOrNumber = z.union([z.string(), z.number()]);
// string | number

stringOrNumber.parse("foo"); // ok
stringOrNumber.parse(14);    // ok
```

`z.discriminatedUnion` usa uma chave discriminadora e evita tentativa e erro entre ramos.

```ts
const MyResult = z.discriminatedUnion("status", [
  z.object({ status: z.literal("success"), data: z.string() }),
  z.object({ status: z.literal("failed"), error: z.string() }),
]);
```

A doc alerta que ramos de objeto podem se sobrepor porque `z.object()` remove chaves
desconhecidas. Em união exclusiva (`z.xor`), use `z.strictObject()` (ou `.strict()`) no ramo
mais estreito para tornar as opções mutuamente exclusivas.

## Enums e literals

```ts
const FishEnum = z.enum(["Salmon", "Tuna", "Trout"]);
FishEnum.parse("Salmon");   // => "Salmon"
FishEnum.parse("Swordfish"); // erro

// objeto literal com valores numéricos
const Fish = { Salmon: 0, Tuna: 1 } as const;
const FishEnum2 = z.enum(Fish);
FishEnum2.parse(Fish.Salmon); // => 0

// enum do TypeScript
enum Fish3 {
  Salmon = "Salmon",
  Tuna = "Tuna",
  Trout = "Trout",
}
const FishEnum3 = z.enum(Fish3);

const tuna = z.literal("tuna");
const colors = z.literal(["red", "green", "blue"]);

colors.parse("green");  // ok
colors.parse("yellow"); // erro
```

## Formatos de string da v4

São schemas de topo, não métodos de `z.string()`:

```ts
z.email();
z.email({ pattern: z.regexes.rfc5322Email });
z.email({ pattern: z.regexes.unicodeEmail });

z.url();
z.url({ hostname: /^example\.com$/ });
z.url({ protocol: /^https$/ });

z.uuid();
z.uuid({ version: "v4" });
z.uuidv4();
z.guid();

z.iso.datetime();                    // exige UTC com Z
z.iso.datetime({ offset: true });    // aceita +02:00
z.iso.datetime({ local: true });     // aceita datetime sem timezone
z.iso.datetime({ precision: 3 });    // milissegundos
```

## Composição de objetos

```ts
const Recipe = z.object({
  title: z.string(),
  description: z.string().optional(),
  ingredients: z.array(z.string()),
});

const JustTheTitle = Recipe.pick({ title: true });
const RecipeNoId = Recipe.omit({ id: true });
const PartialRecipe = Recipe.partial();
const DogWithBreed = Dog.extend({ breed: z.string() });
```

- `.extend(shape)` adiciona/sobrescreve campos. A doc avisa que chaves em comum são
  sobrescritas.
- `.pick({ chave: true })` e `.omit({ chave: true })` espelham `Pick`/`Omit` do TypeScript.
- `.partial()` deixa todas as propriedades opcionais; `z.deepPartial(schema)` faz isso
  recursivamente (e lança em objetos com refinements).
- `.extend()` encadeado é caro para o `tsc`; a doc recomenda spread para juntar shapes:

```ts
const DogWithBreed = z.object({
  ...Animal.shape,
  ...Pet.shape,
  breed: z.string(),
});
```

### Sobre `.merge()`

A consulta específica sobre `.merge()` não retornou esse método na documentação v4
indexada. O que apareceu foi: prefira `.extend()` (ou spread) a `z.intersection(A, B)`,
porque a interseção devolve um `ZodIntersection` e perde métodos de objeto como `pick` e
`omit`.

```ts
const EmployedPerson = z.intersection(Person, Employee);
type EmployedPerson = z.infer<typeof EmployedPerson>; // Person & Employee
```

Se precisar de `.merge()` em código existente, confirme na versão instalada antes de usá-lo;
não assuma que ele existe na v4 apenas por hábito de v3.
