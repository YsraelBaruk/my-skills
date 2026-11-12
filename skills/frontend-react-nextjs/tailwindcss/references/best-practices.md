# Boas práticas: ordem, reuso, acessibilidade e scan

> Fonte: `/tailwindlabs/tailwindcss.com` via `ctx7 docs` (consultado em 2026-11-10).

## 1. Ordem das classes: deixe o Prettier decidir

`prettier-plugin-tailwindcss` já está nas devDependencies e ordena as classes. Não ordene à
mão; rode `npm run format:write`.

```json
{
  "plugins": ["prettier-plugin-tailwindcss"]
}
```

Antes e depois:

```html
<!-- antes -->
<button class="text-white px-4 sm:px-8 py-2 sm:py-3 bg-sky-700 hover:bg-sky-800">Submit</button>

<!-- depois -->
<button class="bg-sky-700 px-4 py-2 text-white hover:bg-sky-800 sm:px-8 sm:py-3">Submit</button>
```

Regras que o plugin aplica:

- utilitários de layout/alto impacto primeiro, decorativos no fim;
- relacionados agrupados: `ml-4 flex h-24 border-2 border-gray-300 p-3 text-gray-700 shadow-md`;
- variantes de estado (`hover:`, `focus:`) agrupadas depois das classes sem variante;
- variantes responsivas no fim, do menor para o maior.

```html
<div class="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4"></div>
```

Consequência prática: ao revisar um diff, reordenação de classes é ruído esperado do Prettier,
não mudança de comportamento.

## 2. Reuso: componente antes de `@apply`

A recomendação da documentação é clara: para reusar estilo entre arquivos, crie um
**componente** (React/Svelte/Vue) ou partial de template, não CSS customizado com `@apply`.

```tsx
// src/app/_components/button.tsx
export function Button(props: React.ComponentProps<"button">) {
  return (
    <button
      {...props}
      className="rounded-md bg-violet-500 px-4 py-2 text-sm font-medium text-white hover:bg-violet-600 focus-visible:outline-2 focus-visible:outline-offset-2"
    />
  );
}
```

Isso mantém uma única fonte de verdade e permite variantes por props — desde que as classes
apareçam **completas** no código (ver seção 6).

## 3. `@apply`: quando e como

`@apply` é útil em CSS global pequeno; não é a ferramenta primária de componentização.

Ao usar `@layer components`, prefira `@utility`: ele entra na camada `utilities` e recebe
ordenação automática por quantidade de propriedades.

```css
/* preferido no v4 */
@utility btn {
  border-radius: 0.5rem;
  padding: 0.5rem 1rem;
  background-color: ButtonFace;
}
```

Stylesheets processados separadamente (CSS modules, `<style>` de Vue/Svelte/Astro) não veem
tema, utilitários e variantes do CSS global. Nesse caso, importe a referência **sem duplicar
CSS** no bundle:

```css
@reference "../app.css";

button {
  @apply bg-blue-500;
}
```

Alternativa com melhor performance: usar a variável do tema direto, sem `@apply`.

```css
button {
  background-color: var(--color-blue-500);
}
```

Neste repo (Next.js App Router + CSS global em `src/styles/globals.css`), o caminho normal é
classes no JSX; `@apply` fica restrito ao CSS global.

## 4. Acessibilidade: foco e contraste

Foco por teclado deve ser visível e usar `focus-visible` para não incomodar quem usa mouse:

```html
<button class="focus-visible:outline-2 focus-visible:outline-offset-2">Submit</button>
```

- `outline-hidden` esconde o outline do navegador **preservando** o modo de cores forçadas;
  use quando você desenha seu próprio indicador de foco.
- `outline-none` remove o outline de verdade (`outline-style: none`) — só use se houver
  substituto visível.
- `outline-<number>` já define `outline-style: solid`; não precisa combinar com `outline`.
- Preflight não define contraste: escolha pares texto/fundo com contraste suficiente
  (WCAG AA). A documentação consultada não traz tabela de contraste — valide com ferramenta
  externa ou confirme com `ctx7 docs /tailwindlabs/tailwindcss.com "accessibility contrast"`.
- Não remova `:focus` sem substituto; `focus:outline-hidden` combinado com
  `focus:border-indigo-600` é o exemplo da doc para inputs.

```html
<input class="focus:border-indigo-600 focus:outline-hidden" type="text" />
```

## 5. Dark mode: escolha uma estratégia

- **Automático** (`prefers-color-scheme`): é o default; não requer nada no CSS.
- **Manual** (classe/atributo): declare a variante uma vez no CSS global.

```css
@custom-variant dark (&:where(.dark, .dark *));
```

Depois alterne `<html class="dark">` via JS (e sincronize a preferência, ex.: `localStorage`).
Não misture as duas estratégias no mesmo projeto: `dark:` passa a significar apenas uma coisa.

## 6. Scan e performance do build

O Tailwind varre os arquivos como **texto**, sem interpretar a linguagem. Ele ignora
`.gitignore`, `node_modules`, binários, CSS e lockfiles.

Nunca construa nomes de classe dinamicamente:

```html
<!-- não gera text-red-600 nem text-green-600 -->
<div class="text-{{ error ? 'red' : 'green' }}-600"></div>
```

Mapeie para classes completas:

```tsx
const tone = {
  error: "text-red-600",
  success: "text-green-600",
} as const;

<div className={tone[status]} />
```

Boas práticas de scan:

- Prefira strings literais completas; interpolação quebra a detecção.
- Se um diretório grande não usa Tailwind (legado, vendor), exclua com
  `@source not "./src/components/legacy";` (v4.1).
- Em CSS com múltiplas entradas, desligue a detecção automática e registre fontes:
  `@import "tailwindcss" source(none);` + `@source "../src";`.
- Para classes geradas em runtime e impossíveis de detectar, use `@source inline("...")`
  (v4.1/substituto do `safelist`) em vez de espalhar strings falsas no código.
- Evite `@apply` em massa: cada uso obriga o Tailwind a processar mais CSS.

## 7. Armadilhas do v4 ao revisar classes

- `border` e `ring` usam `currentColor`; informar a cor é obrigatório na prática.
- `ring` tem 1px; para o visual antigo de 3px use `ring-3`.
- `shadow-sm`/`rounded-sm`/`blur-sm` mudaram de escala; confira a tabela em
  `references/configuration.md`.
- `outline-none` ≠ `outline-hidden`: o primeiro remove, o segundo preserva forced-colors.
- Modificador `!` no fim (`bg-red-500!`).
- `@layer components` não é mais o lugar de utilitário próprio; use `@utility`.
