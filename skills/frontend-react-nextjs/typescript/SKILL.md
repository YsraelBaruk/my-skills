---
name: frontend-react-nextjs-typescript
description: Referência atual do TypeScript obtida do Context7. Use ao modelar tipos, generics, narrowing, utility types ou ao lidar com as opções strict deste projeto.
---

# TypeScript — referência do projeto

Fonte consultada: Context7 ID `/microsoft/typescript-website` (documentação oficial do
TypeScript no repositório `microsoft/TypeScript-website`). O `ctx7` não expôs um número de
versão; o material mais recente retornado é das release notes do TypeScript 5.0. Para
recursos mais novos, confirme com `ctx7 docs /microsoft/typescript-website "<pergunta>"`.

Este projeto usa TypeScript 5.8 em modo `strict`, com `noUncheckedIndexedAccess` e
`verbatimModuleSyntax` ligados. Os exemplos abaixo assumem esse cenário.

## Tarefa → arquivo

| Tarefa                                                              | Arquivo                              |
| ------------------------------------------------------------------- | ------------------------------------ |
| Modelar tipos, união/interseção, interfaces x aliases               | `references/types.md`                |
| Generics com constraints e defaults                                 | `references/types.md`                |
| Narrowing (`typeof`, `instanceof`, `in`, discriminated unions)      | `references/types.md`                |
| Utility types (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`, `Awaited`) | `references/types.md`        |
| Mapped types, conditional types, `satisfies`, `as const`            | `references/types.md`                |
| Configuração estrita, `any` x `unknown`, erros, módulos             | `references/best-practices.md`       |
| Organização de tipos e padrões de erro                              | `references/best-practices.md`       |

## Opções do tsconfig deste repo (o que a doc explica)

| Opção                     | Efeito prático documentado                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------ |
| `strict: true`            | Liga o conjunto recomendado de checagens. A doc cita `strictNullChecks`, `noImplicitAny`, `noImplicitThis`, `alwaysStrict`, além de `strictFunctionTypes` e `useUnknownInCatchVariables` (defaults "true if strict"). |
| `strictNullChecks`        | `null` e `undefined` passam a ter tipos próprios; usá-los onde se espera valor concreto vira erro. |
| `noImplicitAny`           | Erro sempre que o compilador inferiria `any` implicitamente (ex.: parâmetro sem tipo).     |
| `strictFunctionTypes`     | Parâmetros de tipos de função são checados de forma contravariante; atribuição insegura vira erro. |
| `useUnknownInCatchVariables` | A variável do `catch` é `unknown`, exigindo narrowing antes de acessar `.message`.        |
| `noUncheckedIndexedAccess` | Acesso por índice (`foo["bar"]`, `arr[i]`) passa a incluir `undefined` no tipo. **Não** é ligado por `strict`. |
| `verbatimModuleSyntax`    | Imports/exports sem `type` são preservados; os marcados com `type` são removidos.          |
| `isolatedModules`         | Transpiladores de arquivo único exigem que tipos importados/exportados sejam marcados como type-only. |

Consequências imediatas para quem escreve código aqui:

- Não use `any` para "resolver" erro de tipo; use `unknown` e refine (a doc mostra que
  qualquer operação em `unknown` é proibida até o narrowing).
- Indexação de array/objeto retorna `T | undefined`: trate o caso ausente, itere com
  `for...of`/`forEach` ou use `!` apenas quando houver invariante real.
- Tipos importados precisam de `import { type X }` ou `import type { X }`; reexporte com
  `export type`.
- `catch (err)` é `unknown`: verifique `err instanceof Error` antes de ler `message`.

## Regra de uso

Antes de escrever código, carregue o reference correspondente à tarefa. Se um exemplo
necessário não estiver nos references, consulte o Context7 em vez de inventar API:

```bash
ctx7 docs /microsoft/typescript-website "pergunta específica"
```
