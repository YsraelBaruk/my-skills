# App Router: rotas, layouts, boundaries e Route Handlers

> Fonte: `/vercel/next.js` via `ctx7 docs` (consultado em 2026-11-10).

O segmento de URL vem dos nomes das pastas; arquivos especiais definem o comportamento.

## Arquivos de rota

| Arquivo | Papel |
| --- | --- |
| `layout.tsx` | UI compartilhada que envolve `template`, `error`, `loading`, `not-found` e `page` |
| `page.tsx` | Torna o segmento acessível |
| `loading.tsx` | UI de carregamento com Suspense automático |
| `not-found.tsx` | UI para `notFound()` |
| `error.tsx` | Error boundary do segmento (Client Component) |
| `route.ts` | Endpoint HTTP (Route Handler) |

## Root layout e layouts aninhados

O `app` precisa de um root layout com `<html>` e `<body>` próprios.

```tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="pt-BR">
      <body>{children}</body>
    </html>
  )
}
```

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return <section>{children}</section>
}
```

O layout envolve o segmento e os filhos e **não re-renderiza** na navegação entre eles.

## Route groups

Pasta entre parênteses `(nome)` organiza arquivos e não entra na URL:
`app/(marketing)/sobre/page.tsx` responde em `/sobre`. Use para dar layouts distintos a
áreas distintas (site público x área autenticada) sem criar segmento extra.

## Segmentos dinâmicos

```text
app/blog/[slug]/page.tsx     -> /blog/hello       params: { slug: 'hello' }
app/shop/[...slug]/page.tsx  -> /shop/a/b/c       params: { slug: ['a', 'b', 'c'] }
```

```tsx
export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  return <h1>{slug}</h1>
}
```

Catch-all chega como array: em `app/shop/[...slug]/page.tsx`, o tipo é
`params: Promise<{ slug: string[] }>` e `/shop/a/b/c` vira `['a', 'b', 'c']`.

## `searchParams` em pages

É `Promise` e só existe em `page.tsx`. Layout não recebe o prop porque não re-renderiza na
navegação e os valores ficariam obsoletos.

```tsx
export default async function Page({
  searchParams,
}: {
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}) {
  const { page = '1', sort = 'asc', query = '' } = await searchParams

  return (
    <div>
      <p>Busca: {query}</p>
      <p>Página: {page}</p>
      <p>Ordenação: {sort}</p>
    </div>
  )
}
```

Em Client Component (não pode ser `async`), leia com `use`:

```tsx
'use client'

import { use } from 'react'

export default function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = use(params)
  return <p>{slug}</p>
}
```

## `generateStaticParams`

Gera segmentos dinâmicos no build. Em segmentos aninhados, o filho recebe os params do pai
de forma síncrona (`params` aqui não é Promise).

```tsx
export async function generateStaticParams() {
  const posts = await fetch('https://.../posts').then((res) => res.json())
  return posts.map((post) => ({ slug: post.slug }))
}

export default async function Page({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  // ...
}
```

Para catch-all, retorne arrays: `[{ slug: ['a', '1'] }]`.

## `loading.tsx`

Um arquivo `loading.tsx` no segmento exporta a UI de carregamento
(`export default function Loading() { return <p>Carregando...</p> }`) e cria um boundary de
Suspense em volta da page e dos filhos.

## `error.tsx`

Precisa de `'use client'`. Não captura erros de event handlers nem de código assíncrono fora
da renderização.

```tsx
'use client'

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  // Registre `error` em um serviço de log
  return (
    <div>
      <h2>Algo deu errado</h2>
      <button onClick={() => reset()}>Tentar novamente</button>
    </div>
  )
}
```

Em produção, `error.message` de Server Component é genérico; use `error.digest` para
correlacionar com os logs. `global-error.tsx` substitui o root layout e precisa renderizar
`<html>` e `<body>` próprios. Confirme as props da sua versão com
`ctx7 docs /vercel/next.js/v15.5.28 "global-error props"`.

## `not-found.tsx` e `notFound()`

```tsx
// app/user/[id]/page.tsx
import { notFound } from 'next/navigation'

async function fetchUser(id: string) {
  const res = await fetch(`https://.../${id}`)
  if (!res.ok) return undefined
  return res.json()
}

export default async function Profile({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const user = await fetchUser(id)

  if (!user) {
    notFound()
  }

  return <h1>{user.name}</h1>
}
```

```tsx
// app/user/[id]/not-found.tsx
export default function NotFound() {
  return <h1>Usuário não encontrado</h1>
}
```

`notFound()` lança `NEXT_HTTP_ERROR_FALLBACK;404` e precisa ser chamado no caminho de render
(dentro de algo que o componente aguarda); em promise não aguardada, o 404 não renderiza.
Funciona em Server Components, Server Functions e Route Handlers.

## `redirect()`

```tsx
import { redirect } from 'next/navigation'

export default async function Page({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const team = await fetchTeam(id)

  if (!team) {
    redirect('/login')
  }

  // ...
}
```

Lança `NEXT_REDIRECT`: chame fora de `try/catch`. Em Server Action, navega no cliente ou
responde 303 sem JavaScript; em Client Component, só durante render (em handler use
`useRouter`).

## Route Handlers

`route.ts` define métodos HTTP nomeados e não pode coexistir com `page.tsx` no mesmo
segmento.

```ts
// app/api/users/[id]/route.ts
import { type NextRequest } from 'next/server'

export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> },
) {
  const { id } = await params
  const query = request.nextUrl.searchParams.get('q')
  return Response.json({ id, query })
}

export async function POST(request: Request) {
  const body = await request.json()
  return Response.json(body, { status: 201 })
}
```

No Next.js 15, `GET` de Route Handler não é cacheado por padrão. Para cachear:

```ts
export const dynamic = 'force-static'

export async function GET() {}
```

## Faça / não faça

- Faça `await params` e `await searchParams` em Server Components.
- Faça `use(params)` somente em Client Components.
- Não leia `searchParams` em `layout.tsx`.
- Não coloque `page.tsx` e `route.ts` no mesmo segmento.
- Não assuma cache em `GET`; declare `dynamic`/`revalidate` quando quiser.
