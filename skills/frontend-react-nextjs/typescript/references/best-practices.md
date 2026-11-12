# Configuração estrita, erros e módulos

> Fonte: `/microsoft/typescript-website` via `ctx7 docs` (consultado em 2026-11-10).

Aplicação das opções estritas deste repositório (TypeScript 5.8, `strict`,
`noUncheckedIndexedAccess`, `verbatimModuleSyntax`) conforme a documentação oficial.

## 1. O que `strict` liga

A doc descreve `strict` como o conjunto recomendado de checagens: `strictNullChecks`,
`noImplicitAny`, `noImplicitThis` e `alwaysStrict`; `strictFunctionTypes` e
`useUnknownInCatchVariables` também têm default "true if strict".

### `strictNullChecks`

Desligado, `null`/`undefined` são ignorados e geram erro em runtime. Ligado, têm tipos
próprios e não podem ser usados onde se espera valor concreto.

```ts
function firstLetter(name?: string) {
  return name.toUpperCase(); // erro: possibly 'undefined'
}
```

### `noImplicitAny`

Erro sempre que o compilador inferiria `any`.

```ts
// @errors: 7006
function fn(s) {
  console.log(s.subtr(3));
}
```

### `strictFunctionTypes`

Parâmetros de tipos de função são checados de forma contravariante (métodos e
construtores ficam de fora para preservar compatibilidade de classes genéricas).

```ts
function fn(x: string) {
  console.log("Hello, " + x.toLowerCase());
}

type StringOrNumberFunc = (ns: string | number) => void;
let func: StringOrNumberFunc = fn; // erro 2322: atribuição insegura
```

### `useUnknownInCatchVariables`

O `catch` recebe `unknown`, exigindo narrowing.

```ts
try {
  JSON.parse("{");
} catch (err) {
  console.error(err.message); // erro: 'message' não existe em unknown

  if (err instanceof Error) {
    console.error(err.message); // ok
  }
}
```

Erro evitado: assumir que todo `throw` lança `Error`; a doc lembra que não há garantia.

## 2. `noUncheckedIndexedAccess` não vem com `strict`

O flag não é habilitado por `strict`, mas está ligado aqui. Todo acesso por índice ou
propriedade de index signature é potencialmente `undefined`.

```ts
interface EnvironmentVars {
  NAME: string;
  [propName: string]: string;
}

declare const env: EnvironmentVars;

const sysName = env.NAME; // string (declarado)
const nodeEnv = env.NODE_ENV; // string | undefined (index signature)
```

Em arrays, até loop com bounds check precisa tratar o ausente:

```ts
// @errors: 2532 18048
function screamLines(strs: string[]) {
  for (let i = 0; i < strs.length; i++) {
    console.log(strs[i].toUpperCase()); // erro: possivelmente undefined
  }
}
```

Padrões seguros documentados:

```ts
function screamLines(strs: string[]) {
  for (const str of strs) console.log(str.toUpperCase());
  strs.forEach((str) => console.log(str.toUpperCase()));
}
```

Se o índice for necessário, cheque antes (`if (value) ...`) ou use `!` apenas com
invariante real. A doc trata `!` como afirmação de confiança, não como correção de tipo.

## 3. `any` x `unknown`

`unknown` é o par type-safe de `any`: tudo é atribuível a `unknown`, mas ele só é
atribuível a ele mesmo e a `any`; nenhuma operação é permitida sem narrowing ou asserção.

```ts
function f1(a: any) {
  a.b(); // ok, mas pode quebrar em runtime
}

function f2(a: unknown) {
  a.b(); // erro: 'a' é unknown
}
```

Em `unknown` valem igualdade e pouco mais; aritmética, acesso a propriedade/índice,
chamada e `new` são proibidos.

Use `unknown` em fronteiras de confiança:

```ts
declare const someRandomString: string;

function safeParse(s: string): unknown {
  return JSON.parse(s);
}

const obj = safeParse(someRandomString); // narrowing antes de usar
```

Erro evitado: propagar o `any` de `JSON.parse`/SDK externo. Retorne `unknown` e valide com
Zod (padrão do repo) ou type guard.
## 4. Async, `Promise` e erros

Anote `Promise<T>` quando o retorno não for óbvio.

```ts
async function getFavoriteNumber(): Promise<number> {
  return 26;
}
```

`await` pode lançar: uma função async que lança gera promise rejeitada, e a doc mostra que
envolver a chamada em `try/catch` é necessário para falhas inesperadas.

```ts
const myThrowingFunction = async () => {
  throw new Error("Do not call this");
};

async function run() {
  try {
    await myThrowingFunction();
  } catch (error) {
    console.error("failed", error); // error: unknown
  }
}
```

Com `useUnknownInCatchVariables`, reduza antes de acessar:
`if (error instanceof Error) return error.message;` (o `error` é `unknown`).

Erro evitado: `catch (e: any) { e.message }` ou `error as Error` sem checagem — a asserção
não valida nada em runtime.
## 5. Módulos e imports type-only

`verbatimModuleSyntax` preserva imports/exports sem `type` e remove os marcados com `type`.

```ts
// Erased away entirely.
import type { A } from "a";

// Rewritten to 'import { b } from "bcd";'
import { b, type c, type d } from "bcd";
```

Estilo do repo (type-imports inline):

```ts
import { prisma } from "~/server/db";
import { type Post, type User } from "~/types";
```

Erro evitado: importar tipo como valor e o bundler procurar um export de runtime inexistente.

`isolatedModules`: transpiladores de arquivo único não sabem se um import é valor ou tipo;
por isso a doc exige marcação type-only.

```ts
import { someFunc, BaseType } from "./some-module.js";
//                 ^^^^^^^^
// Erro: 'BaseType' é tipo e precisa de type-only
```

O mesmo vale ao reexportar. Forma correta:

```ts
export { someFunction };
export type { someType };
```

## 6. Organização de tipos

- `interface` para formatos de objeto extensíveis; `type` para uniões, tuplas, aliases de
  primitivo e composições.
- Derive da fonte da verdade: `ReturnType<typeof fn>`,
  `Awaited<ReturnType<typeof fn>>`, `Pick<T, ...>`, `Record<Keys, T>`.
- Prefira `satisfies` a anotação quando quiser validar sem perder o tipo específico.
- Use `as const` em tabelas e derive unions com `typeof`.
- Em `switch` sobre união discriminada, inclua `default` com
  `const _exhaustiveCheck: never = value`.

## 7. Padrão de erro explícito

Derivado das seções de discriminated union e exaustividade: modele falhas esperadas em vez
de usar `throw` genérico.

```ts
type Result<T> =
  | { ok: true; value: T }
  | { ok: false; error: string };

function parseCount(input: string): Result<number> {
  const n = Number(input);
  return Number.isFinite(n)
    ? { ok: true, value: n }
    : { ok: false, error: "not a number" };
}

const result = parseCount("abc");
if (!result.ok) console.error(result.error); // narrowing por 'ok'
```

Use `try/catch` para falhas inesperadas de terceiros.

## 8. Checklist antes de commitar

- `npm run check` verde (ESLint + `tsc --noEmit`).
- Nenhum `any` novo; fronteiras externas como `unknown` validadas.
- Acesso por índice trata `undefined` (`noUncheckedIndexedAccess`).
- `catch` usa narrowing antes de ler propriedades.
- Tipos importados/exportados usam `type` (`verbatimModuleSyntax` + `isolatedModules`).
- Uniões discriminadas têm tratamento exaustivo com `never`.
