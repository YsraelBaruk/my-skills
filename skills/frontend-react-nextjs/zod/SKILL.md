---
name: frontend-react-nextjs-zod
description: Referência atual do Zod v4 obtida do Context7. Use ao definir schemas, validar input de procedures/env, transformar dados ou formatar erros de validação.
---

# Zod v4 — referência do projeto

Fonte: `/colinhacks/zod/v4.6.7` via `ctx7 docs` (consultado em 2026-11-10). O ID sem
versão (`/colinhacks/zod`) também foi consultado e devolveu a mesma documentação v4.
Consultas cobertas: objetos, primitivos, coleções, unions/enums/literals, `parse`/`safeParse`,
mensagens e formatação de erro, refinements/transformação/coerção/pipes, inferência de tipos,
composição de objetos, novidades da v4, env e formatos de string.

Quando este guia não responder, confirme na fonte antes de escrever código:

```bash
ctx7 docs /colinhacks/zod/v4.6.7 "<pergunta específica>"
```

## Atenção: versão instalada neste repo

`package.json` declara `"zod": "^3.24.2"` (linha 37) e ainda não há `node_modules`
instalado. A documentação abaixo é **v4**. APIs como `z.email()`, `z.url()`,
`z.strictObject` e `{ error }` são v4; o código atual usa `z.string().url()` e
`error.cause.flatten()`, que não aparecem na doc v4 consultada.

Antes de codar, confirme a versão real:

```bash
node -p "require('zod/package.json').version"
```

Se o projeto continuar em v3, não misture as duas APIs: atualize o Zod ou consulte a doc da versão instalada. Este arquivo não autoriza migração por conta própria.

## Onde procurar

| Tarefa | Arquivo |
| --- | --- |
| Tipos primitivos, `z.object`, arrays, tuplas, unions, discriminated unions, enums, literals, records | `references/schemas.md` |
| `extend`, `pick`, `omit`, `partial`, `deepPartial`, spread e composição de objetos | `references/schemas.md` |
| `parse`/`safeParse`, `refine`, `superRefine`, `transform`, `coerce`, `pipe`, `preprocess` | `references/validation-and-errors.md` |
| Mensagens customizadas, precedência de erro, `treeifyError`/`flattenError`/`prettifyError` | `references/validation-and-errors.md` |
| Validação de variáveis de ambiente (`src/env.js`) | `references/validation-and-errors.md` |
| `z.infer`, `z.input`, `z.output`, schema como fonte de tipo, input de procedures tRPC | `references/typescript-integration.md` |
| Reuso de schema entre create/update, DTOs e funções | `references/typescript-integration.md` |

## Diferenças v3 -> v4 que apareceram na documentação

- **Formatos de string viraram construtores de topo.** `z.email()`, `z.uuid()`, `z.url()`
  e `z.iso.datetime()` são schemas próprios; os métodos de string equivalentes foram
  marcados `@deprecated` a favor dos construtores de topo (a doc cita explicitamente
  `.uuid()`, `.uuidv4()`, `.guid()`). `z.uuid()` aceita `{ version }` e há `z.uuidv4()`,
  `z.uuidv6()`, `z.uuidv7()` e `z.guid()`.
- **Erro customizado unificado em `{ error }`.** `z.string({ error: "Bad!" })`,
  `.min(5, { error: "Too short!" })` e `z.array(z.string(), { error: "Bad array!" })`.
  `setErrorMap`/`getErrorMap` estão depreciados em favor de `z.config()` com `customError`.
- **Formatação de erro por funções de topo.** `z.treeifyError(error)`,
  `z.flattenError(error)` e `z.prettifyError(error)`. O `ZodError` da v4 expõe `issues`;
  o array `errors` da v3 não faz parte da interface mostrada.
- **Objetos ganharam construtores explícitos de strictness.** `z.strictObject()` rejeita
  chaves desconhecidas, `z.looseObject()` nunca marca `additionalProperties: false`, e
  `z.object()` continua removendo chaves desconhecidas por padrão. `.strict()` segue
  existindo.
- **Aliases de tipo da v3 foram depreciados.** `TypeOf`/`Infer` -> `z.output<T>`;
  `ZodTypeAny`/`ZodSchema`/`Schema` -> `z.ZodType` sem genéricos. `ZodRawShape` foi mantido
  como alias de compatibilidade.
- **O entrypoint `zod` é a API v4 classic.** A v3 vive no subpath `zod/v3`, então a v4 não
  é drop-in replacement.
- **Performance.** `z.compile(schema)` pré-compila um schema; `import "zod/compile"` compila
  todos os schemas no primeiro parse; `safeParse` de falha ficou cerca de 7.5x mais rápido
  que na 4.4 por não capturar stack trace.

## Regras do repo ao usar Zod

1. Toda entrada de procedure tRPC é validada com Zod no `.input(...)`; o tipo de `input`
   vem do schema, não de uma interface escrita à mão.
2. Variáveis de ambiente são validadas em `src/env.js` via `createEnv`; toda variável nova
   entra no schema, no `runtimeEnv` e no `.env.example`.
3. Prefira derivar schemas (`pick`, `omit`, `partial`, `extend`) a duplicar shapes.
4. Rode `npm run check` depois de qualquer mudança que toque schemas ou procedures.
