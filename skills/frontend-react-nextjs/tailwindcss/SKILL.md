---
name: frontend-react-nextjs-tailwindcss
description: Referência atual do Tailwind CSS v4 obtida do Context7. Use ao configurar o tema, escrever utilitários/variantes ou revisar classes Tailwind neste projeto (config CSS-first, sem tailwind.config).
---

# Tailwind CSS v4 — referência do projeto

Fonte consultada: Context7 ID `/tailwindlabs/tailwindcss.com` (documentação oficial em
`tailwindlabs/tailwindcss.com`) via `ctx7 docs`. O material retornado é das docs v4 e dos
posts de lançamento v4/v4.1. O `ctx7` não expõe número de versão; este projeto declara
`tailwindcss: ^4.0.15` e `@tailwindcss/postcss: ^4.0.15`, então recursos marcados como v4.1
(`@source not`, `@source inline`) exigem a versão instalada correspondente — confirme com
`npm ls tailwindcss` ou `ctx7 docs /tailwindlabs/tailwindcss.com "<pergunta>"`.

## Como este repo usa Tailwind

- **CSS-first**: não existe `tailwind.config.js`. Todo o tema fica em
  `src/styles/globals.css`, que já começa com `@import "tailwindcss";` e um bloco `@theme`
  definindo `--font-sans`.
- **PostCSS**: o plugin é `@tailwindcss/postcss` (v4 moveu o plugin para pacote próprio;
  `postcss-import` e `autoprefixer` não são necessários).
- **Ordenação**: `prettier-plugin-tailwindcss` reordena as classes; nunca ordene à mão.
- **Sem config JS**: customizações novas vão para `@theme`, `@utility` e `@custom-variant`
  em CSS, não para um arquivo de configuração.

## Tarefa → arquivo

| Tarefa                                                                 | Arquivo                                |
| ---------------------------------------------------------------------- | -------------------------------------- |
| Instalar/entender `@import`, PostCSS e ausência de `tailwind.config`    | `references/configuration.md`           |
| Definir cores, fontes, espaçamento e breakpoints com `@theme`           | `references/configuration.md`           |
| Sobrescrever ou limpar o tema padrão (`--color-*: initial`)             | `references/configuration.md`           |
| Controlar o scan de arquivos (`@source`, `@source not`, `@source inline`) | `references/configuration.md`         |
| Criar variantes próprias (`@custom-variant`, dark por classe)           | `references/configuration.md`           |
| Carregar plugins (`@plugin`) e migrar de `tailwind.config.js`           | `references/configuration.md`           |
| Escrever utilitários comuns, responsivo, dark, `group`/`peer`           | `references/utilities-and-variants.md`  |
| Usar `data-*`/`aria-*` e valores arbitrários                            | `references/utilities-and-variants.md`  |
| Container queries (`@container`, `@md:`) e layout grid/flex/gap          | `references/utilities-and-variants.md`  |
| Criar utilitário customizado com `@utility`                             | `references/utilities-and-variants.md`  |
| Ordem de classes, componentes, `@apply`, acessibilidade e scan          | `references/best-practices.md`          |

## Diferenças v4 x v3 que importam aqui

| v3                                        | v4 neste repo                                                        |
| ----------------------------------------- | -------------------------------------------------------------------- |
| `tailwind.config.js` + `content`          | `@theme` em CSS; detecção automática de fontes                        |
| `@tailwind base/components/utilities`     | `@import "tailwindcss";`                                              |
| plugin dentro de `tailwindcss`            | pacote separado `@tailwindcss/postcss`                                |
| `postcss-import` + `autoprefixer`         | removidos; o plugin v4 já resolve import e prefixos                   |
| `@layer components` para classes próprias | `@utility` (ordenação automática por quantidade de propriedades)      |
| safelist no config                        | `@source inline("...")` (v4.1)                                        |
| `border` = `gray-200`, `ring` = 3px azul  | `border` = `currentColor`, `ring` = 1px (`currentColor`)              |
| dark mode via `darkMode: 'class'`         | `@custom-variant dark (&:where(.dark, .dark *));`                     |
| `outline-none` sumia no forced-colors     | `outline-hidden` mantém acessibilidade; `outline-none` remove de fato  |

## Regra de uso

Carregue o reference da tarefa antes de escrever classes ou CSS. Se o exemplo necessário não
estiver nos references, consulte o Context7 em vez de inventar API:

```bash
ctx7 docs /tailwindlabs/tailwindcss.com "pergunta específica"
# alternativa: /websites/tailwindcss
```
