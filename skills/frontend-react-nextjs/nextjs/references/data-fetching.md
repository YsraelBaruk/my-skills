# Data fetching, caching e Server Actions

> Fonte: `/vercel/next.js` via `ctx7 docs` (consultado em 2026-11-10).

Busque dados onde eles são usados: em Server Components assíncronos, com `fetch` ou banco.
Não crie uma Route Handler intermediária só para isso.

## `fetch` em Server Component

```tsx
export default async function Page() {
  // Data Cache: reutilizável e revalidável sob demanda.
  const cached = await fetch('https://...', { cache: 'force-cache' })
  // Sem cache: busca na origem a cada request.
  const fresh = await fetch('https://...', { cache: 'no-store' })
  // Cache com revalidação de no máximo 10 segundos.
  const timed = await fetch('https://...', { next: { revalidate: 10 } })

  return <div>...</div>
}
```

## Cache é opt-in no Next.js 15

`fetch` deixou de ser cacheado por padrão:

```ts
const a = await fetch('https://...') // não cacheada
const b = await fetch('https://...', { cache: 'force-cache' }) // cacheada
```

| Opção | Comportamento |
| --- | --- |
| `auto no cache` (padrão) | Busca a cada request em dev; no build busca uma vez se a rota for estática; com APIs de request, busca sempre |
| `no-store` | Busca na origem a cada request, mesmo sem APIs de request |
| `force-cache` | Usa o cache quando URL, método, headers e body batem; só armazena respostas `200` |

Para cachear todas as requests de um layout/page, exporte
`export const fetchCache = 'default-cache'`. Uma opção individual `cache` vence o
`fetchCache` do segmento.

## `revalidatePath`

Invalida o cache de um caminho. Funciona em Server Functions e Route Handlers; não pode ser
chamada em Client Component.

```ts
import { revalidatePath } from 'next/cache'

export async function createPost(formData: FormData) {
  'use server'

  // ...mutação

  revalidatePath('/posts')
}
```

Assinatura: `revalidatePath(path, type?: 'page' | 'layout')`. Caminho literal dispensa
`type`; padrão dinâmico exige. `revalidatePath('/', 'layout')` invalida tudo.

## `revalidateTag`

Invalida entradas marcadas com uma tag. Na v15 a assinatura recebe apenas a tag:

```ts
'use server'

import { revalidateTag } from 'next/cache'

export async function createPost() {
  revalidateTag('posts')
}
```

O índice `canary` mostra `revalidateTag('posts', 'max')`, com segundo argumento, de versão
mais nova; neste projeto use a forma de um argumento.

## Consulta direta ao banco

Server Components podem usar ORM/client diretamente; credenciais e queries não vão para o
bundle do cliente.

```tsx
import { db } from '~/server/db'

export default async function Page() {
  const posts = await db.post.findMany({ orderBy: { createdAt: 'desc' } })
  return <div>{posts.map((post) => <p key={post.id}>{post.title}</p>)}</div>
}
```

Isso não dispensa autenticação/autorização. Neste repositório o padrão é tRPC + Prisma em
`src/server/api`; use query direta só quando a página for exclusivamente server-rendered e
não houver procedure adequada.

## Server Actions para mutações

Uma Server Action é uma função `async` de servidor usada como action. Pode ser passada para
`<form action={...}>` ou `<button formAction={...}>`; o Next.js embrulha a chamada em uma
transition e devolve UI + dados em um único roundtrip.

```ts
// app/actions.ts
'use server'

import { auth } from '@/lib/auth'

export async function createPost(formData: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }

  const title = formData.get('title')
  const content = formData.get('content')

  // muta e revalida
}
```

```tsx
import { createPost } from '@/app/actions'

export function Form() {
  return (
    <form action={createPost}>
      <input type="text" name="title" />
      <input type="text" name="content" />
      <button type="submit">Criar</button>
    </form>
  )
}
```
## Segurança de Server Actions

Actions são endpoints POST públicos. Um objeto vindo do cliente é controlado pelo cliente,
inclusive o `id`. Receba o mínimo, derive o usuário da sessão e valide a propriedade:

```ts
'use server'

import { auth } from '@/lib/auth'
import { db } from '@/lib/db'

export async function completeItem(itemId: string) {
  const session = await auth()
  if (!session?.user) return

  const item = await db.item.findFirst({
    where: { id: itemId, ownerId: session.user.id },
  })
  if (!item) return
  await db.item.update({ where: { id: item.id }, data: { completed: true } })
}
```

O padrão inseguro é aceitar a entidade inteira (`completeItem(item: Item)`) e atualizar por
`item.id` sem checar dono.

## Formulários com `useActionState`

Para exibir erros e estado pendente, o componente do formulário vira Client Component. A
action passa a receber `prevState` como primeiro argumento.

```tsx
'use client'

import { useActionState } from 'react'
import { createPost } from '@/app/actions'

export function Form() {
  const [state, formAction, pending] = useActionState(createPost, {})

  return (
    <form action={formAction}>
      <input name="title" required />
      {state?.message && <p aria-live="polite">{state.message}</p>}
      <button disabled={pending}>Criar</button>
    </form>
  )
}
```

Validação com Zod dentro da action:

```ts
'use server'

export async function signup(state: FormState, formData: FormData) {
  const validatedFields = SignupFormSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    password: formData.get('password'),
  })

  if (!validatedFields.success) {
    return { errors: validatedFields.error.flatten().fieldErrors }
  }
}
```

Erros esperados devem ser **retornados**, não lançados: `useActionState` lê o retorno.
Reserve `throw` para falhas inesperadas.

## Redirect após mutação

```ts
'use server'

import { revalidatePath } from 'next/cache'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  // ...mutação
  revalidatePath('/posts')
  redirect('/posts')
}
```

`redirect` lança uma exceção de controle de fluxo: chame por último e fora de `try/catch`.

## `use cache`, `cacheLife` e `cacheTag`

Exigem `experimental.cacheComponents: true` no `next.config`. O `next.config.js` deste
repositório está vazio, então **não estão habilitadas**.

```ts
import { cacheLife, cacheTag } from 'next/cache'

export async function getProduct(id: string) {
  'use cache'
  cacheLife('hours')
  cacheTag(`product:${id}`)
  return db.product.findUnique({ where: { id } })
}
```

Na v15.5 a referência mostra `unstable_cacheLife`/`unstable_cacheTag` e exige o flag
`cacheComponents` em `experimental`. Não introduza esse modelo sem habilitar e validar.

## Faça / não faça

- Faça: buscar dados no Server Component que os renderiza.
- Faça: declarar cache explicitamente quando a resposta puder ser reaproveitada.
- Faça: revalidar por path ou tag depois de cada mutação.
- Não faça: chamar a própria Route Handler a partir de um Server Component.
- Não faça: cachear dado por usuário sem separar a chave.
- Não faça: engolir erro esperado com `try/catch`; devolva estado ao formulário.
