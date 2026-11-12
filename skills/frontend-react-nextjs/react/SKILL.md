---
name: frontend-react-nextjs-react
description: Referência atual do React 19 obtida do Context7. Use ao escrever componentes, hooks, estado, efeitos ou otimizações de renderização neste projeto.
---

# React 19 — referência do projeto

Fonte: `/reactjs/react.dev` via `ctx7 docs` (consultado em 2026-11-10). Os excertos vêm da
documentação oficial indexada pelo Context7, incluindo `learn/*` e `reference/react/*`.
Quando a resposta não estiver aqui, confirme com:

```bash
ctx7 docs /reactjs/react.dev "<pergunta específica>"
```

O projeto usa React 19 com Next.js 15 (App Router) e TypeScript `strict`. Toda entrada
dinâmica passa por tRPC/Zod; componentes de rota ficam em `_components/` e o acesso a dados
server-side em `src/server/**`.

## Onde procurar

| Tarefa | Arquivo |
| --- | --- |
| Criar componente, tipar props, `children`, composição | `references/components.md` |
| Listas, `key`, renderização condicional, eventos | `references/components.md` |
| Assinatura e regras de qualquer hook | `references/hooks.md` |
| `useState`, `useReducer`, `useContext`, estado local | `references/hooks.md` |
| `useEffect`, limpeza, dependências, `useLayoutEffect` | `references/hooks.md` |
| `useMemo`, `useCallback`, `memo`, re-renders | `references/hooks.md` e `references/best-practices.md` |
| `useTransition`, `useDeferredValue`, `useOptimistic`, Actions | `references/hooks.md` |
| Estado derivado, levantar estado, resetar com `key` | `references/best-practices.md` |
| Server Components x Client Components (`'use client'`) | `references/best-practices.md` |
| Revisar anti-patterns e checklist antes de commitar | `references/best-practices.md` |

## Mudanças do React 19 encontradas na documentação

- **`ref` é uma prop comum.** `forwardRef` não é mais necessário e será descontinuado:
  `function MyInput({ ref }) { return <input ref={ref} /> }`. `forwardRef` segue listado
  como API legada.
- **Actions e estado de formulário.** `<form action={fn}>` aceita função assíncrona ou
  Server Function; React embrulha o envio em uma Transition. O ecossistema inclui
  `useActionState`, `useOptimistic` e `useFormStatus` (de `react-dom`).
- **`use(resource)`.** Lê Promise (suspende e exige `<Suspense>`) ou Context; diferente de
  `useContext`, pode ser chamado dentro de `if` e laços.
- **Provider de Context simplificado.** A partir do React 19, `<SomeContext value={...}>`
  funciona diretamente; `<SomeContext.Provider>` é o formato antigo.
- **Metadados de documento.** `<meta>`, `<title>` e afins podem ser renderizados em qualquer
  componente; React faz hoist para o `<head>` (exceto `<meta itemProp>`).
- **`useEffectEvent`.** Separa lógica não reativa de um Effect; sempre vê os valores mais
  recentes e não entra no array de dependências. Só pode ser chamado de dentro de Effects.
- **`browser()` (react-dom, blog 19.3).** `use(browser())` marca o componente como
  browser-only: suspende no servidor e não suspende no cliente após hidratação.
- **APIs removidas no React 19.** `findDOMNode`, `hydrate`, `render`,
  `unmountComponentAtNode`, `renderToNodeStream`, `renderToStaticNodeStream`,
  `createFactory` e, em classes, `static propTypes`, `static contextTypes`,
  `static childContextTypes`, `getChildContext` e `this.refs`.
- **Sem `defaultProps` em função.** Use valor padrão na desestruturação; ele vale para prop
  ausente ou `undefined`, não para `null` nem `0`.
