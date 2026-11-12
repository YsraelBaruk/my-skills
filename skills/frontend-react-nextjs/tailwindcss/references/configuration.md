# Configuração: instalação, tema CSS-first e fontes

> Fonte: `/tailwindlabs/tailwindcss.com` via `ctx7 docs` (consultado em 2026-11-10).

Este repositório usa Tailwind CSS 4 com `@tailwindcss/postcss` e **sem** `tailwind.config.js`.
Todo design token vive em `src/styles/globals.css`, dentro de `@theme`.

## 1. Instalação e PostCSS

```shell
npm i tailwindcss @tailwindcss/postcss
```

```js
// postcss.config.mjs
export default {
  plugins: {
    "@tailwindcss/postcss": {},
  },
};
```

O plugin PostCSS mora em um pacote dedicado no v4. Import e vendor prefixing são feitos
automaticamente, então **não** inclua `postcss-import` nem `autoprefixer`.

O CSS de entrada é uma única linha (o arquivo deste repo):

```css
/* src/styles/globals.css */
@import "tailwindcss";
```

Preflight (reset) é injetado automaticamente na camada `base`. A composição interna é
`theme`, `base`, `components`, `utilities` (via `tailwindcss/theme.css`,
`tailwindcss/preflight.css` e `tailwindcss/utilities.css`).

## 2. `@theme`: tokens em CSS

`@theme` declara variáveis que geram utilitários e variantes. Namespaces relevantes:

| Namespace          | Gera                                              |
| ------------------ | ------------------------------------------------- |
| `--color-*`        | `bg-red-500`, `text-sky-300`, `border-*`           |
| `--font-*`         | `font-sans`, `font-poppins`                        |
| `--text-*`         | `text-xl` (tamanho de fonte)                       |
| `--breakpoint-*`   | variantes `sm:`, `3xl:`                            |
| `--container-*`    | `@sm:` e `max-w-md`                                |
| `--spacing-*`      | `px-4`, `max-h-16`, `size-4`                       |
| `--radius-*`       | `rounded-sm`                                       |
| `--shadow-*`       | `shadow-md`                                        |
| `--ease-*`         | `ease-out`                                         |
| `--animate-*`      | `animate-spin`                                     |

Exemplo (o `--font-sans` deste repo segue esse padrão):

```css
@import "tailwindcss";

@theme {
  --font-display: "Satoshi", "sans-serif";

  --breakpoint-3xl: 120rem;

  --color-avocado-100: oklch(0.99 0 0);
  --color-avocado-500: oklch(0.84 0.18 117.33);

  --ease-fluid: cubic-bezier(0.3, 0, 0, 1);
  --ease-snappy: cubic-bezier(0.2, 0, 0, 1);
}
```

```html
<h1 class="font-display text-avocado-500 ease-fluid 3xl:text-6xl">...</h1>
```

Adicionar variáveis em um namespace **estende** o padrão (equivalente ao antigo `extend`).
Para substituir o namespace inteiro, zere com `initial`:

```css
@theme {
  --color-*: initial;
  --color-white: #fff;
  --color-midnight: #121063;
  --color-tahiti: #3ab7bf;
}
```

Para descartar o tema inteiro: `--*: initial;` seguido dos seus tokens.

## 3. `@theme inline`, `@theme static` e compartilhamento

Use `inline` quando o token referencia outra variável CSS dinâmica (ex.: tema em runtime):

```css
:root {
  --acme-canvas-color: oklch(0.967 0.003 264.542);
}

[data-theme="dark"] {
  --acme-canvas-color: oklch(0.21 0.034 264.665);
}

@theme inline {
  --color-canvas: var(--acme-canvas-color);
}
```

Use `static` quando precisar de **todas** as variáveis no build, mesmo as não usadas em
classes: `@theme static { --color-primary: var(--color-red-500); }`.

Tokens podem viver em um CSS separado e ser importados, útil em monorepo:

```css
/* packages/brand/theme.css */
@theme {
  --font-body: Inter, sans-serif;
  --color-lagoon: oklch(0.72 0.11 221.19);
}
```

```css
/* app.css */
@import "tailwindcss";
@import "../brand/theme.css";
```

## 4. Funções e camadas

`--spacing()` converte um número da escala em comprimento:

```css
.my-element {
  margin: --spacing(4);
}
/* compila para: margin: calc(var(--spacing) * 4); */
```

O default é `--spacing: 0.25rem`, e utilitários numéricos usam
`calc(var(--spacing) * <number>)` (ex.: `h-15` = `3.75rem`, `size-2.5` = `0.625rem`).

Estilos base próprios entram em `@layer base`, acima do Preflight:

```css
@layer base {
  a {
    color: var(--color-blue-600);
    text-decoration-line: underline;
  }
}
```

## 5. `@source`: quais arquivos são escaneados

O Tailwind trata os arquivos como texto puro e ignora: arquivos do `.gitignore`, `node_modules`,
binários (imagens, vídeos, zip), arquivos CSS e lockfiles de gerenciador de pacotes.

Registro explícito e exclusão:

```css
/* registro explícito, sem detecção automática */
@import "tailwindcss" source(none);

@source "../src";
@source "../../packages/ui";

/* ignora diretórios grandes/legados (v4.1) */
@source not "./src/components/legacy";
```

Safelist por CSS (v4.1), substituindo a opção `safelist` do config — `@source inline("underline");`.

## 6. `@custom-variant`

```css
@custom-variant data-checked (&[data-ui~="checked"]);

@custom-variant theme-midnight {
  &:where([data-theme="midnight"] *) {
    @slot;
  }
}
```

## 7. Plugins e compatibilidade com JS

Plugin legado (JS) é carregado por `@plugin`:

```css
@plugin "@tailwindcss/typography";
```

`@config` existe apenas para compatibilidade com configs v3 e pode conviver com `@theme`,
`@utility` e `@custom-variant` durante migração incremental; o que está em CSS tem precedência.

## 8. Migração de `tailwind.config.js`

Ferramenta oficial (Node 20+):

```sh
npx @tailwindcss/upgrade
```

Mudanças que afetam este repo: `@tailwind base/components/utilities` → `@import "tailwindcss";`
e as renomeações abaixo.

| v3               | v4              |
| ---------------- | --------------- |
| `shadow-sm`      | `shadow-xs`     |
| `shadow`         | `shadow-sm`     |
| `drop-shadow-sm` | `drop-shadow-xs`|
| `blur-sm`        | `blur-xs`       |
| `rounded-sm`     | `rounded-xs`    |
| `rounded`        | `rounded-sm`    |
| `outline-none`   | `outline-hidden`|
| `ring`           | `ring-3`        |

Outros pontos:

- `border` e `ring` passam a usar `currentColor`; `ring` tem 1px por padrão. `ring-3` preserva
  os 3px do v3.
- `outline-<number>` já implica `outline-style: solid`.
- O modificador `!` agora vai no fim: `bg-red-500!` (o formato antigo ainda funciona, mas está
  depreciado).
- `@layer components` deve virar `@utility`:

```css
/* v3 */
@layer components {
  .btn {
    border-radius: 0.5rem;
    padding: 0.5rem 1rem;
  }
}

/* v4 */
@utility btn {
  border-radius: 0.5rem;
  padding: 0.5rem 1rem;
}
```

- O `container` não tem mais opções `center`/`padding`; recrie com `@utility`:

```css
@utility container {
  margin-inline: auto;
  padding-inline: 2rem;
}
```
