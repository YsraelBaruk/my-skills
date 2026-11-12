# Tipos, generics, narrowing e utility types

> Fonte: `/microsoft/typescript-website` via `ctx7 docs` (consultado em 2026-11-10).

Snippets do handbook oficial, release notes e referência de utility types.

## 1. Type alias e literais

Alias só dá nome a um tipo; não cria uma versão distinta dele. O handbook mostra que
`type UserInputSanitizedString = string` continua aceitando reatribuição com `string`.
Para restringir de verdade, use literal union.

```ts
type WindowStates = "open" | "closed" | "minimized";
type PositiveOddNumbersUnderTen = 1 | 3 | 5 | 7 | 9;
```

## 2. `interface` x `type`

`interface` pode ser reaberta e estendida; `type` não. A recomendação documentada é usar
`interface` para formato de objeto e `type` para união ou tupla.

```ts
interface Animal {
  name: string;
}
interface Bear extends Animal {
  honey: boolean;
}
```

Objeto de domínio extensível → `interface`. União, tupla, alias de primitivo ou composição
`A & B` → `type` (este último via interseção).

## 3. União e interseção

União aceita um dos formatos; interseção exige todos os membros.

```ts
interface Colorful { color: string }
interface Circle { radius: number }
type ColorfulCircle = Colorful & Circle;
```

## 4. Generics: constraints e defaults

`extends` garante os membros usados no corpo.

```ts
function longest<Type extends { length: number }>(a: Type, b: Type) {
  return a.length >= b.length ? a : b;
}

const longerArray = longest([1, 2], [1, 2, 3]); // number[]
const longerString = longest("alice", "bob"); // "alice" | "bob"
longest(10, 100); // erro: number não tem length
```

Defaults tornam o parâmetro opcional. Regras: obrigatórios não vêm depois de opcionais; o
default precisa satisfazer a constraint; pode referenciar parâmetros anteriores.

```ts
type Container<T, U> = { element: T; children: U };

declare function create<
  T extends HTMLElement = HTMLDivElement,
  U extends HTMLElement[] = T[],
>(element?: T, children?: U): Container<T, U>;

const div = create(); // Container<HTMLDivElement, HTMLDivElement[]>
```

Erro evitado: `function f<T>(x: T) { return x.length }` sem constraint — `T` pode não ter `length`.

## 5. Narrowing

`typeof`:

```ts
function printId(id: number | string) {
  if (typeof id === "string") return id.toUpperCase();
  return id;
}
```

`instanceof`:

```ts
function logValue(x: Date | string) {
  if (x instanceof Date) return x.toUTCString();
  return x.toUpperCase();
}
```

`in` estreita pelo teste de existência da propriedade (inclusive na cadeia de protótipos):

```ts
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
  if ("swim" in animal) return animal.swim();
  return animal.fly();
}
```

Discriminated union: um campo literal comum permite `switch` exaustivo.

```ts
interface Circle {
  kind: "circle";
  radius: number;
}
interface Square {
  kind: "square";
  sideLength: number;
}

type Shape = Circle | Square;

function getArea(shape: Shape) {
  switch (shape.kind) {
    case "circle":
      return Math.PI * shape.radius ** 2;
    case "square":
      return shape.sideLength ** 2;
    default:
      const _exhaustiveCheck: never = shape;
      return _exhaustiveCheck;
  }
}
```

Adicionar `Triangle` sem tratar o caso quebra em `_exhaustiveCheck`: `never` não recebe
nenhum tipo restante. Type predicate (`is`):

```ts
function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}
```

Forma `parameterName is Type`, com nome de um parâmetro da função; sem o predicado, o TS
não estreita o valor após a chamada.

## 6. Utility types

```ts
interface Todo {
  title: string;
  description: string;
  completed: boolean;
  createdAt: number;
}

type TodoUpdate = Partial<Todo>; // todos os campos opcionais
type TodoPreview = Pick<Todo, "title" | "completed">;
type TodoInfo = Omit<Todo, "completed" | "createdAt">;

const todo: TodoPreview = { title: "Clean room", completed: false };
const todoInfo: TodoInfo = { title: "Pick up kids", description: "..." };
```

`Pick` seleciona, `Omit` remove e `Partial` torna os campos opcionais.

`Record` fecha o conjunto de chaves:

```ts
function mapObject<K extends string, T, U>(
  obj: Record<K, T>,
  f: (x: T) => U,
): Record<K, U>;

const names = { foo: "hello", bar: "world", baz: "bye" };
const lengths = mapObject(names, (s) => s.length); // { foo: number; ... }
```

`ReturnType` e `Awaited`:

```ts
declare function f1(): { a: number; b: string };

type T4 = ReturnType<typeof f1>; // { a: number; b: string }

type A = Awaited<Promise<string>>; // string
type C = Awaited<boolean | Promise<number>>; // boolean | number
```

Em sobrecarga, `ReturnType` usa a última assinatura. `ReturnType<any>`, `<never>`,
`<string>` e `<Function>` são erros 2344 — o argumento precisa ser tipo de função.
`Awaited` desembrulha promises recursivamente, inclusive em união.

## 7. Mapped e conditional types

Mapped percorre `keyof T`; conditional escolhe ramos com `extends`.

```ts
type FunctionPropertyNames<T> = {
  [K in keyof T]: T[K] extends Function ? K : never;
}[keyof T];

type FunctionProperties<T> = Pick<T, FunctionPropertyNames<T>>;

interface Part {
  id: number;
  name: string;
  updatePart(newName: string): void;
}

type T1 = FunctionPropertyNames<Part>; // "updatePart"
type T2 = FunctionProperties<Part>; // { updatePart(...): void }
```

O `[keyof T]` final transforma o objeto mapeado em união dos valores. Key remapping com
`as` filtra chaves:

```ts
type RemoveKindField<Type> = {
  [Property in keyof Type as Exclude<Property, "kind">]: Type[Property];
};

type KindlessCircle = RemoveKindField<Circle>; // { radius: number }
```

## 8. `satisfies` e `as const`

`satisfies` valida sem perder o tipo inferido.

```ts
type Colors = "red" | "green" | "blue";
type RGB = [red: number, green: number, blue: number];

const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  bleu: [0, 0, 255], // erro: "bleu" não existe em Colors
} satisfies Record<Colors, string | RGB>;

const greenNormalized = palette.green.toUpperCase(); // tipo preservado
```

Anotar `const palette: Record<Colors, string | RGB>` pega o typo, mas perde a
especificidade e faz `palette.green.toUpperCase()` falhar porque `green` poderia ser `RGB`.

`as const` impede alargamento e cria estruturas `readonly`:

```ts
let x = "hello" as const; // "hello"
let y = [10, 20] as const; // readonly [10, 20]
let z = { text: "hello" } as const; // { readonly text: "hello" }
```