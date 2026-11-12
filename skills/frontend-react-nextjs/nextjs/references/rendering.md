# Rendering: Server Components, Client Components e streaming

> Fonte: `/vercel/next.js` via `ctx7 docs` (consultado em 2026-11-10).

No App Router, todo componente é **Server Component por padrão**. Client Components são a
exceção, marcada explicitamente com `'use client'`.

## Server Components

- Executam no servidor, inclusive durante o build.
- Podem ser `async` e fazer I/O direto (`fetch`, ORM, banco).
- Credenciais e lógica de query não entram no bundle do cliente.

```tsx
import { db, posts } from '@/lib/db'

export default async function Page() {
  const allPosts = await db.select().from(posts)
  return (
    <ul>
      {allPosts.map((post) => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  )
}
```

Consultar o banco direto no Server Component é permitido, mas ainda exige autenticação e
autorização. Neste repositório, prefira os routers/procedures de `src/server/api` antes de
abrir uma segunda camada de acesso.

## Client Components

Adicione `'use client'` no topo do arquivo, antes dos imports, quando precisar de estado,
eventos ou APIs do browser.

```tsx
'use client'

import { useState } from 'react'

export default function Counter() {
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>{count} curtidas</p>
      <button onClick={() => setCount(count + 1)}>Curtir</button>
    </div>
  )
}
```

A diretiva marca uma **fronteira entre os module graphs** de servidor e cliente. Tudo que o
arquivo importa e os componentes que ele renderiza diretamente entram no bundle do cliente;
não é necessário repetir `'use client'` em cada arquivo filho.

O que **não** entra no bundle do cliente: Server Components passados como `children` ou
outras props. Eles são renderizados no servidor e chegam ao Client Component como resultado
já renderizado.

## Client Component não pode ser `async`

Para ler `params`/`searchParams` em um Client Component, use `use` de `react`:

```tsx
'use client'

import { use } from 'react'

export default function Page({
  params,
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = use(params)
  return <h1>{slug}</h1>
}
```

## Interleaving: server dentro de client

Passe o Server Component como `children` de um Client Component. O servidor renderiza o
filho antecipadamente; o cliente recebe o payload pronto e mantém apenas o estado da
interação.

```tsx
// app/page.tsx (Server Component)
import { Cart } from './cart'
import { Modal } from './modal'

export default async function Page() {
  return (
    <Modal>
      <Cart />
    </Modal>
  )
}
```

```tsx
// app/modal.tsx (Client Component)
'use client'

import { useState } from 'react'

export function Modal({ children }: { children: React.ReactNode }) {
  const [open, setOpen] = useState(false)

  return (
    <>
      <button onClick={() => setOpen(true)}>Abrir carrinho</button>
      {open && <div role="dialog">{children}</div>}
    </>
  )
}
```

`Cart` pode consultar o banco e continua sendo Server Component, porque não é importado
pelo módulo `'use client'`.

## Props são serializadas

Tudo que atravessa a fronteira server → client é serializado no payload do React Server
Component. Seja deliberado sobre o que passa como prop: não envie objetos de banco inteiros,
campos sensíveis nem funções comuns (só Server Functions).

```tsx
// Prefira um DTO mínimo ao invés da entidade completa
const posts = await getPosts()
return <PostList posts={posts.map(({ id, title }) => ({ id, title }))} />
```

## Diretiva `'use server'`

Use `'use server'` para declarar Server Functions (Server Actions). A diretiva pode ficar no
topo do arquivo (todos os exports) ou no topo de uma função (só aquela função). Só vale em
módulos de servidor, nunca em arquivo `'use client'`.

```ts
// app/actions.ts
'use server'

import { auth } from '@/lib/auth'

export async function createPost(formData: FormData) {
  const session = await auth()
  if (!session?.user) {
    throw new Error('Unauthorized')
  }

  // ...
}
```

Regras práticas:

- Todo export coberto por `'use server'` no topo do arquivo precisa ser `async`.
- Para importar a função em um Client Component, a diretiva deve estar no topo do módulo
  servidor; o cliente recebe uma referência que executa no servidor.
- A função é um endpoint POST público: valide entrada e autorização dentro dela.

## Streaming com Suspense

Conteúdo fora do boundary é enviado imediatamente; o conteúdo assíncrono entra depois.

```tsx
import { Suspense } from 'react'
import BlogList from '@/components/BlogList'
import BlogListSkeleton from '@/components/BlogListSkeleton'

export default function BlogPage() {
  return (
    <div>
      <header>
        <h1>Blog</h1>
        <p>Últimos posts abaixo.</p>
      </header>
      <main>
        <Suspense fallback={<BlogListSkeleton />}>
          <BlogList />
        </Suspense>
      </main>
    </div>
  )
}
```

Boundaries irmãos streamam em paralelo, sem bloquear um ao outro:

```tsx
import { Suspense } from 'react'
import { Revenue } from './revenue'
import { RecentOrders } from './recent-orders'

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>
      <Suspense fallback={<p>Carregando receita...</p>}>
        <Revenue />
      </Suspense>
      <Suspense fallback={<p>Carregando pedidos...</p>}>
        <RecentOrders />
      </Suspense>
    </div>
  )
}
```

`loading.tsx` é um atalho para envolver a page inteira em Suspense. Para streamar apenas uma
parte da tela, use `<Suspense>` explicitamente.

## Hooks de navegação (client)

`useRouter`, `usePathname` e `useSearchParams` vêm de `next/navigation` e só funcionam em
Client Components. O `useRouter` do App Router não expõe `pathname`/`query`.

```tsx
'use client'

import { usePathname, useRouter, useSearchParams } from 'next/navigation'

export default function ExampleClientComponent() {
  const router = useRouter()
  const pathname = usePathname()
  const searchParams = useSearchParams()

  return (
    <button type="button" onClick={() => router.push('/dashboard')}>
      {pathname} — {searchParams.get('tab')}
    </button>
  )
}
```

Prefira `<Link>` a `router.push` quando a navegação for um link normal.

## Faça / não faça

- Faça: manter `'use client'` na folha da árvore, não no topo de tudo.
- Faça: passar Server Components como `children` para client components interativos.
- Faça: usar `<Suspense>` em volta de cada bloco assíncrono independente.
- Não faça: marcar um componente como client só para buscar dados.
- Não faça: esperar hooks de browser, `useState` ou handlers em Server Component.
- Não faça: passar a entidade crua do Prisma para um Client Component.
- Não faça: esquecer que `'use server'` cria um endpoint POST acessível fora da UI.
