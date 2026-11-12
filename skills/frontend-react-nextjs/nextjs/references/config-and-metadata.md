# Configuração, imagens, fontes e metadata

> Fonte: `/vercel/next.js` via `ctx7 docs` (consultado em 2026-11-10).

## `next.config`

O arquivo é tipado por `NextConfig` (ou por JSDoc, como no repo). Neste projeto ele está
vazio e importa `./src/env.js` antes de exportar:

```js
import "./src/env.js";

/** @type {import("next").NextConfig} */
const config = {};

export default config;
```

Com tipagem TypeScript:

```ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  typedRoutes: true,
}

export default config
```

### `images.remotePatterns`

Libera hosts externos para o otimizador. Seja específico — a doc recomenda restringir ao
bucket/caminho real.

```ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 's3.amazonaws.com',
        port: '',
        pathname: '/my-bucket/**',
        search: '',
      },
    ],
  },
}

export default config
```

### `serverExternalPackages`

Remove dependências do bundle de Server Components para que usem `require` nativo do Node:

```js
/** @type {import('next').NextConfig} */
const nextConfig = {
  serverExternalPackages: ['@acme/ui'],
}
```

### `typedRoutes`

`typedRoutes` habilita links estaticamente tipados (requer TypeScript). A opção é estável;
use `typedRoutes`, não `experimental.typedRoutes`, no objeto de configuração do Next.

## Imagens com `next/image`

`src`, `width`, `height` e `alt` são obrigatórios no uso básico:

```tsx
import Image from 'next/image'

export default function Page() {
  return (
    <Image src="/profile.png" width={500} height={500} alt="Foto do autor" />
  )
}
```

Quando não se sabe a proporção, use `fill` dentro de um pai posicionado:

```tsx
import Image from 'next/image'
import mountains from '../public/mountains.jpg'

export default function Fill() {
  return (
    <div style={{ position: 'relative', width: 400, height: 300 }}>
      <Image
        alt="Montanhas"
        src={mountains}
        fill
        sizes="(min-width: 808px) 50vw, 100vw"
        style={{ objectFit: 'cover' }}
      />
    </div>
  )
}
```

- O pai precisa de `position: relative`, `fixed` ou `absolute`; a imagem usa
  `position: absolute` e ocupa o ancestral posicionado mais próximo.
- Sempre informe `sizes` quando o layout for responsivo.

### `sizes`

Define o tamanho por breakpoint e alimenta o `srcset`; use sempre que a imagem for
responsiva. Sem `sizes` e com `width`, o `srcset` fica limitado a 1x/2x; sem `sizes` e com
`fill`, usa só `deviceSizes`, como se ocupasse a viewport; com `sizes`, gera o `srcset`
completo em `w`.

## Fontes com `next/font`

### Google Fonts

```tsx
import { Inter, Roboto_Mono } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-inter',
})

const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  display: 'swap',
  variable: '--font-roboto-mono',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html
      lang="pt-BR"
      className={`${inter.variable} ${robotoMono.variable} antialiased`}
    >
      <body>{children}</body>
    </html>
  )
}
```

Fonte variável não precisa de `weight`.

### Fonte local

```tsx
import localFont from 'next/font/local'

const myFont = localFont({
  src: './my-font.woff2',
  display: 'swap',
})

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="pt-BR" className={myFont.className}>
      <body>{children}</body>
    </html>
  )
}
```

Arquivos de fonte podem ficar dentro de `app`.

## Metadata estática

Exporte `metadata` de um layout ou page. No root layout, configure `metadataBase` para
resolver URLs relativas.

```tsx
// app/layout.tsx
import type { Metadata } from 'next'

export const metadata: Metadata = {
  metadataBase: new URL('https://acme.com'),
  title: {
    template: '%s | Acme',
    default: 'Acme',
  },
}
```

`title.template` aplica prefixo/sufixo aos títulos dos filhos; `default` é obrigatório
quando há template. Uma page filha com `metadata = { title: 'Sobre' }` gera
`<title>Sobre | Acme</title>`.

## `generateMetadata`

Use quando a metadata depende de params, searchParams ou dados externos. `params` e
`searchParams` são `Promise`.

```tsx
import type { Metadata, ResolvingMetadata } from 'next'

type Props = {
  params: Promise<{ id: string }>
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}

export async function generateMetadata(
  { params }: Props,
  parent: ResolvingMetadata,
): Promise<Metadata> {
  const { id } = await params
  const product = await fetch(`https://.../${id}`).then((res) => res.json())

  const previousImages = (await parent).openGraph?.images || []

  return {
    title: product.title,
    openGraph: {
      images: ['/some-specific-page-image.jpg', ...previousImages],
    },
  }
}

export default function Page({ params }: Props) {
  // ...
}
```

`parent` permite **estender** a metadata do segmento pai em vez de substituí-la. Se a page
puder ser prerenderizada e `generateMetadata` não introduzir comportamento dinâmico, a
metadata já sai no HTML inicial; caso contrário, pode ser streamada após a UI inicial.

## Streaming de metadata e bots

A doc consultada (`canary`) mostra `htmlLimitedBots`, um regex de user agents que devem
receber metadata bloqueante em vez de streamada. Defina no `NextConfig`, por exemplo
`htmlLimitedBots: /MySpecialBot|SimpleCrawler/`. Usar `/.*/` desativa completamente o
streaming, ao custo de respostas mais lentas. Confirme a disponibilidade na sua versão com
`ctx7 docs /vercel/next.js/v15.5.28 "htmlLimitedBots"`.

## Faça / não faça

- Faça: usar `next/image` e restringir `remotePatterns` ao host/caminho real.
- Faça: sempre passar `alt`; `fill` exige pai posicionado e `sizes`.
- Faça: carregar fontes com `next/font` no root layout e expor CSS variables.
- Faça: definir `metadataBase` e `title.template` no layout raiz.
- Faça: `await params` dentro de `generateMetadata`.
- Não faça: apontar `remotePatterns` para `hostname: '**'`.
- Não faça: duplicar `metadata` estática e `generateMetadata` no mesmo arquivo.
