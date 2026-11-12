# Boas práticas de estado, efeitos e renderização

> Fonte: `/reactjs/react.dev` via `ctx7 docs` (consultado em 2026-11-10).

## Estado mínimo

Guarde apenas o que não pode ser calculado. Se um valor pode ser derivado de props ou de
outro estado, calcule durante o render.

```tsx
function Form() {
  const [firstName, setFirstName] = useState('Taylor');
  const [lastName, setLastName] = useState('Swift');

  const fullName = firstName + ' ' + lastName; // ✅ calculado no render
}
```

Não faça:

```tsx
const [fullName, setFullName] = useState('');
useEffect(() => {
  setFullName(firstName + ' ' + lastName);
}, [firstName, lastName]);
```

O mesmo vale para listas filtradas:

```tsx
const [visibleTodos, setVisibleTodos] = useState<Todo[]>([]);
useEffect(() => {
  setVisibleTodos(getFilteredTodos(todos, filter));
}, [todos, filter]); // 🔴 estado redundante + Effect
```

Se o filtro for caro, `useMemo` é o lugar:

```tsx
const visibleTodos = useMemo(
  () => getFilteredTodos(todos, showActive),
  [todos, showActive],
);
```

## Estado derivado e seleção

Guarde o identificador, não o objeto inteiro. O objeto pode ficar obsoleto; o id não.

```tsx
function List({ items }: { items: Item[] }) {
  const [selectedId, setSelectedId] = useState<string | null>(null);
  const selection = items.find((item) => item.id === selectedId) ?? null;
  // ...
}
```

Isso dispensa Effects para "sincronizar" a seleção quando `items` muda e mantém a escolha
durante atualizações do array.

## Levantar estado

- Mantenha o estado o mais local possível.
- Levante apenas quando dois componentes realmente precisam compartilhar.
- Não coloque estado transitório (texto de formulário, item em hover) no topo da árvore nem
  em biblioteca global.
- Estado controlado por um pai vira prop + callback no filho.

## Resetar estado com `key`

Trocar a `key` de um componente destrói e recria sua subárvore, zerando o estado interno.
Use isso em vez de Effects que "resetam" campos manualmente.

```tsx
export default function App() {
  const [version, setVersion] = useState(0);

  return (
    <>
      <button onClick={() => setVersion((v) => v + 1)}>Reset</button>
      <Form key={version} />
    </>
  );
}
```

O mesmo princípio separa entidades em listas e alternâncias (`key="Taylor"` x `key="Sarah"`).

## Effects: só para sistemas externos

Effects sincronizam com algo fora do React: conexão, listener do browser, API imperativa,
subscription. Não use Effect para:

- transformar dados para renderização;
- reagir a um evento do usuário;
- ajustar estado a partir de outro estado;
- derivar valores que podem ser calculados no render.

```tsx
// ✅ sincronização real com sistema externo
useEffect(() => {
  function handleScroll() {
    console.log(window.scrollX, window.scrollY);
  }
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);
```

Cadeias de Effects que atualizam estado são a maior fonte de problemas de performance:
cada atualização dispara outro render. Prefira calcular no render ou tratar no event handler.

## Dependências de Effect

Liste todo valor reativo usado no Effect: props, estado, funções e objetos declarados no
componente. O array precisa ter tamanho constante e ser escrito inline.

```tsx
function ChatRoom({ roomId }: { roomId: string }) {
  const [serverUrl, setServerUrl] = useState('https://localhost:1234');

  useEffect(() => {
    const connection = createConnection(serverUrl, roomId);
    connection.connect();
    return () => connection.disconnect();
  }, [serverUrl, roomId]);
}
```

Se uma função/objeto criado no componente obriga o Effect a rodar demais:

1. mova a função para fora do componente, se ela não usa valores reativos;
2. mova a lógica para dentro do Effect;
3. use `useEffectEvent` para lógica que deve ver o valor mais recente sem ser reativa.

Não use `useEffectEvent` para simplesmente calar o lint.

## Limpeza

Todo setup precisa de cleanup simétrico: `addEventListener` → `removeEventListener`,
`setInterval` → `clearInterval`, subscription → unsubscribe, requisição → abort/ignore. Em
StrictMode o ciclo roda duas vezes em desenvolvimento; se a limpeza estiver errada, o bug
aparece ali.

```tsx
useEffect(() => {
  const connection = createConnection();
  connection.connect();
  return () => connection.disconnect();
}, []);
```

## Memoização: quando e quando não

Memoize apenas se:

- o cálculo é lento e muda pouco;
- o valor vai para um componente `memo`;
- o valor é dependência de outro Hook.

Antes de adicionar `memo`, `useMemo` ou `useCallback`, a documentação recomenda:

1. aceitar JSX como `children` em wrappers, para que atualizações do wrapper não re-renderizem
   os filhos;
2. manter estado local e não levantar mais que o necessário;
3. manter o render puro — se re-renderizar causa efeito visual, o bug é do componente;
4. remover Effects desnecessários que atualizam estado;
5. remover dependências desnecessárias dos Effects.

Memoização não compensa em aplicações com interações "grossas" (troca de página/seção). Em
editores e interações granulares (arrastar formas), tende a valer. Meça com o profiler antes
de espalhar `useCallback`.

## Context e re-renders

Quando o `value` do provider muda, React re-renderiza todos os consumidores; `memo` não evita
isso. Divida contextos por responsabilidade (estado e dispatch separados), mantenha o provider
perto de quem consome e evite objetos/funções novos no `value` sem necessidade.

## Render puro

O render deve ser puro: não mutar props/estado, não escrever em refs durante o render nem
disparar Effects. Em StrictMode, updaters rodam duas vezes para expor impureza.

## Server x Client no Next.js

Server Components renderizam no servidor, podem ser `async` e acessar banco/arquivos. Não usam
`useState`, Effects ou event handlers. Para interatividade, componha com um Client Component.

```tsx
// Server Component: pode ser async e acessar dados
import Expandable from './Expandable';

async function Notes() {
  const notes = await db.notes.getAll();
  return notes.map((note) => (
    <Expandable key={note.id}>
      <p>{note.text}</p>
    </Expandable>
  ));
}

// Client Component: interatividade e estado
'use client';

export default function Expandable({ children }: { children: React.ReactNode }) {
  const [expanded, setExpanded] = useState(false);
  return (
    <div>
      <button onClick={() => setExpanded(!expanded)}>Toggle</button>
      {expanded && children}
    </div>
  );
}
```

Regras práticas:

- `'use client'` no topo do arquivo, antes dos imports, marca o módulo e suas dependências
  como código de cliente.
- Um componente sem estado/eventos não precisa de `'use client'`; pode ser renderizado dos
  dois lados.
- Não importe módulos de `src/server/**` em Client Components.

## Anti-patterns

| Não faça | Faça |
| --- | --- |
| Estado para valor derivável | Calcular no render (ou `useMemo` se caro) |
| `useEffect` para sincronizar estado com estado | Calcular durante o render |
| `useEffect` para reagir a clique | Lógica no event handler |
| `key={index}` em lista reordenável | `key={item.id}` |
| `key={Math.random()}` | Id estável dos dados |
| `onClick={handleClick()}` | `onClick={handleClick}` |
| Mutar array/objeto do estado | `map`, `filter`, spread |
| Ler estado logo após `set` | Usar updater ou ler no próximo render |
| `useCallback` em tudo | Só com `memo`, dependência de Hook ou cálculo caro |
| Provider de contexto no mesmo componente que lê | Provider acima |
| Effect sem cleanup | Cleanup simétrico |
| `defaultProps` em função | Valor padrão na desestruturação |

## Checklist antes de commitar

- [ ] `npm run check` passa (lint + `tsc --noEmit`).
- [ ] `npm run format:write` executado.
- [ ] Nenhum estado redundante ou derivável.
- [ ] Nenhum Effect sem sistema externo ou sem cleanup.
- [ ] Listas com `key` estável dos dados.
- [ ] `'use client'` apenas onde há interatividade.
- [ ] Memoização justificada por medição ou por `memo`/dependência de Hook.
