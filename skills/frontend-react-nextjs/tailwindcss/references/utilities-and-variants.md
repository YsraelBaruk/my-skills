# Utilitários, variantes, layout e `@utility`

> Fonte: `/tailwindlabs/tailwindcss.com` via `ctx7 docs` (consultado em 2026-11-10).

## 1. Utilitários comuns

Tailwind gera utilitários a partir dos tokens do tema. Espaçamento é dinâmico: qualquer
número multiplica `--spacing` (default `0.25rem`).

```html
<div class="flex items-center justify-between gap-4 rounded-lg border border-gray-300 bg-white p-4 text-sm text-gray-700 shadow-sm">
  <span class="font-medium">Título</span>
  <button class="rounded-md bg-violet-500 px-4 py-2 text-white hover:bg-violet-600">
    Salvar
  </button>
</div>
```

- `px-4`, `py-2`, `gap-4`, `size-4`, `max-h-16` → `calc(var(--spacing) * n)`.
- `h-15` é válido (`3.75rem`); frações de sizing geram percentuais (`h-1/2`).
- Cores com opacidade usam barra: `bg-black/75`, `text-white/90`.

## 2. Variantes de estado

```html
<button class="bg-violet-500 hover:bg-violet-600 focus:outline-2 focus:outline-offset-2 focus:outline-violet-500 active:bg-violet-700">
  Save changes
</button>
```

Variantes usadas com frequência: `hover`, `focus`, `focus-visible`, `active`, `disabled`,
`placeholder`, `before`, `after`, `odd`, `even`, `first`, `last`.

`focus-visible` aplica só em foco por teclado — é o par correto de `outline-*`:

```html
<button class="focus-visible:outline-2 ...">Submit</button>
<input class="focus:border-indigo-600 focus:outline-hidden" />
```

`outline-hidden` mantém o outline invisível no modo de cores forçadas; `outline-none` remove
de fato (`outline-style: none`).

## 3. Empilhamento e ordem das variantes

Variantes podem ser empilhadas na mesma classe; a leitura é da esquerda para a direita:

```html
<button class="dark:lg:data-current:hover:bg-indigo-600 ...">
```

```css
@media (prefers-color-scheme: dark) and (width >= 64rem) {
  button[data-current]:hover {
    background-color: var(--color-indigo-600);
  }
}
```

## 4. Dark mode

O default é `prefers-color-scheme`. Para controlar por classe (ou atributo), sobrescreva a
variante no CSS global:

```css
@import "tailwindcss";

@custom-variant dark (&:where(.dark, .dark *));
```

```html
<html class="dark">
  <body>
    <div class="bg-white dark:bg-black">
      <h1 class="text-gray-900 dark:text-white">Dark mode</h1>
    </div>
  </body>
</html>
```

Alternativa por atributo: `@custom-variant dark (&:where([data-theme=dark], [data-theme=dark] *));`.

Combinar com estados funciona normalmente: `dark:bg-white dark:hover:bg-gray-50`.

## 5. Responsivo (mobile-first)

Utilitário sem prefixo vale em todos os tamanhos; prefixado vale **do breakpoint para cima**.
Não use `sm:` para "celular".

```html
<!-- centraliza no mobile, alinha à esquerda a partir de 640px -->
<div class="text-center sm:text-left"></div>

<div class="grid grid-cols-3 md:grid-cols-4 lg:grid-cols-6"></div>
```

Breakpoints default (rem): `sm` 40, `md` 48, `lg` 64, `xl` 80, `2xl` 96. Cada um ganha a
variante `max-*` (intervalo abaixo do breakpoint):

| Variante  | Media query                     |
| --------- | ------------------------------- |
| `max-sm`  | `@media (width < 40rem)`        |
| `max-md`  | `@media (width < 48rem)`        |
| `max-lg`  | `@media (width < 64rem)`        |
| `max-xl`  | `@media (width < 80rem)`        |
| `max-2xl` | `@media (width < 96rem)`        |

Combine os dois para faixas: `md:max-lg:*`.

## 6. `group` e `peer`

`group` estiliza filhos a partir do estado do pai; `peer` a partir de um irmão anterior.

```html
<a href="#" class="group rounded-lg p-3 hover:bg-gray-50">
  <span class="text-gray-700 group-hover:text-violet-600">Abrir</span>
</a>

<form>
  <label for="email">Email:</label>
  <input id="email" name="email" type="email" class="peer is-dirty" required />
  <div class="hidden peer-[.is-dirty]:peer-required:block">This field is required.</div>
</form>
```

Grupos aninhados usam nome: `group/sidebar` + `group-hover/sidebar:*`.

```html
<div class="group/sidebar">
  <div class="group/navitem">
    <a class="opacity-50 group-hover/navitem:bg-black/75 group-hover/sidebar:opacity-75">...</a>
  </div>
</div>
```

Valores arbitrários aceitam seletor próprio: `group-[.is-published]:block`.

## 7. `data-*` e `aria-*`

```html
<div data-size="large" class="data-[size=large]:p-8">...</div>

<div data-size="large" class="group">
  <div class="group-data-[size=large]:p-8">...</div>
</div>

<table>
  <thead>
    <tr>
      <th aria-sort="ascending" class="group">
        Invoice #
        <svg class="group-aria-[sort=ascending]:rotate-0 group-aria-[sort=descending]:rotate-180"></svg>
      </th>
    </tr>
  </thead>
</table>
```

Também existe `aria-[sort=ascending]:...` para o próprio elemento. Atalhos comuns podem virar
variante nomeada com `@custom-variant data-checked (&[data-ui~="checked"]);`.

## 8. Valores arbitrários

```html
<div class="top-[117px] bg-[#121063] grid-cols-[1fr_500px_2fr]">...</div>
```

- `[valor]` para qualquer CSS: `w-[calc(100%-2rem)]`.
- Propriedade arbitrária: `[mask-type:luminance]`.
- Modificador `!` agora no final: `bg-red-500!`.
- Prefira tokens do tema quando o valor se repete.

## 9. Container queries

Suporte nativo no v4; não precisa do plugin `@tailwindcss/container-queries`.

```html
<div class="@container">
  <div class="grid grid-cols-1 @sm:grid-cols-3 @lg:grid-cols-4">...</div>
</div>

<div class="@container">
  <div class="flex flex-col @md:flex-row">...</div>
</div>

<div class="@container">
  <div class="flex flex-col @min-[475px]:flex-row">...</div>
</div>
```

Container queries são mobile-first como os breakpoints: `@md:` vale do tamanho do container
para cima. Tamanhos vêm de `--container-*`; valores avulsos usam `@min-[...]`/`@max-[...]`.

## 10. Layout: grid, flex e gap

```html
<div class="grid grid-cols-2 gap-4">
  <div>01</div>
  <div>02</div>
</div>

<div class="flex flex-col gap-2 md:flex-row md:items-center md:justify-between">
  <h2 class="text-lg font-semibold">Título</h2>
  <button class="rounded-md px-3 py-1.5">Ação</button>
</div>
```

`gap-<number>` aplica linhas e colunas; `gap-x-*`/`gap-y-*` separam os eixos.

## 11. `@utility`: utilitário próprio

Utilitários simples entram na camada `utilities` e já funcionam com variantes:

```css
@utility content-auto {
  content-visibility: auto;
}
```

```html
<div class="content-auto"></div>
<div class="hover:content-auto"></div>
```

Utilitários funcionais usam `--value()` (`*` no nome captura o argumento):

```css
@utility tab-* {
  tab-size: --value(integer, --default(4));
}
```

```css
/* compilado */
.tab {
  tab-size: 4;
}
.tab-2 {
  tab-size: 2;
}
```

Tipos aceitos em `--value()`: `absolute-size`, `angle`, `bg-size`, `color`, `family-name`,
`generic-name`, `image`, `integer`, `length`, `line-width`, `number`, `percentage`,
`position`, `ratio`, `relative-size`, `url`, `vector`, `*`.

Para valores arbitrários, marque o tipo com colchetes: `--value([length])`, o que habilita
`w-tab-[76]` e similares.
